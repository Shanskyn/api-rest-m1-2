## Les TP de la demi-journée

<p class="lead">Quatre TP, quatre blocs du vérificateur à passer au vert.</p>

<div class="left">

| | | | Bloc du vérificateur |
|---|---|---|---|
| **TP4** | ~1h15 | Écriture : créer, arrêter, démarrer, supprimer | `ecriture` |
| **TP5** | ~45′ | Authentification : deux tokens, deux rôles | `authentification` |
| **TP6** | ~45′ | Inventaire : NetBox suit vos conteneurs | `netbox` |
| **TP7** | ~45′ | Webhook d'auto-remédiation | `webhook` |

</div>

<div class="memo">
Ce programme est dense. <strong>L'ordre compte plus que la vitesse</strong> : un bloc vraiment vert vaut mieux que quatre blocs à moitié faits. Ce qui n'est pas fini en séance se termine avant la dernière demi-journée.
</div>

---

## Avant de commencer

```bash
docker compose up -d                 # l'environnement
docker compose ps                    # netbox doit être "healthy"
python verifier.py lecture           # doit afficher 7/7
```

Tester votre API à la main, tout au long de la séance :

```bash
set -a; source .env; set +a          # charge les variables du .env dans ce terminal
A=http://127.0.0.1:8000

curl -i -X POST $A/containers \
     -H "Authorization: Bearer $API_TOKEN_ADMIN" -H "Content-Type: application/json" \
     -d '{"name": "web2", "image": "nginx:alpine"}'
```

<p class="small">Ou depuis <a href="http://127.0.0.1:8000/docs">/docs</a>, bouton <em>Try it out</em>. Le header <code>Authorization</code> ne sert qu'à partir du TP5.</p>

---

## TP4 : Écriture <span class="small">(~1h15)</span>

**Objectif** : votre API crée, arrête, démarre et supprime des conteneurs, et refuse ce qui est invalide.

<div class="left">

1. Déclarez le modèle pydantic du corps de `POST /containers` : `name` avec son **motif**,
   `image` en **allowlist** de trois valeurs.
2. Écrivez `POST /containers` : créer le conteneur **avec le label**, le **démarrer**,
   répondre **201** avec sa représentation. **409** si le nom est pris.
3. Écrivez `POST /containers/{id}/stop` et `/start` : **200** et le conteneur, **409** s'il
   est déjà dans l'état demandé, **404** s'il est inconnu **ou non géré**.
4. Écrivez `DELETE /containers/{id}` : **204** sans corps, **404** s'il est inconnu ou non géré.
5. Vérifiez avec `docker ps -a` que ce que votre API annonce est vrai.

</div>

<div class="valid">
✅ <strong>Validation</strong> : <code>python verifier.py lecture ecriture</code> au vert (7/7 et 13/13).
Un commit par route.
</div>

--

## TP4 : mémo API Docker

<div class="memo">

```python
corps = {"Image": image, "Labels": {"adminapi.managed": "true"}}
reponse = session.post(f"{DOCKER}/containers/create",
                       params={"name": nom}, json=corps, timeout=10)
# 201 : reponse.json()["Id"]     409 : nom déjà pris     404 : image absente de la machine

session.post(f"{DOCKER}/containers/{ident}/start", timeout=10)
session.post(f"{DOCKER}/containers/{ident}/stop", params={"t": 2}, timeout=30)
session.post(f"{DOCKER}/containers/{ident}/restart", params={"t": 2}, timeout=30)
# 204 : fait     304 : déjà dans cet état     404 : inconnu

session.delete(f"{DOCKER}/containers/{ident}", params={"force": "true"}, timeout=30)
# 204 : supprimé     404 : inconnu
```

`create` ne démarre pas le conteneur : il faut enchaîner avec `start`.

`t=2` : Docker laisse 2 secondes au processus pour s'arrêter proprement avant de le tuer.

Renvoyer 204 sans corps : `return Response(status_code=204)` (`from fastapi import Response`).

</div>

--

## TP4 : si vous bloquez

<div class="left">

- **« 404 à la création »** → l'image n'est pas sur la machine : `docker pull nginx:alpine`.
- **« Le conteneur est créé mais `state` vaut `created` »** → vous n'avez pas appelé `start`,
  ou vous renvoyez la représentation lue **avant** le démarrage. Relisez le conteneur après.
- **« Je réponds 200 au lieu de 409 sur un second `stop` »** → Docker répond **304** dans ce cas.
  `raise_for_status()` ne lève rien sur un 304 : testez-le explicitement.
- **« Mon `stop` arrête un conteneur sans label »** → avant toute action, relisez le
  conteneur et vérifiez son label. Une seule fonction pour ça, appelée partout.
