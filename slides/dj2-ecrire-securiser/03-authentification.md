## 3. Authentification et secrets

<p class="lead">Une API d'administration sans authentification est une télécommande posée sur le trottoir.</p>

---

## Deux questions distinctes

<div class="left">

| | Question | Si la réponse est non |
|---|---|---|
| **Authentification** | **Qui** es-tu ? | **401** Unauthorized |
| **Autorisation** | As-tu **le droit** de faire ça ? | **403** Forbidden |

</div>

<div class="left">

L'ordre compte : on **authentifie d'abord**, on **autorise ensuite**.

- Pas de token, token expiré, token inconnu → on ne sait pas qui parle → **401**.
- Token valide, mais rôle « lecteur » sur une suppression → on sait qui parle, il n'a pas le droit → **403**.

</div>

<p class="small">Dans votre projet : deux tokens, deux rôles. Le lecteur lit, l'admin lit et écrit.</p>

---

## Où voyage le secret

```http
GET /containers HTTP/1.1
Authorization: Bearer 3fK9xQ…
```

<div class="left">

| Emplacement | Verdict |
|---|---|
| Header **`Authorization`** | ✓ la norme |
| Autre header (`X-API-Key`, `X-Auth-Token`) | ✓ courant |
| **Dans l'URL** (`?token=3fK9…`) | ✗ **jamais** : l'URL finit dans les logs, l'historique, les proxys |
| Dans le corps | ✗ les requêtes GET n'en ont pas |

</div>

<div class="memo">
Sans <strong>HTTPS</strong>, tout cela circule en clair : quiconque écoute le réseau récupère le token. En TP vous êtes sur <code>127.0.0.1</code>. En production, une API d'administration n'est <strong>jamais</strong> exposée en HTTP simple.
</div>

---

## Les quatre familles

| Mécanisme | Ce qui est envoyé | Durée de vie | Usage type |
|---|---|---|---|
| **Basic** | identifiant et mot de passe, encodés en base64 | celle du mot de passe | équipements réseau, outils anciens |
| **Clé d'API** | une longue chaîne aléatoire | longue, jusqu'à révocation | scripts, intégrations entre services |
| **Bearer token** | un token opaque ou un JWT | souvent courte | API modernes |
| **OAuth2** | un token obtenu auprès d'un serveur d'autorisation | courte, renouvelable | clouds, accès délégués |

<div class="left">

- **base64 n'est pas du chiffrement** : `echo YWRtaW46YWRtaW4= | base64 -d` rend `admin:admin`.
- Une **clé d'API** identifie un programme, pas une personne. Une clé par usage, pour pouvoir en révoquer une seule.

</div>

---

## Le JWT : un token qui se lit

Un **JSON Web Token** est fait de trois parties encodées en base64, séparées par des points.

```text
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJjaS1kZXBsb3kiLCJyb2xlIjoiYWRtaW4iLCJleHAiOjE3OTE4OTAwMDB9.4mPq…
└────── header ─────┘└──────────────────────── contenu ───────────────────────────────┘└ signature ┘
```

```json
{ "sub": "ci-deploy", "role": "admin", "exp": 1791890000 }
```

<div class="left">

- Le contenu est **lisible par tous** : un JWT est **signé**, pas chiffré. Aucun secret dedans.
- La **signature** prouve qu'il a été émis par le bon serveur et n'a pas été modifié.
- Le serveur le vérifie **sans interroger de base** : il lui suffit de la clé de signature.
- `exp` : la date d'expiration. Un JWT ne se révoque pas facilement, donc il vit **peu de temps**.

</div>

<p class="small">Décodez-en un sur <a href="https://jwt.io">jwt.io</a> : vous verrez son contenu sans connaître aucune clé.</p>

---

## OAuth2 : déléguer sans donner son mot de passe

