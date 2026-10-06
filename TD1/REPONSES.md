A1 :
La première fois, docker à du pull l'image 'hello-world' sur un repo distant.
La seconde fois l'image était déjà téléchargée en local, donc le lancement à été plus rapide

A2 :
Docker a mappé les ports de notre machine pour faire un bridge entre les ports 80 des machines nginx
respectivement sur les ports 8080 et 8081 de notre machine.
Si on essaie de lancer le deuxième nginx sur le même port que le premier (8080) on obtient une erreur:
impossible de lancer le conteneur car le port est déjà alloué

A3 :
Commande pour voir les logs de web1 :
```
docker logs web1
```

Commande pour voir les logs en continu :
```
docker logs -f web1
```
a chaque fois que l'on rafraichis localhost, plusieurs nouvelles lignes apparaissent dans les logs

A4 :
La modification a disparu après la supression et le restart du conteneur. c'est du au fait que nous
avons modifié le conteneur et non l'image. Docker exec ne permet donc pas de faire des modifications
définitives

B1 :
Dans le premier cas, mon prenom s'affiche, en revanche dans le second la commande échoue car 
docker ne comprend pas que PRENOM est une variable d'environnement sans l'option -E. Une variable 
passée avec -e ne vit que dans le conteneur

B2 :
Non, curl n'est plus la après le second lancement car on l'as intallé dans notre premier conteneur
jetable, qui a été détruit depuis.

C1 :
node:24 : 1,14GB, 666 commandes, tous les outils
node:24-slim : 230MB, 275 commandes, aucun outils
node:24-alpine : 171MB, 143 commandes, aucun outils

C2 :
il y a 9 couche, la plus lourde est celle la :
```RUN /bin/sh -c addgroup -g 1000 node ```

C3 : 
cette commande est lancée au démarrage du conteneur :
```nginx -g daemon off```
Le port 80 est exposé, c'est cohérent avec le mapping qu'on a fait en A1

D1: 
docker ps affiche bien quelquechose

D2:
Nginx écoute sur le port 80 dans son conteneur pas 8080. 
Le mapping -p 9082:8080 redirige le port 9082 de l'hôte vers le port 8080 du conteneur,
où aucun processus n'écoute

D3:
le conteneur a mis 10 secondes à s'arrêter et a sorti le code 137
L'arrêt a été long car c'est un arrêt "propre" mais que sleep était en train de tourner donc
donc le conteneur n'as pas pu l'arrêter proprement

E1:
Les images sur mon docker occupent 2GB
```docker system df```
supression des conteneurs du TP
``` docker rm web1 web2 dormeur dormeur2 gourmand ```
Docker prend toujours 2GB, c'est les images qui prennent de la place pas les conteneurs