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

## Question 2-3 : Document your quality gate configuration :

On a ajouté en secrets : **SONAR_TOKEN** qui est le token que l'on a ajouté directement via Sonar. Ensuite l'enrichissement se fait directement via les CIs. 

Puis on a modifié la ligne suivante dans le fichier pour les CI afin d'envoyer le resultat à Sonar :
```yaml
     #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=devopscpe_devops -Dsonar.organization=devopscpe -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./simple-api/pom.xml
    # define job to build and publish docker image
```


## Question 3-1 : Document your inventory and base commands:

### __Inventory__

```yaml
all:
  vars:
    ansible_user: centos # L'utilisateur que nous allons utiliser dans notre connexion ssh
    ansible_ssh_private_key_file: /id_rsa # La ou se site notre clé RSA
  children:
    prod:
      hosts: maxime.brossard.takima.cloud # Le NDD pour accéder à notre serveur
```

### __Base command__

```shell
ansible all -i inventories/setup.yml -m ping
```
Le paramètre **all** de ansible va passer sur tous les hosts présent dans l'inventaire. Ensuite le paramètre **-i** permet de signifier quel fichier d'inventaire nous allons utiliser, ici *inventories/setup.yml*. Puis le paramètre **-m** nous sert à indiquer le module à éxécuter, ici c'est *ping*.


#### Command:

```shell
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```

Le paramètre **all** de ansible va passer sur tous les hosts présent dans l'inventaire. Ensuite le paramètre **-i** permet de signifier quel fichier d'inventaire nous allons utiliser, ici *inventories/setup.yml*. Puis le paramètre **-m** nous sert à indiquer le module à éxécuter, ici c'est *setup*. Ce module nous permet d'avoir des informations concernant l'hôte. Le paramètre **-a** permet de rajouter des arguments au module à exécuter, ici c'est *"filter=ansible_distribution\*"*

#### Response: 

```json
maxime.brossard.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution": "CentOS",
        "ansible_distribution_file_parsed": true,
        "ansible_distribution_file_path": "/etc/redhat-release",
        "ansible_distribution_file_variety": "RedHat",
        "ansible_distribution_major_version": "7",
        "ansible_distribution_release": "Core",
        "ansible_distribution_version": "7.9",
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false
}
```

#### Command:

```shell
ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become
```

Le paramètre **all** de ansible va passer sur tous les hosts présent dans l'inventaire. Ensuite le paramètre **-i** permet de signifier quel fichier d'inventaire nous allons utiliser, ici *inventories/setup.yml*. Puis le paramètre **-m** nous sert à indiquer le module à éxécuter, ici c'est *yum*. Ce module nous permet d'installer ou de désinstaller des fonctionnalitées. Le paramètre **-a** permet de rajouter des arguments au module à exécuter, ici c'est *name=httpd state=absent* le **=absent** veut dire que l'on veut désinstaller ce module de la machine. Et enfin le **--become** nous permet d'exécuter tous ça en super utilisateur

#### Response: 

```json
maxime.brossard.takima.cloud | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": true,
    "changes": {
        "removed": [
            "httpd"
        ]
    },
    "msg": "",
    "rc": 0,
    "results": [
        "Loaded plugins: fastestmirror\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-99.el7.centos.1 will be erased\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package      Arch          Version                       Repository       Size\n================================================================================\nRemoving:\n httpd        x86_64        2.4.6-99.el7.centos.1         @updates        9.4 M\n\nTransaction Summary\n================================================================================\nRemove  1 Package\n\nInstalled size: 9.4 M\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Erasing    : httpd-2.4.6-99.el7.centos.1.x86_64                           1/1 \n  Verifying  : httpd-2.4.6-99.el7.centos.1.x86_64                           1/1 \n\nRemoved:\n  httpd.x86_64 0:2.4.6-99.el7.centos.1                                          \n\nComplete!\n"
    ]
}
```

## Question 3-2 : Document your playbook :

### __Main.yml__:

```yaml
- hosts: all # Définition des hosts à utiliser
  gather_facts: false
  become: true # Connexion en root

  roles: # Liste des différents roles à utiliser
    - docker
    - network
    - database
    - app
    - proxy
```

### __App roles playbook Tasks__ :

```yaml
- name: Run simpleapi # Lors de l'exécution on verra ce nom s'afficher
  docker_container: # On va créer un container docker grâce à cette ligne
    name: simpleapi # Cette ligne sera le nom du container
    image: planche69/devops-backend:latest # Cette ligne va choisir quelle image va run sur le container
    env: # Définition des variables d'environnement
      DB_URL: "{{ DB_URL }}"
      DB_USR: "{{ DB_USR }}"
      DB_PWD: "{{ DB_PWD }}"
    networks: # Définition du network utilisée par le container
      - name: "app-network"
```

### __Database roles playbook Tasks__ :

```yaml
- name: Run database # Lors de l'exécution on verra ce nom s'afficher
  docker_container: # On va créer un container docker grâce à cette ligne
    name: database # Cette ligne sera le nom du container
    image: planche69/devops-database:latest # Cette ligne va choisir quelle image va run sur le container
    env: # Définition des variables d'environnement
      POSTGRES_DB: "{{ POSTGRES_DB }}"
      POSTGRES_USER: "{{ POSTGRES_USER }}"
      POSTGRES_PASSWORD: "{{ POSTGRES_PASSWORD }}"
    networks: # Définition du network utilisée par le container
      - name: "app-network"
```

### __Proxy roles playbook Tasks__ :

```yaml
- name: Run http # Lors de l'exécution on verra ce nom s'afficher
  docker_container: # On va créer un container docker grâce à cette ligne
    name: httpd # Cette ligne sera le nom du container
    image: planche69/devops-httpd:latest # Cette ligne va choisir quelle image va run sur le container
    ports: # Définition des ports utilisés par le container
      - "80:80"
    networks: # Définition du network utilisée par le container
      - name: "app-network"
```

Les variables d'environnements présent dans les playbooks sont des variables qui ont subi une encryption avec **ansible-vault**. Les variables chiffré sont présentes dans le fichier *secret.yml*. Pour récupérer les secrets lors de l'exécution des playbooks il faudra executer la ligne de commande suivante :

```shell
ansible-playbook -i ../inventories/setup.yml -e @secret.yml --ask-vault-pass main.yml
```