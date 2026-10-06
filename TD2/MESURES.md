| Version | Taille de l'image | Build à froid | Rebuild après modif d'une ligne de code | .env dans l'image ? | Utilisateur |
|---|---|---|---|---|---|
| v1 | 1,18 GB | 4,7 s | 4,5 s | Oui (API_KEY lisible) | root (uid=0) |
| v2 | 1,18 GB | 4,6 s | 3,8 s | Oui | root |
| v3 | 1,18 GB | 11,7 s | 5,8 s | Non | root |
| v4 | 175 MB | 53 s | 2 s | Non | node (uid=1000) |

Q1 :
Les variable d'environnement (dont la clé d'API) sont bien dans l'image.
c'est un problème car si cette image est hébergée sur un repository docker, la clé
d'api sera en ligne.

Q2 :
Le rebuild est plus rapide car npm ci est en cache : la couche ne dépend que de
package.json et package-lock.json qui n'ont pas changé. Docker réutilise cette
couche et ne rejoue que COPY + tsc. Si on modifie package.json, le cache est
invalidé et npm ci retourne (les dépendances peuvent avoir changé).

Q3 :
Exclus : .env (secret), node_modules et dist (régénérés dans le build),
*.Zone.Identifier (métadonnées Windows inutiles), Dockerfile* et .dockerignore
(fichiers de build), *.md (doc) et .git.
La ligne "transferring context" a chuté : moins de fichiers envoyés au démon
Docker, donc contexte plus léger et cache plus stable.

Q4 :
Dans l'image de build mais plus dans la finale : le compilateur TypeScript et
toutes les devDependencies, les sources .ts, l'image de base node:24 complète
(outils Debian : gcc, git, curl...), le cache npm. La finale ne contient que
node:24-alpine + node_modules de prod + dist/.

Q5 :
Les deux conteneurs utilisent la même image td2:v4 avec des MESSAGE différents :
localhost:3001 renvoie "version bleue", localhost:3002 renvoie "version verte".
Ne pas reconstruire garantit que c'est exactement le même artefact qui tourne
(traçabilité, immutabilité) : la config passe par les variables d'environnement
au runtime, pas dans l'image. Rebuilder pour changer un message créerait des
images différentes à tester et déployer séparément.
