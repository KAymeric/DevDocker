A1 :
Grace au health test, lorsque l'on fait un docker compose up, docker nous informe
que tous nos services sont "Healthy". Le double $ sert a intégrer la variable d'environnement
docker `POSTGRES_USER` (${POSTGRES_USER}) en tant que variable d'environnement pour cette
commande (d'ou le deuxième $)

A2 :
La base ne doit contenir aucun port car on ne publie pas forcément la prod et le dev sur les mêmes
ports. Par exemple en dev, on peut avoir un conflit avec une autre projet qui tourne aussi sur le port 80