- **« 422 alors que mon JSON est bon »** → il manque `-H "Content-Type: application/json"`.

</div>

<p class="small">Si ce n'est pas déjà fait, sortez les appels Docker dans <code>app/docker_client.py</code> : cinq routes qui dupliquent les mêmes URL deviennent vite ingérables.</p>

---

## TP5 : Authentification <span class="small">(~45′)</span>

**Objectif** : plus personne ne touche à vos conteneurs sans montrer patte blanche.

<div class="left">

1. Générez **trois secrets différents** et rangez-les dans `.env` : `API_TOKEN_ADMIN`,
   `API_TOKEN_READER`, `WEBHOOK_SECRET`. Mettez `.env.example` à jour, **sans les valeurs**.
2. Lisez-les dans l'environnement, **sans valeur par défaut**.
3. Écrivez `app/securite.py` : une dépendance qui exige un token valide, une autre qui exige le rôle admin.
4. Protégez les routes : **lecture** pour `GET /containers` et `GET /containers/{id}`,
   **admin** pour les quatre routes d'écriture. `/health` reste ouvert.
5. Vérifiez les trois cas à la main : sans token, token lecteur, token admin.

</div>

<div class="valid">
✅ <strong>Validation</strong> : <code>python verifier.py authentification</code> au vert (9/9), et les blocs
précédents toujours verts. <code>git log -p | grep -c "$API_TOKEN_ADMIN"</code> renvoie <strong>0</strong>.
</div>

--

## TP5 : mémo

<div class="memo">

**Générer un secret**

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

**Les trois cas à la main**

```bash
curl -s -o /dev/null -w "%{http_code}\n" $A/containers                                        # 401
curl -s -o /dev/null -w "%{http_code}\n" $A/containers -H "Authorization: Bearer $API_TOKEN_READER"   # 200
curl -s -o /dev/null -w "%{http_code}\n" -X DELETE $A/containers/web2 \
     -H "Authorization: Bearer $API_TOKEN_READER"                                              # 403
```

**Le piège de `HTTPBearer`** : avec son réglage par défaut, c'est lui qui répond quand
le header manque, et pas forcément avec le code que vous voulez. Déclarez-le avec
`auto_error=False` et levez vous-même le **401**.

**Dans le conteneur**, les variables arrivent par `env_file: .env` du compose. Après avoir
modifié `.env` : `docker compose up -d api` pour les recharger.

</div>

---

## TP6 : Inventaire NetBox <span class="small">(~45′)</span>

**Objectif** : à tout moment, NetBox dit la vérité sur les conteneurs que votre API gère.

<div class="left">

1. Créez `app/netbox_client.py` : une `Session` avec le token, l'adresse lue dans `NETBOX_URL`.
2. Écrivez une fonction qui retrouve l'**identifiant du cluster** `docker-local`.
3. À la **création** d'un conteneur : créez la machine virtuelle du même nom, statut `active`.
4. À l'**arrêt** : statut `offline`. Au **démarrage** : statut `active`.
5. À la **suppression** : supprimez la machine virtuelle.
6. Ouvrez NetBox (Virtualization → Virtual Machines) et regardez vos conteneurs y apparaître.

</div>

<div class="valid">
✅ <strong>Validation</strong> : <code>python verifier.py netbox</code> au vert (5/5), les blocs précédents toujours verts.
</div>

--

## TP6 : mémo API NetBox

<div class="memo">

```python
N = f"{NETBOX_URL}/api/virtualization"

# L'identifiant du cluster
cluster = netbox.get(f"{N}/clusters/", params={"name": "docker-local"}, timeout=10)
id_cluster = cluster.json()["results"][0]["id"]

# Créer la machine virtuelle : 201
netbox.post(f"{N}/virtual-machines/", timeout=10,
            json={"name": nom, "cluster": id_cluster, "status": "active"})

# La retrouver par son nom : results est une liste, vide si elle n'existe pas
trouvee = netbox.get(f"{N}/virtual-machines/", timeout=10,
                     params={"name": nom, "cluster_id": id_cluster}).json()["results"]

# Changer son statut : PATCH, seul le champ envoyé change
netbox.patch(f"{N}/virtual-machines/{id_vm}/", json={"status": "offline"}, timeout=10)

# La supprimer : 204
netbox.delete(f"{N}/virtual-machines/{id_vm}/", timeout=10)
```

Les URL NetBox **finissent par `/`**. Sans lui, NetBox répond **404**.

</div>

--

## TP6 : la question qui fâche

<div class="left">

Votre API fait maintenant **deux écritures** pour une seule demande : une dans Docker, une
dans NetBox. Que se passe-t-il si la seconde échoue ?

