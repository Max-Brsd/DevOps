# DockerFile

```sh
FROM postgres:14.1-alpine

COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d
COPY 02-InsertData.sql /docker-entrypoint-initdb.d
```

**FROM** va permettre de sélectionne l'image postgres.
**COPY** va permettre de copier le fichier présent sur la machine dans le répertoire destination du docker.

## Commandes

On va build l'image docker avec la commande suivante : 
```shell
docker build -t maxime/postgres .
```

Puis on execute le conteneur postgresql avec la commande suivante : 
```shell
docker run -p 8888:5432 --net=app-network -e "POSTGRES_DB=db" -e "POSTGRES_USER=usr" -e "POSTGRES_PASSWORD=pwd" --name postgres -v /data:/var/lib/postgresql/data maxime/postgres
```

Pour lancer l'adminer on va executer la commande suivante : 
```shell
docker run -p "8090:8080" --net=app-network --name=adminer -d adminer
```

Pour faire communiquer les deux container entre eux il faut créer un network que l'on va lier à chaque container. Pour créer le network on va utiliser la commande suivante :

```shell
docker network create app-network
```

# Backend-API

La build en plusieurs étapes permet de réduire la taille de l'image Docker finale en séparant les dépendances de construction (Maven, code source, etc.) de l'environnement d'exécution (JRE, JAR d'application). La première étape crée les artefacts nécessaires, et la deuxième étape ne copie que les artefacts essentiels dans l'image finale. Cette façon de faire garantit que l'image finale ne contient que ce qui est nécessaire à l'exécution de l'application, ce qui permet d'obtenir une image Docker plus petite et plus efficace et plus légère.




### Docker compose build : 

```
version: '3.7'

services:
    backend:
        container_name: database #Oblige le container à avoir ce nom
        build: ./simple-api # Endroit ou se trouve le Dockerfile du Backend
        networks:
          - app-network # Définis le network à utiliser
        depends_on:
          - database # Ne build pas normalement si le service database n'est pass déja présent

    database:
        container_name: database
        build: ./Postgres
        networks:
          - app-network
        volumes:
          - /data:/var/lib/postgresql/data

    httpd:
        container_name: httpd
        build: ./https-server
        ports:
          - 8081:80
        networks:
          - app-network
        depends_on:
          - backend

networks:
    app-network: 

```
#### Commande principale docker compose :
**docker compose up** permet de génerer les builds et de monter les container
**docker compose down** permet de stopper les container et de les supprimer

### Docker compose image :

```docker
version: '3.7'

services:
    backend:
        container_name: simpleapi
        image: planche69/devops-backend:1.0
        networks:
          - app-network
        depends_on:
          - database

    database:
        container_name: database
        image: planche69/devops-database:1.0
        networks:
          - app-network
        volumes:
          - /data:/var/lib/postgresql/data

    httpd:
        container_name: httpd
        image: planche69/devops-httpd:1.0
        ports:
          - 8081:80
        networks:
          - app-network
        depends_on:
          - backend

networks:
    app-network: 

```

### Commande pour publish :

```docker tag maxime/simpleapi planche69/devops-backend:1.1```
**maxime/simpleapi** est le nom de l'image que l'on a build en local et le nom que l'on met ensuite est celui qui sera présent sur DockerHub.

```docker push planche69/devops-backend:1.1```
Cette commande permet de push notre image avec le nom que l'on veut en ligne.