## 1. Infrastructure as Code et NetDevOps

<p class="lead">Décrire l'infrastructure dans des fichiers, et laisser des outils la mettre en conformité par API.</p>

---

## Infrastructure as Code

<div class="left">

L'état voulu de l'infrastructure est **écrit dans des fichiers texte**, versionnés dans git,
relus comme du code, et appliqués par un outil. Plus de clics, plus de commandes tapées à la main.

| Ce qu'on y gagne | Parce que |
|---|---|
| **Reproductible** | le même fichier donne le même résultat, en recette comme en production |
| **Traçable** | `git log` dit qui a changé quoi, quand, et pourquoi |
| **Relu** | une modification passe par une revue avant d'être appliquée |
| **Réversible** | revenir en arrière, c'est appliquer la version précédente |

</div>

<p class="small">Terraform, Ansible, Pulumi, les manifestes Kubernetes : tous ces outils sont, au fond, des <strong>clients d'API</strong>. Ils lisent vos fichiers et font les appels à votre place.</p>

---

## Impératif ou déclaratif

<div class="cols">
<div>
<h4>Impératif : la liste des gestes</h4>
<pre><code class="language-bash">curl -X POST $API/containers \
     -d '{"name": "web1", …}'
curl -X POST $API/containers \
     -d '{"name": "web2", …}'</code></pre>
<p class="small">« Crée web1, crée web2. » Relancé une seconde fois, le script échoue : les noms sont pris.</p>
</div>
<div>
<h4>Déclaratif : l'état voulu</h4>
<pre><code class="language-yaml">conteneurs:
  - name: web1
    image: nginx:alpine
  - name: web2
    image: nginx:alpine</code></pre>
<p class="small">« Il doit exister web1 et web2. » L'outil compare avec l'existant et ne fait que l'écart.</p>
</div>
</div>

<div class="valid">
Le déclaratif est <strong>idempotent par construction</strong> : appliquer dix fois le même fichier donne le même état. C'est ce qui permet de le rejouer sans crainte, à la main comme dans une tâche planifiée.
</div>

---

## La source de vérité

<div class="left">

Pour comparer « l'état voulu » et « l'état réel », il faut **un endroit qui fait autorité**
sur ce qui doit exister : quelles machines, quelles adresses, quels VLAN, quels rôles.

</div>

| | Sans source de vérité | Avec NetBox |
|---|---|---|
| Quelle IP est libre ? | un tableur, peut-être à jour | `GET /api/ipam/prefixes/12/available-ips/` |
| Qui possède cette machine ? | on demande dans le chat | un champ de l'inventaire |
| Qu'est-ce qui devrait tourner ? | ce qui tourne (on espère) | ce que dit l'inventaire |

<p class="lead">NetBox ne configure rien : il <strong>décrit</strong>. Les outils d'automatisation le lisent, et le tiennent à jour.</p>

<p class="small">C'est le rôle qu'il joue dans votre projet : chaque conteneur créé par votre API y est déclaré.</p>

---

## Le réseau, dernier bastion du CLI

<div class="left">

Pendant des décennies, on a configuré routeurs et commutateurs en SSH, ligne par ligne :

```text
switch# configure terminal
switch(config)# interface GigabitEthernet1/0/12
switch(config-if)# description Borne wifi etage 2
switch(config-if)# switchport access vlan 30
```

Pour automatiser, on a d'abord écrit des scripts qui **tapent ces commandes à la place d'un humain**
et **lisent l'écran** pour savoir si ça a marché. Chaque constructeur, chaque version a sa syntaxe
et ses messages d'erreur.

</div>

<p class="lead"><strong>NetDevOps</strong> : appliquer au réseau les pratiques du DevOps. Configuration versionnée, tests, déploiement automatisé, et des <strong>API</strong> au lieu du terminal.</p>

---

## Piloter par modèle : YANG

**YANG** est un langage qui décrit **la structure des données** d'un équipement : ce qui est
configurable, avec quels types, quelles contraintes.

```yang
container interfaces {
  list interface {
    key "name";
    leaf name        { type string; }
    leaf description { type string; }
    leaf enabled     { type boolean; default "true"; }
  }
}
```

<div class="left">