| Scénario | État |
|---|---|
| Docker crée le conteneur, NetBox est en panne | un conteneur **sans fiche d'inventaire** |
| NetBox supprime la fiche, Docker refuse la suppression | une fiche manquante pour un conteneur **qui tourne** |

Il n'y a pas de transaction entre deux API. Trois attitudes possibles :

- **compenser** : si NetBox échoue, défaire ce qu'on vient de faire dans Docker, et répondre 502 ;
- **signaler** : répondre 502 et laisser l'incohérence, en la journalisant ;
- **réconcilier** : une tâche périodique compare Docker et NetBox et corrige les écarts.

</div>

<p class="small">Non vérifié par le contrat. Choisissez une attitude et écrivez-la en commentaire : c'est la vraie difficulté de l'orchestration.</p>

---

## TP7 : Webhook d'auto-remédiation <span class="small">(~45′)</span>

**Objectif** : un conteneur tombe, une alerte arrive, votre API le redémarre toute seule.

<div class="left">

1. Déclarez le modèle `Alerte` : `alert` limité à `container_down`, `container` avec le motif des noms.
2. Écrivez `POST /webhooks/alerts` : **401** si le header `X-Webhook-Secret` est absent ou faux.
3. Retrouvez le conteneur visé : **404** s'il est inconnu ou non géré.
4. S'il est arrêté : **redémarrez-le**, mettez NetBox à jour, répondez **202**.
   S'il tourne déjà : répondez **202** sans rien faire.
5. Lancez `python surveillant.py` dans un second terminal, puis `docker stop` sur un conteneur
   géré. Regardez-le repartir.

</div>

<div class="valid">
✅ <strong>Validation</strong> : <code>python verifier.py webhook</code> au vert (7/7). Démonstration : un
<code>docker stop web2</code> est suivi, en moins de dix secondes, d'un conteneur de nouveau <code>running</code>
et d'un statut <code>active</code> dans NetBox.
</div>

--

## TP7 : mémo

<div class="memo">

**Envoyer une alerte à la main**

```bash
curl -i -X POST $A/webhooks/alerts \
     -H "X-Webhook-Secret: $WEBHOOK_SECRET" -H "Content-Type: application/json" \
     -d '{"alert": "container_down", "container": "web2"}'
```

**Lire un header** avec FastAPI :

```python
from fastapi import Header

def recevoir_alerte(alerte: Alerte, x_webhook_secret: str | None = Header(default=None)):
```

**Cette route n'utilise pas les tokens de l'API.** L'émetteur d'alertes ne connaît que le
secret du webhook : s'il fuit, il ne permet ni de lister ni de supprimer quoi que ce soit.

**Le surveillant** (`surveillant.py`, fourni) interroge Docker toutes les 5 secondes et envoie
une alerte par conteneur géré arrêté. Lisez ses 60 lignes : c'est un client d'API complet.

</div>

--

## TP7 : pour aller plus loin

<div class="left">

- Tant qu'un conteneur est arrêté, le surveillant renvoie l'alerte toutes les 5 secondes.
  Votre route le supporte-t-elle sans rien casser ? C'est l'**idempotence** du récepteur.
- Un conteneur qui plante **dès son démarrage** sera redémarré à l'infini. Comment limiter
  le nombre de remédiations par conteneur sur une période donnée ?
- Remplacez le secret partagé par une **signature HMAC** du corps (voir la slide dédiée).
- Ajoutez un second type d'alerte de votre invention, et son traitement.

</div>

<p class="small">Rien de tout cela n'est vérifié par le contrat. Gardez le bloc <code>webhook</code> au vert en expérimentant.</p>

---

## Avant de partir

```bash
python verifier.py lecture ecriture authentification netbox webhook
git status && git log --oneline | head
git push
```

<div class="valid">
D'ici la dernière demi-journée : amenez ces cinq blocs au vert. La dernière séance est courte :
un TP d'une heure, le quiz, puis le rendu. Il n'y aura pas le temps de rattraper l'écriture ou l'authentification.
</div>

---

## La prochaine fois

<div class="left">

**Observer, documenter, rendre**

- Les métriques : exposer l'état de votre API à un outil de supervision
- Soigner sa documentation, limiter le débit, faire évoluer une API sans casser ses clients
- **TP8** : `/metrics` et finitions
- **Quiz** : 20 questions sur les concepts des trois demi-journées, 30 minutes, sur feuille
- **Rendu** : le dernier commit poussé en fin de séance est celui qui est noté

</div>

<p class="small">Pour réviser le quiz : les slides « Ce qu'il faut retenir » de chaque chapitre.</p>
