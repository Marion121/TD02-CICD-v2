# TD02-CICD-v2
## Doker 
### database container 

#### Dockerfiled :
```
FROM postgres:14.1-alpine

COPY /sql /docker-entrypoint-initdb.d

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd
```
#### Principales commandes : 
* ```docker network create app-network ``` : creation du network
* ``` docker run -p "8090:8080" --net=app-network --name=adminer -d adminer``` : run Adminer en precisant le network et le port
* ```docker run -p 8888:5000 --name postgres —network app-network marionchineaud/postgres``` : run la database en precisant le network et le port

### build et run une application avec docker 

Ici, on utilise un multi-stage build car cela permet d'optimiser la taille des images Docker finales en séparant les étapes de construction des étapes de déploiement. Grâce à cela, on peut réduire la taille des images finales, le temps d'exécution et améliorer la sécurité.

#### étape 1 : build 
Récupération l'image de base ici amazoncorretto-17 étiquetée myapp-build :
```
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
```
Definition des variables d'environnement :
```
ENV MYAPP_HOME /opt/myapp
```
Définition du répertoire de travail dans le conteneur :
```
WORKDIR $MYAPP_HOME
```
Copie du fichier pom.xml et du répertoire src dans le répertoire de travail :
```
COPY pom.xml .
COPY src ./src
```
Exécution de la construction de l'application :
```
RUN mvn package -DskipTests
```

#### étape 2 : run 

Dans cette seconde partie, c'est la même idée : on récupère l'image de base, on définit une variable d'environnement, puis on définit le répertoire de travail et on copie des fichiers dedans. 
```
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
```
Définition de la commande d'entrée pour exécuter l'application :
```
ENTRYPOINT java -jar myapp.jar
```

### Docker compose 
#### Principales commandes docker compose 

* ```docker-compose up``` : Démarrage les conteneurs définis dans le fichier docker-compose.yaml
* ```docker-compose build``` : Construction des les images docker définies dans le fichier docker-compose.yaml
* ```docker-compose exec``` : Exécution d'une commande à l'intérieur d'un conteneur
* ```docker-compose -d``` : Démarrage en mode détaché
  
#### Docker compose 
```
version: '3.7'
services:
```
Création du conteneur de la base de donnée en récupérent les variables secrétes du .env, puis connexion au network et au volume :
```
  database:
    build:
      dockerfile: Dockerfile
    container_name: postgres
    env_file:
      - .env
    networks:
      - app-network
    volumes:
      - db-app:/var/lib/postgresql/data
```
Création du conteneur de l'api (connecté au network) uniquement si la base de données a démarré :
```
  backend:
    build:
      context: /simple-api-student
      dockerfile: Dockerfile
    container_name: api3
    networks:
      - app-network
    depends_on:
      - database
```
Création du proxy (en indiquant le port et le network) uniquement si l'api a démarré :
```
  httpd:
    build:          
      context: /frontend
      dockerfile: Dockerfile
    ports:
      - "8089:80"
    networks:
      - app-network
    depends_on:
      - backend
```
Création d'un network pour permettre aux conteneurs de communiquer : 
```
networks:
  app-network:
```
Création d'un volume pour ne pas perdre les données de la base de donnée :
```
volumes:
  db-app:
```
### Publication des images 

Push des images sur un registre distant (docker Hub) permet de les rendre accessible à d'autres utilisateurs ou à d'autres machiens.

#### Principales commandes pour publier des images sur dockerHub

```docker tag docker-tp1-database marionchineaud/docker-tp1-database:1.0.0``` : Donne un tag à une image docker existante, ici le tag est 1.0.0 . 

```docker push marionchineaud/docker-tp1-database:1.0.0``` : push l'image docker vers notre registre distant (docker Hub).


## Github Action

###  Testcontainers

ce sont des  bibliothèques Java qui facilitent l'utilisation de conteneurs Docker dans des scénarios de test.

### Configurations de github actions 
Initialisation :
* Création d'un repository github
* Création des dossiers .github/workflows avec le fichier main.yml
  
contenue du fichier main.yml :
```
name: CI-CD
on:
 push:
 branches:
     - master
     - develop
jobs:
 test-backend:
 runs-on: ubuntu-22.04
 steps:
   - name: Checkout code
     uses: actions/checkout@v2.5.0
   - name: Set up JDK 17
     uses: actions/setup-java@v3
     with:
       distribution: 'adopt'
       java-version: '17'
   - name: Build and test with Maven
 working-directory : simple-api-student
 run: mvn clean verify
```
A chaque fois que l'on push cela relance une pipeline si elle entre dans les conditions cité dans le main.yml. 

### Configurations des quality gate 

* Création d'un compte Sonar Cloud
* Récupèration des project key & organization key
* Création des secrets correspondants dans gitHub.
* Modification de main.yml, ajout de :
```run: mvn -B verify sonar:sonar -Dsonar.projectKey=test-marmar_devops -Dsonar.organization=test-marmar -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  -f ./pom.xml```

## Ansible
### Setup
Initialisation : 

Création des dossiers .ansible/inventories avec le fichier setup.yml

Contenue du fichier setup.yml :
```
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: ../../id_rsa
 children:
   prod:
     hosts: marion.chineaud.takima.cloud
```
Principales commandes ansible :

* ```ansible all -i inventories/setup.yml -m ping``` : permet de tester la connectivité 
* ```ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"``` :  utilise le module "setup" pour collecter des informations sur les hôtes, en filtrant spécifiquement les informations liées à la distribution du système
  
### Playbooks 
Initialisation : 

* Création du fichier playbooks.yml dans le dossier .ansible/inventories
* Création du dossier roles
  
Contenue du fichier playbooks.yml : 
```
- hosts: all # definit les hôtes
  gather_facts: false
  become: yes # désactive la collecte des facts
  roles: # rôles Ansible à exécuter dans un ordre précis
    - docker
    - networks
    - database
    - api
    - proxy

```
Principales commandes ansible :

* ```ansible-playbook -i inventories/setup.yml playbook.yml``` : exécute les tâches et les rôles décrits dans le playbook sur les hôtes definit dans le setup 

### Configuration des docker_container tasks 

* Nettoie les paquets YUM sur le système cible : 
```
- name: Clean packages
  command:
    cmd: yum clean -y packages
```
* Installe le paquet "device-mapper-persistent-data" :
```
- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest
```
* Installe le paquet "lvm2" :
```
- name: Install lvm2
  yum:
    name: lvm2
    state: latest
```
* Ajoute le référentiel Docker au système :
```
- name: add repo docker
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
```
* Installe le paquet "docker-ce" avec l'état "present", ce qui signifie que Docker sera installé s'il n'est pas déjà présent :
```
- name: Install Docker
  yum:
    name: docker-ce
    state: present
```
* Assure que le service Docker est démarré :
```
- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```
