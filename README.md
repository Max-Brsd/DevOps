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


## Question 2-1 : What are testcontainers ?

Les conteneurs de tests sont les différents modules que va utiliser l'api backend. Ici jdbc et postgresql.

## Question 2-2 : Document your Github Actions configurations :

```yml
name: CI devops 2023
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: master # On va lancer une CI que lorsque l'on reçoit des changements sur la branche main
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3 # On va utiliser la V3 de java pour avoir JDK 17
        with:
          java-version: '17' # On spécifie la version 17
          distribution: 'adopt' 

     #finally build your app with the latest command
      - name: Build and test with Maven
        run: | 
              cd simple-api 
              mvn clean test
      # On se déplace dans le dossier ou le pom.xml est présent et on lance un clean et des tests.
```