# API Rest pour l'administration système : demi-journée 2

Support de la **deuxième demi-journée** : écrire, sécuriser, réagir.
Les slides de la première demi-journée sont incluses, pour référence.

## Avant de commencer

Ton API doit passer le bloc `lecture` du vérificateur. Depuis ton dépôt :

```bash
docker compose up -d
docker compose ps                # netbox doit être "healthy"
python verifier.py lecture       # attendu : 7/7
```

Si ce n'est pas le cas, c'est ta première tâche de la séance.

Le dossier `projet/` de cette archive contient les mêmes fichiers qu'à la première
demi-journée. Si `verifier.py` ou `docker-compose.yml` diffèrent des tiens, prends ceux-ci
(en gardant, dans le compose, le service `api` que tu as décommenté).

## Lancer les slides

```bash
cd slides
python -m http.server 8001
```

Puis ouvre <http://127.0.0.1:8001>.

## Programme de la demi-journée

| Théorie (une heure) | TP en autonomie |
|---|---|
| Infrastructure as Code et NetDevOps, YANG et RESTCONF, API des clouds, authentification et secrets, validation et injection, webhooks | TP4 écriture (~1h15) · TP5 authentification (~45′) · TP6 inventaire NetBox (~45′) · TP7 webhook (~45′) |

Le programme est dense : ce qui n'est pas terminé en séance se finit avant la dernière
demi-journée, qui est courte (un TP, le quiz, le rendu).

## Arborescence

```
.
├── slides/    supports Reveal.js des demi-journées 1 et 2
└── projet/    sujet, environnement, contrat, vérificateur, surveillant
```
