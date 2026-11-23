## 2. Les API des clouds et des hyperviseurs

<p class="lead">Dans un cloud, tout est un appel d'API. La console web, la ligne de commande et Terraform ne sont que trois clients du même <em>control plane</em>.</p>

---

## Control plane et data plane

<div class="left">

| | Rôle | Exemple |
|---|---|---|
| **Control plane** | **gérer** les ressources : créer, configurer, supprimer | « crée une machine virtuelle de 4 Go » |
| **Data plane** | **faire le travail** pour lequel la ressource existe | la machine virtuelle sert ses pages web |

</div>

<div class="left">

- Vous administrez **par le control plane**, et c'est une API REST.
- Si le control plane tombe, ce qui tourne continue de tourner : on ne peut juste plus rien changer.
- Dans votre projet : l'API Docker Engine est le control plane, les conteneurs nginx sont le data plane.

</div>

---

## Trois plateformes, le même geste

Démarrer une machine virtuelle :

```bash
# Proxmox VE : token d'API dans un header
curl -X POST "https://pve.example.net:8006/api2/json/nodes/pve1/qemu/104/status/start" \
     -H "Authorization: PVEAPIToken=automate@pve!ci=3b1f…"

# OpenStack : token temporaire obtenu auprès du service d'identité (Keystone)
curl -X POST "https://cloud.example.net:8774/v2.1/servers/7c2e…/action" \
     -H "X-Auth-Token: gAAAAAB…" -H "Content-Type: application/json" \
     -d '{"os-start": null}'

# AWS : requête signée, en pratique toujours faite par le SDK ou la CLI
aws ec2 start-instances --instance-ids i-0abc123def456
```

<p class="small">Trois vocabulaires, trois modes d'authentification, mais toujours : une ressource, une méthode HTTP, un code de statut, du JSON.</p>

---

## SDK ou appel direct ?

<div class="left">

Un **SDK** est une bibliothèque qui enveloppe l'API REST d'une plateforme :
`boto3` pour AWS, `openstacksdk`, `proxmoxer`, `pynetbox`, le SDK `docker` pour Python.

| Le SDK s'occupe de | Ce que ça vous évite |
|---|---|
| l'authentification (signature des requêtes, renouvellement des tokens) | la partie la plus pénible à écrire soi-même |
| la pagination | les boucles sur `next` |
| les nouvelles tentatives | la gestion des 429 et des 5xx |
| les types | les fautes de frappe dans les noms de champs |

</div>

<div class="memo">
En production, utilisez le SDK quand il existe. Dans ce cours, vous appelez les API <strong>directement avec <code>requests</code></strong> : c'est le seul moyen de comprendre ce que le SDK fait pour vous, et de vous débrouiller le jour où il n'y en a pas.
</div>

---

## Les opérations longues

Créer une machine virtuelle prend une minute. L'API ne va pas garder la connexion ouverte.

```http
POST /servers HTTP/1.1                       HTTP/1.1 202 Accepted
                                             Location: /tasks/8841
```

```python
tache = session.post(f"{API}/servers", json=demande, timeout=10)
tache.raise_for_status()
url_tache = tache.headers["Location"]

while True:                                        # on interroge la tâche jusqu'à sa fin
    etat = session.get(url_tache, timeout=10).json()
    if etat["status"] in ("done", "failed"):
        break
    time.sleep(2)
```

<div class="left">

- **202 Accepted** : « j'ai bien reçu, je m'en occupe, ce n'est pas encore fait ».
- Le client **interroge** une ressource « tâche » jusqu'à son terme (*polling*).
- Toujours une **limite de temps** sur la boucle : une tâche bloquée ne doit pas bloquer le script.

</div>

---

## Les quotas : 429 et la pause qui s'allonge

Un cloud protège son control plane. Trop de requêtes, et il répond :

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 4
```

```python
for tentative in range(5):
    reponse = session.get(url, timeout=10)
    if reponse.status_code != 429:
        break
    pause = int(reponse.headers.get("Retry-After", 2 ** tentative))   # 1, 2, 4, 8, 16 s
    time.sleep(pause)
reponse.raise_for_status()
```

<div class="left">

- **Respectez `Retry-After`** quand il est présent.
- Sinon, attendez de plus en plus longtemps : c'est le *backoff* exponentiel.
- Réessayer **immédiatement et en boucle** aggrave la surcharge, et finit par vous faire bannir.

</div>

---

## Ce qu'il faut retenir

<div class="left">

1. Le **control plane** gère les ressources, le **data plane** fait le travail. On administre par le premier.
2. Proxmox, OpenStack, AWS : même patron REST, seuls le vocabulaire et l'authentification changent.
3. Un **SDK** enveloppe l'API : authentification, pagination, nouvelles tentatives.
4. Opération longue : **202 Accepted**, puis on interroge une tâche jusqu'à sa fin.
5. **429** : on ralentit, en respectant `Retry-After` ou avec une pause croissante.

</div>
