# Partie A — Lancement manuel

## Commandes

```bash
# 1. Build de l'image de l'API (cible prod)
docker build --target prod -t td3-api:prod app/
docker images td3-api        # l'image apparait : 184MB

# 2. Volume nomme + Postgres
docker volume create td3-pgdata
docker run -d --name db \
  -e POSTGRES_DB=visites -e POSTGRES_USER=app -e POSTGRES_PASSWORD=s3cret \
  -v td3-pgdata:/var/lib/postgresql \
  postgres:18-alpine
docker logs db               # ... database system is ready to accept connections

# 3. Redis
docker run -d --name cache redis:8-alpine

# 4. API (premier essai : SANS reseau)
docker run -d --name api -p 3000:3000 \
  -e DB_HOST=db -e DB_NAME=visites -e DB_USER=app -e DB_PASSWORD=s3cret \
  -e REDIS_URL=redis://cache:6379 \
  td3-api:prod
docker logs api
# -> Error: getaddrinfo ENOTFOUND db  (le conteneur meurt, exit 1)
```

## A1 — Pourquoi l'API a plante, et la correction

Sans option de reseau, les conteneurs vont tous sur le **bridge par defaut**, qui
ne fait **pas de resolution DNS par nom de conteneur** : `db` et `cache` sont
donc introuvables (`ENOTFOUND`). On aurait pu joindre l'IP du conteneur, mais
elle change a chaque recree.

**Correction** : creer un reseau utilisateur et y placer les **trois** conteneurs
(le DNS embarque de Docker n'existe que sur les reseaux utilisateur) :

```bash
docker rm -f api db cache
docker network create td3-net

docker run -d --name db --network td3-net \
  -e POSTGRES_DB=visites -e POSTGRES_USER=app -e POSTGRES_PASSWORD=s3cret \
  -v td3-pgdata:/var/lib/postgresql \
  postgres:18-alpine
docker run -d --name cache --network td3-net redis:8-alpine
docker run -d --name api --network td3-net -p 3000:3000 \
  -e DB_HOST=db -e DB_NAME=visites -e DB_USER=app -e DB_PASSWORD=s3cret \
  -e REDIS_URL=redis://cache:6379 \
  td3-api:prod

curl localhost:3000
# {"hitsInRedis":1,"visitsInPostgres":1,"servedBy":"8dd2f80f8958"}
```

## A2 — Survie des compteurs apres suppression/recree des 3 conteneurs

Apres 3 `curl` puis `docker rm -f api db cache` et recree des trois :

```json
{"hitsInRedis":1,"visitsInPostgres":4,"servedBy":"f01b3f9404b3"}
```

- **Postgres a survecu** (3 -> 4) : ses donnees sont dans le **volume nomme**
  `td3-pgdata`, qui existe independamment du conteneur.
- **Redis est reparti a 1** : il n'ecrit nulle part par defaut — le compteur vit
  en memoire dans la couche writable du conteneur, detruite avec lui.
