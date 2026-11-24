# shipit-starter

Dépôt de départ du fil rouge du cours **Intégration et Déploiement Continu** (ESGI 4).
Petite API de tâches (Node.js 24, TypeScript, Fastify, Vitest). **Aucun workflow CI : c'est votre travail.**

## Qu'est-ce que ShipIt ?

**ShipIt** est une petite API web qui gère une liste de tâches (une « to-do list ») : on peut créer une tâche, la lister, la marquer comme faite, changer son titre ou la supprimer. Elle répond en **JSON** sur HTTP.

L'application est volontairement simple : ce qui compte dans ce cours, ce n'est pas ce qu'elle fait, mais **comment on la livre**. Tout au long des cinq séances, c'est cette même application que vous allez tester automatiquement, empaqueter dans une image Docker, publier, déployer et mesurer.

| Séance | Ce que vous faites sur ShipIt                                                                                                         |
| ------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| 1      | Premier pipeline : lancer les tests à chaque push et à chaque pull request                                                            |
| 2      | Pipeline complet : lint, tests sur plusieurs versions de Node, tests d'intégration avec PostgreSQL, couverture, protection de branche |
| 3      | Image Docker multi-étapes, publication sur GHCR, scan, release                                                                        |
| 4      | Déploiement en staging puis en production, blue/green, canary, retour arrière                                                         |
| 5      | Sécurité (Dependabot, secrets), métriques DORA, projet final                                                                          |

## Démarrer

```bash
npm ci
npm test            # 5 tests unitaires, sans base de données
npm run lint
npm run build
npm run dev         # http://localhost:3000/health
```

`npm run test:integration` (PostgreSQL requis) sera utilisé à partir de la séance 2.
`Dockerfile` volontairement naïf : il sera amélioré en séance 3.

## Les routes de l'API

L'API écoute sur le port `3000` par défaut. Les corps de requête et de réponse sont en JSON.

| Méthode  | Route        | Rôle                              | Réponses                                                                              |
| -------- | ------------ | --------------------------------- | ------------------------------------------------------------------------------------- |
| `GET`    | `/health`    | Vérifier que l'application répond | `200` `{"status":"ok"}`                                                               |
| `GET`    | `/version`   | Connaître la version déployée     | `200` `{"version":"1.0.0"}`                                                           |
| `GET`    | `/tasks`     | Lister toutes les tâches          | `200` un tableau de tâches (vide au départ)                                           |
| `POST`   | `/tasks`     | Créer une tâche                   | `201` la tâche créée ; `400` `{"error":"title requis"}` si `title` est absent ou vide |
| `GET`    | `/tasks/:id` | Lire une tâche                    | `200` la tâche ; `404` `{"error":"introuvable"}`                                      |
| `PATCH`  | `/tasks/:id` | Modifier le titre et/ou l'état    | `200` la tâche modifiée ; `404` `{"error":"introuvable"}`                             |
| `DELETE` | `/tasks/:id` | Supprimer une tâche               | `204` sans contenu ; `404` si elle n'existe pas                                       |

Une **tâche** a trois champs :

```json
{
  "id": "3fc07e1d-f9ef-479f-874a-88a83664569b",
  "title": "Écrire le workflow",
  "done": false
}
```

- `id` : un identifiant UUID, généré par le serveur.
- `title` : le texte de la tâche (les espaces autour sont retirés à la création).
- `done` : `false` à la création ; `true` une fois la tâche faite.

### `/health` et `/version`, à quoi servent-elles ?

Ces deux routes ne manipulent pas de tâches : elles servent **au pipeline**.

- `/health` répond `ok` tant que l'application tourne. C'est ce qu'interroge un _smoke test_ après un déploiement.
- `/version` renvoie la valeur de la variable `APP_VERSION` (`dev` si elle n'est pas définie). Elle permet de vérifier que **la bonne version** est bien celle qui tourne, et pas l'ancienne (séances 3 et 4).

### Exemples avec `curl`

Lancez `npm run dev` dans un terminal, puis, dans un autre :

```bash
# L'application est-elle vivante ? Quelle version ?
curl http://localhost:3000/health
curl http://localhost:3000/version

# Créer une tâche
curl -X POST http://localhost:3000/tasks \
  -H "content-type: application/json" \
  -d '{"title":"Écrire le workflow"}'

# Lister les tâches
curl http://localhost:3000/tasks

# Marquer une tâche comme faite (remplacez ID par l'id reçu à la création)
curl -X PATCH http://localhost:3000/tasks/ID \
  -H "content-type: application/json" \
  -d '{"done":true}'

# Lire puis supprimer une tâche
curl http://localhost:3000/tasks/ID
curl -X DELETE http://localhost:3000/tasks/ID
```

Une requête invalide donne une erreur lisible :

```bash
curl -i -X POST http://localhost:3000/tasks \
  -H "content-type: application/json" -d '{"title":""}'
# HTTP/1.1 400 Bad Request   {"error":"title requis"}
```

## Où sont stockées les tâches ?

Cela dépend d'une variable d'environnement :

| `DATABASE_URL`                                                   | Stockage                                                 | Usage                                            |
| ---------------------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------ |
| non définie                                                      | **En mémoire** : tout est perdu à l'arrêt du serveur     | Séance 1 : tests unitaires, sans base            |
| définie (ex. `postgres://postgres:shipit@localhost:5432/shipit`) | **PostgreSQL** : la table `tasks` est créée au démarrage | Séances 2 à 4 : tests d'intégration, déploiement |

Avec PostgreSQL, la liste des tâches est triée par titre ; en mémoire, elle suit l'ordre de création.

### Variables d'environnement

| Variable       | Rôle                                               | Valeur par défaut                 |
| -------------- | -------------------------------------------------- | --------------------------------- |
| `PORT`         | Port d'écoute                                      | `3000`                            |
| `APP_VERSION`  | Valeur renvoyée par `/version`                     | `dev`                             |
| `DATABASE_URL` | Adresse de PostgreSQL (active le stockage en base) | non définie : stockage en mémoire |

### Lancer l'API avec sa base de données

Le fichier `docker-compose.yml` démarre l'API **et** PostgreSQL pour le développement local :

```bash
docker compose up --build   # API sur http://localhost:3000, PostgreSQL sur le port 5432
```

## Limites volontaires

ShipIt est un support de cours, pas une application de production :

- aucune authentification : n'importe qui peut lire et modifier les tâches ;
- le titre n'est vérifié qu'à la création (`PATCH` ne contrôle pas le titre) ;
- aucune pagination : `GET /tasks` renvoie tout.

## Contenu du dépôt

| Chemin               | Rôle                                                                                                                                                                      | Séance |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| `src/`               | L'application (`app.ts` : routes ; `tasks.ts` : stockage en mémoire ; `pg-tasks.ts` : stockage PostgreSQL ; `server.ts` : démarrage) et ses tests unitaires (`*.test.ts`) | 1      |
| `test/`              | Tests d'intégration (PostgreSQL requis)                                                                                                                                   | 2      |
| `Dockerfile`         | Image Docker, volontairement naïve : à améliorer                                                                                                                          | 3      |
| `docker-compose.yml` | API + PostgreSQL en local                                                                                                                                                 | 2 à 4  |
| `deploy/`            | Kit de déploiement : smoke test, staging, blue/green, canary (voir `deploy/README.md`)                                                                                    | 4      |
| `tools/dora.py`      | Estime les métriques DORA à partir des déploiements GitHub                                                                                                                | 5      |
