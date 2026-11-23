# API Rest pour l'administration système

<p class="lead">Demi-journée 2 : écrire, sécuriser, réagir</p>

<p class="small">Maximilien CRUZ</p>

---

## Où vous en êtes

<div class="left">

Votre API sait **lire** : `/health`, la liste des conteneurs gérés, le détail de l'un d'eux.
Elle tourne dans son conteneur et passe le bloc `lecture` du vérificateur.

</div>

<div class="memo">

**Si le bloc `lecture` n'est pas au vert**, c'est votre premier objectif de la séance, avant
le TP4. Tout ce qui suit s'appuie dessus.

Vérifiez d'abord que l'environnement tourne : `docker compose up -d`, puis `docker compose ps`.

</div>

---

## Aujourd'hui, votre API devient dangereuse

<p class="lead">Une API qui lit ne casse rien. Une API qui <strong>crée, arrête et supprime</strong> doit savoir <strong>qui</strong> lui parle et <strong>quoi</strong> on lui envoie.</p>

<div class="left">

| Durée | |
|---|---|
| 15′ | Infrastructure as Code, NetDevOps, équipements réseau pilotés par modèle |
| 8′ | Les API des clouds et des hyperviseurs |
| 15′ | Authentification : clés, tokens, JWT, OAuth2, secrets |
| 10′ | Valider les entrées, bloquer l'injection |
| 10′ | Les webhooks |
| ~1h15 | **TP4** : écriture |
| ~45′ | **TP5** : authentification |
| ~45′ | **TP6** : inventaire NetBox |
| ~45′ | **TP7** : webhook d'auto-remédiation |

</div>