Pour un programme qui agit en son propre nom (le cas de l'administration), le flux
s'appelle **client credentials**.

```bash
# 1. Le programme s'identifie auprès du serveur d'autorisation
curl -s -X POST https://auth.example.net/oauth/token \
     -d grant_type=client_credentials \
     -d client_id=inventaire-bot -d client_secret="$SECRET" -d scope=containers:read
```

```json
{ "access_token": "eyJhbGciOi…", "token_type": "Bearer", "expires_in": 3600 }
```

```bash
# 2. Il présente le token obtenu à l'API, jusqu'à son expiration
curl -s https://api.example.net/containers -H "Authorization: Bearer eyJhbGciOi…"
```

<div class="left">

- Le **secret durable** ne va qu'au serveur d'autorisation. L'API ne voit qu'un token **temporaire**.
- Le **scope** limite ce que le token permet : ici, lire les conteneurs, rien d'autre.

</div>

---

## Dans votre projet : deux Bearer tokens

```python
# app/securite.py
import secrets
from fastapi import Depends, HTTPException
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer

bearer = HTTPBearer(auto_error=False)     # on choisit nous-mêmes le code d'erreur


def role_de(token: HTTPAuthorizationCredentials | None) -> str:
    if token is not None:
        recu = token.credentials.encode()
        if secrets.compare_digest(recu, API_TOKEN_ADMIN.encode()):
            return "admin"
        if secrets.compare_digest(recu, API_TOKEN_READER.encode()):
            return "lecteur"
    raise HTTPException(status_code=401, detail="Token absent ou inconnu",
                        headers={"WWW-Authenticate": "Bearer"})
```

<p class="small"><code>secrets.compare_digest</code> compare en temps constant : avec <code>==</code>, le temps de réponse trahit le nombre de premiers caractères corrects, et un attaquant patient reconstitue le token.</p>

--

## Brancher l'authentification sur une route

```python
def exiger_lecture(token=Depends(bearer)) -> str:
    return role_de(token)                       # admin ou lecteur, sinon 401


def exiger_admin(token=Depends(bearer)) -> str:
    role = role_de(token)                       # 401 si on ne sait pas qui parle
    if role != "admin":
        raise HTTPException(status_code=403, detail="Action réservée au rôle admin")
    return role
```

```python
@app.get("/containers", summary="Liste les conteneurs gérés")
def lister(role: str = Depends(exiger_lecture)):
    ...

@app.delete("/containers/{identifiant}", status_code=204, summary="Supprime un conteneur")
def supprimer(identifiant: str, role: str = Depends(exiger_admin)):
    ...
```

<p class="small"><code>Depends</code> exécute la fonction <strong>avant</strong> la route. Si elle lève une exception, la route n'est jamais appelée. Dans <code>/docs</code>, un bouton <em>Authorize</em> apparaît.</p>

---

## Les secrets : cinq règles

<div class="left">

1. **Jamais dans le code.** Ni en dur, ni « provisoirement », ni dans un commentaire.
2. **Jamais dans git.** `.env` est dans `.gitignore` **avant** le premier commit. Un secret
   commité puis supprimé reste dans l'historique : il faut le considérer comme **compromis** et le changer.
3. **Générés, pas inventés.** `python -c "import secrets; print(secrets.token_urlsafe(32))"`
4. **Un secret par usage.** Le token admin, le token lecteur et le secret du webhook sont trois valeurs différentes.
5. **Le moindre privilège.** Un script qui ne fait que lire reçoit le token lecteur.

</div>

<div class="memo">

```python
API_TOKEN_ADMIN = os.environ["API_TOKEN_ADMIN"]     # pas de valeur par défaut :
```

Sans la variable, l'API **refuse de démarrer**. C'est voulu : mieux vaut une panne franche
qu'une API qui tourne avec un token vide ou un token « changeme ».

</div>

---

## En production, on va plus loin

<div class="left">

| Besoin | Réponse |
|---|---|
| Stocker les secrets | un coffre : HashiCorp Vault, AWS Secrets Manager, les secrets de l'orchestrateur |
| Les changer régulièrement | rotation automatique, tokens à durée de vie courte |
| Savoir qui a fait quoi | un token **par client**, et des logs d'appels |
| Limiter les dégâts d'une fuite | scopes étroits, liste d'adresses IP autorisées |
| Détecter une fuite dans git | `gitleaks`, la détection de secrets de GitHub |

</div>

<p class="small">Votre projet s'arrête aux variables d'environnement : c'est le socle sur lequel tous ces mécanismes se branchent.</p>

---

## Ce qu'il faut retenir

<div class="left">

1. **Authentifier** (qui ?) puis **autoriser** (le droit ?). **401** puis **403**.
2. Le secret voyage dans un **header**, sur **HTTPS**, jamais dans l'URL.
3. **Basic** : mot de passe encodé, pas chiffré. **Clé d'API** : longue durée, une par usage.
4. **JWT** : signé, lisible par tous, vérifiable sans base, durée de vie courte.
5. **OAuth2 client credentials** : un secret durable échangé contre un token temporaire et limité.
6. Les secrets vivent dans l'**environnement**, jamais dans le code ni dans git.

</div>
