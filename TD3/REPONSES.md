# REPONSES — TD3

## B1

`docker compose config` montre les variables interpolees depuis `.env`
(`DB_PASSWORD: s3cret-td3`, `POSTGRES_PASSWORD: s3cret-td3`). Aucun mot de passe
n'est en dur dans `compose.yaml` ; le `.env` n'est pas commite
(seul `.env.example` l'est).

## B2 — Crash de l'API au premier demarrage

Logs au premier `docker compose up -d` :

```
Error: connect ECONNREFUSED 172.19.0.3:5432
```

`depends_on` attend seulement que le conteneur `db` soit **demarre**, pas que
Postgres soit **pret** a accepter des connexions (initialisation du cluster =
quelques secondes). L'API tente la connexion trop tot -> crash, exit 1.

Avec `restart: on-failure`, Docker relance le conteneur en erreur ; au second
essai Postgres est pret et l'API demarre (`docker compose ps` : tous `Up`).

C'est un **contournement** : l'application plante toujours une fois (bruit dans
les logs, code de sortie en erreur), ca repose sur le timing, et rien ne garantit
que la deuxieme tentative reussisse. La vraie solution est un `healthcheck` sur
`db` avec `depends_on: condition: service_healthy` — vu plus tard dans le module.

## B3 — Persistance des donnees

Donnees avant : `{"hitsInRedis":3,"visitsInPostgres":5}`.

### down puis up — les donnees survivent

```
docker compose down && docker compose up -d
curl localhost:3000
{"hitsInRedis":4,"visitsInPostgres":6,"servedBy":"d79b263d1276"}
```

Les volumes nommes `td3_pgdata` et `td3_redisdata` ne sont pas supprimes par
`down`, donc Postgres ET Redis retrouvent leurs donnees.

### down -v — tout disparait

```
docker compose down -v
# Volume td3_pgdata   Removed
# Volume td3_redisdata Removed
docker compose up -d
curl localhost:3000
{"hitsInRedis":1,"visitsInPostgres":1,"servedBy":"b7ea894b74f3"}
```

`-v` supprime les volumes nommes -> compteurs reinitialises.

### Pour que Redis survive

`docker compose exec cache redis-cli CONFIG GET dir` -> `/data` : c'est la que
Redis ecrit son dump/AOF. Dans `compose.yaml` :

```yaml
cache:
  image: redis:8-alpine
  command: redis-server --appendonly yes
  volumes:
    - redisdata:/data
```

Le volume **seul ne suffit pas** : il faut aussi `--appendonly yes` (ou un `save`),
sinon Redis garde tout en memoire et n'ecrit jamais dans /data.

## B4 — getent hosts db

```
docker compose exec api getent hosts db        -> 172.19.0.3  db
docker compose up -d --force-recreate db
docker compose exec api getent hosts db        -> 172.19.0.3  db
```

Ici l'IP est la meme (le bail vient d'etre libere, le DHCP Docker la reattribue),
mais **rien ne le garantit** : les IP sont allouees dynamiquement a la creation
des conteneurs. Ecrire une IP en config est fragile : des qu'elle change, l'app
casse sans message d'erreur evident. La reference stable est le **nom de service**
`db`, resolu a chaque appel par le DNS embarque de Docker.

## C1 — Environnement dev

`compose.override.yaml` : cible `dev`, `develop.watch` (sync `./app/src` ->
`/app/src`, `rebuild` sur `package.json`), port de db publie sur
`${DB_PORT_EXT:-5432}:5432`.

Preuve de synchro sans rebuild :

```
docker compose up -d --build && docker compose watch
# modification de /health dans app/src/server.js -> "Syncing service api"
curl localhost:3000/health
{"status":"UP","mode":"dev-watch"}
```

La modif est visible : `watch` a copie le fichier dans le conteneur et
`node --watch` a recharge le serveur.

## C2 — Bascule dev -> prod

`docker compose -f compose.yaml up -d` **sans** `--build` recree l'API depuis
l'image `td3-api` existante — qui etait la cible **dev** du dernier build :

```
docker inspect td3-api-1 --format 'CMD={{.Config.Cmd}} USER={{.Config.User}}'
CMD=[npm run dev] USER=          <- image dev, root, node --watch
```

Les deux configurations partagent le meme tag d'image : Compose ne reconstruit
pas spontanement, il reutilise ce qui existe. Sans `--build`, on fait donc
tourner la mauvaise image (dev au lieu de prod : root, watch, devDependencies).

Correction : `docker compose -f compose.yaml up -d --build`

```
docker inspect td3-api-1 --format 'CMD={{.Config.Cmd}} USER={{.Config.User}}'
CMD=[node src/server.js] USER=node     <- prod, non-root
```
