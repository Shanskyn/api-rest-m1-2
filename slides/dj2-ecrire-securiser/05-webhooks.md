## 5. Les webhooks

<p class="lead">Jusqu'ici, c'est vous qui appeliez les API. Un webhook, c'est l'inverse : <strong>le système vous appelle</strong> quand il se passe quelque chose.</p>

---

## Interroger ou être prévenu

<div class="cols">
<div>
<h4>Interrogation (<em>polling</em>)</h4>
<pre><code class="language-python">while True:
    etat = session.get(url).json()
    if etat["state"] == "exited":
        reparer()
    time.sleep(60)</code></pre>
<p class="small">« Tu es tombé ? Et maintenant ? Et maintenant ? » Mille requêtes pour rien, et jusqu'à 60 secondes de retard.</p>
</div>
<div>
<h4>Webhook</h4>
<pre><code class="language-http">POST /webhooks/alerts HTTP/1.1
X-Webhook-Secret: 9dK2…
Content-Type: application/json
&#32;
{"alert": "container_down",
 "container": "web1"}</code></pre>
<p class="small">« Je viens de tomber. » Une seule requête, au moment où l'événement se produit.</p>
</div>
</div>

<p class="lead">Un webhook est une requête HTTP <strong>entrante</strong> : votre API devient le serveur de l'outil de supervision.</p>

---

## Qui envoie des webhooks ?

<div class="left">

| Émetteur | Événement type | Réaction automatisable |
|---|---|---|
| Supervision (Alertmanager, Zabbix, Grafana) | un service ne répond plus | le redémarrer, ouvrir un ticket |
| Forge git (GitHub, GitLab) | un `push` sur `main` | lancer un déploiement |
| NetBox | une adresse IP est attribuée | créer l'enregistrement DNS |
| Cloud | une machine virtuelle est arrêtée | la retirer du load balancer |
| Outil de tickets | une demande est validée | créer le compte demandé |

</div>

<p class="small">L'<strong>auto-remédiation</strong> : une alerte déclenche automatiquement l'action corrective qu'un humain aurait faite. C'est votre TP7.</p>

---

## Problème 1 : n'importe qui peut appeler

Une route de webhook est une URL publique qui **déclenche une action**. Il faut prouver que
l'appel vient bien de l'émetteur attendu.

<div class="left">

| Méthode | Principe |
|---|---|
| **Secret partagé** dans un header | l'émetteur envoie une valeur connue des deux côtés. Simple. **Choix du projet.** |
| **Signature HMAC** | l'émetteur signe le corps avec le secret. Le secret ne circule jamais, et le corps ne peut pas être modifié en route. Choix de GitHub, Stripe, GitLab. |

</div>

```python
@app.post("/webhooks/alerts", status_code=202)
def recevoir_alerte(alerte: Alerte, x_webhook_secret: str | None = Header(default=None)):
    if x_webhook_secret is None or not secrets.compare_digest(
        x_webhook_secret.encode(), WEBHOOK_SECRET.encode()
    ):
        raise HTTPException(status_code=401, detail="Secret de webhook absent ou faux")
    ...
```

<p class="small">Le paramètre <code>x_webhook_secret</code> lit le header <code>X-Webhook-Secret</code> : FastAPI convertit les tirets.</p>

--

## La signature HMAC, pour comprendre

```python
import hashlib, hmac

# Côté émetteur : signer le corps exact de la requête
signature = hmac.new(SECRET.encode(), corps, hashlib.sha256).hexdigest()
# envoyée dans un header : X-Signature: sha256=5d41402abc4b…

# Côté récepteur : recalculer et comparer en temps constant
attendue = hmac.new(SECRET.encode(), corps_recu, hashlib.sha256).hexdigest()
if not hmac.compare_digest(attendue, signature_recue):
    raise HTTPException(status_code=401)
```

<div class="left">

- Le secret **ne quitte jamais** les deux machines.
- Un octet du corps modifié, et la signature ne correspond plus.
- Pour bloquer le replay d'une vieille requête interceptée, on signe aussi un **timestamp**.

</div>

<p class="small">Non demandé dans le projet. À connaître : c'est ce que vous rencontrerez en configurant un webhook GitHub ou GitLab.</p>

---

## Problème 2 : la même alerte arrive deux fois

<div class="left">

Un émetteur qui ne reçoit pas de réponse à temps **renvoie** l'événement. Une supervision
répète une alerte tant qu'elle n'est pas résolue. Votre récepteur recevra des **doublons**.

</div>

```python
conteneur = docker_client.lire(alerte.container)

if conteneur["state"] == "running":
    # Alerte reçue deux fois, ou conteneur déjà reparti : il n'y a rien à faire.
    return {"status": "ignoree", "container": conteneur["name"]}

docker_client.redemarrer(conteneur["id"])
return {"status": "remediee", "container": conteneur["name"]}
```

<div class="valid">
Un récepteur de webhook doit être <strong>idempotent</strong> : traiter deux fois le même événement ne doit pas produire deux fois l'effet. Ici, on regarde l'état réel avant d'agir, au lieu de redémarrer aveuglément.
</div>

---

## Problème 3 : répondre vite

<div class="left">

L'émetteur attend votre réponse, en général quelques secondes au plus. Passé ce délai, il
considère l'envoi comme échoué, et recommence.

| Si le traitement est | Alors |
|---|---|
| rapide (redémarrer un conteneur) | traiter, puis répondre **202** |
| long (reconstruire une machine) | répondre **202 tout de suite**, traiter en arrière-plan |

</div>

<div class="memo">

**202 Accepted** dit exactement cela : « reçu et pris en charge », sans rien promettre sur le
résultat. C'est le code attendu par le contrat pour `/webhooks/alerts`.

Avec FastAPI, un traitement différé s'écrit avec `BackgroundTasks`. Dans une vraie
architecture, on dépose l'événement dans une **file d'attente** et un autre processus le traite.

</div>

---

## Ce que valide le corps

```python
class Alerte(BaseModel):
    alert: Literal["container_down"]                       # allowlist des types connus
    container: str = Field(pattern=r"^[a-z][a-z0-9-]{1,30}$")
```

| Cas | Réponse |
|---|---|
| header `X-Webhook-Secret` absent ou faux | **401**, rien n'est redémarré |
| type d'alerte inconnu, corps invalide | **422** |
| conteneur inconnu, ou sans le label de votre API | **404** |
| alerte valide, conteneur arrêté | **202**, redémarré, NetBox mis à jour |
| alerte valide, conteneur déjà démarré | **202**, rien à faire |

<p class="small ko">Un webhook ne doit jamais permettre de redémarrer un conteneur que l'API n'a pas le droit de toucher : la règle du label s'applique ici aussi.</p>

---

## Ce qu'il faut retenir

<div class="left">

1. **Webhook** : une requête HTTP entrante, envoyée par un système quand un événement se produit.
2. Il remplace l'**interrogation périodique** : moins de requêtes, réaction immédiate.
3. **Authentifier l'émetteur** : secret partagé, ou mieux, signature **HMAC** du corps.
4. **Idempotence** : les doublons sont normaux, le récepteur ne doit pas agir deux fois.
5. **Répondre vite** : 202, et traitement en arrière-plan si c'est long.
6. **Auto-remédiation** : l'alerte déclenche la réparation, sans humain dans la boucle.

</div>