- C'est un **schéma**, comme OpenAPI en est un pour une API REST.
- Des modèles **normalisés** (IETF, OpenConfig) décrivent une interface ou une route de la
  même façon chez tous les constructeurs.
- L'équipement **valide** la configuration contre le modèle avant de l'appliquer.

</div>

---

## NETCONF et RESTCONF

Deux protocoles pour lire et écrire les données décrites par les modèles YANG.

| | NETCONF | RESTCONF |
|---|---|---|
| Transport | SSH (port 830) | **HTTPS** |
| Format | XML | **JSON** ou XML |
| Opérations | `get-config`, `edit-config`, `commit`… | **GET, POST, PUT, PATCH, DELETE** |
| Transactions | oui : configuration candidate, validation, `commit`, retour arrière | non : chaque requête s'applique tout de suite |
| Usage type | changements lourds et atomiques sur plusieurs équipements | intégration simple avec des outils qui parlent déjà REST |

<p class="lead">RESTCONF, c'est une API REST dont les ressources sont les nœuds du modèle YANG.</p>

---

## RESTCONF en pratique

```bash
# Lire la configuration d'une interface
curl -s -u "$UTILISATEUR:$MOT_DE_PASSE" \
     -H "Accept: application/yang-data+json" \
     https://routeur.example.net/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet2
```

```json
{ "ietf-interfaces:interface": {
    "name": "GigabitEthernet2",
    "description": "Lien vers le coeur",
    "type": "iana-if-type:ethernetCsmacd",
    "enabled": true } }
```

```bash
# Modifier sa description : PATCH, seul le champ envoyé change
curl -s -u "$UTILISATEUR:$MOT_DE_PASSE" -X PATCH \
     -H "Content-Type: application/yang-data+json" \
     -d '{"ietf-interfaces:interface": {"description": "Lien vers le coeur, fibre B"}}' \
     https://routeur.example.net/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet2
```

<p class="small">Le chemin de l'URL <strong>est</strong> le chemin dans le modèle YANG : <code>module:conteneur/liste=clé</code>.</p>

--

## Ce que vous reconnaissez déjà

<div class="left">

Rien de nouveau par rapport à la première demi-journée :

| Notion | Dans RESTCONF |
|---|---|
| Ressource et URI | un nœud du modèle YANG, adressé par son chemin |
| Verbes | GET lit, PATCH modifie en partie, PUT remplace, DELETE supprime |
| Codes de statut | 200, 201, 204, 400, 401, 404, 409 |
| Schéma | YANG, là où votre API a OpenAPI |
| Idempotence | PUT et DELETE rejouables sans risque |

</div>

<div class="valid">
C'est tout l'intérêt de REST : une fois les principes compris, <strong>chaque nouvelle API se lit en dix minutes</strong>. Seuls le vocabulaire des ressources et le mode d'authentification changent.
</div>

<p class="small">Pas de TP sur RESTCONF dans ce cours : il faudrait un équipement réseau par étudiant. Le TP6 applique les mêmes gestes à NetBox.</p>

---

## La chaîne complète

<div class="left">

1. L'état voulu est décrit dans la **source de vérité** (NetBox) et dans **git**.
2. Un outil lit cet état **par API**.
3. Il le compare à l'état réel, lu **par API** sur les équipements (RESTCONF, API cloud, API Docker).
4. Il applique l'écart **par API**.
5. Il met à jour la source de vérité et rend compte.

</div>

<p class="lead">À chaque étape, une API REST. Votre projet réalise cette chaîne en miniature : Docker est l'équipement, NetBox l'inventaire, et votre API l'outil au milieu.</p>

---

## Ce qu'il faut retenir

<div class="left">

1. **Infrastructure as Code** : l'état voulu est dans des fichiers versionnés, appliqués par un outil.
2. **Déclaratif** : on décrit l'état, pas les gestes. C'est idempotent, donc rejouable.
3. La **source de vérité** (NetBox) décrit ce qui doit exister. Elle ne configure rien.
4. **YANG** est un langage de modélisation : il décrit la structure des données d'un équipement.
5. **NETCONF** : SSH, XML, transactions. **RESTCONF** : HTTPS, JSON, verbes HTTP.
6. **NetDevOps** : les pratiques DevOps appliquées au réseau, API à la place du terminal.

</div>
