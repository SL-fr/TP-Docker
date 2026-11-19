# Project SIOSISR

## TP Docker / Introduction à Docker

Prise en main de Docker : containers, images, Dockerfile et orchestration avec Docker Compose.

---

## 1. Contexte et objectifs

Ce TP (BTS SIO SISR) présente les bases de Docker. Il comporte quatre objectifs :

- **Installer Docker** sur une Debian 12 et valider l'installation avec `hello-world`.
- **Manipuler images et containers** : lancer nginx et Apache depuis Docker Hub, lister, arrêter et supprimer.
- **Créer ses propres images** avec un `Dockerfile`, puis comparer une image Debian et une image Alpine.
- **Orchestrer des containers** avec Docker Compose : une application Flask et sa base Redis.

Ce compte rendu présente la démarche, les commandes utilisées et les validations (captures d'écran dans le dossier `IMG docker/`).

---

## 2. Infrastructure mise en place

### Environnement

| Élément | Valeur |
| --- | --- |
| Système | Debian 12 (machine virtuelle) |
| Accès | SSH avec PuTTY, port 22, vers `192.168.20.x`, utilisateur `etudiant` (groupes `sudo` et `docker`) |
| Docker | Installé depuis le dépôt officiel Docker, puis `docker-compose` |

---

## 3. Installation de Docker

Installation depuis le dépôt officiel :

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo docker run hello-world
```

![hello-world](IMG%20docker/01-hello-world.png)

Pour utiliser `docker` sans `sudo`, ajout de l'utilisateur au groupe `docker` puis reconnexion SSH :

```bash
sudo usermod -aG docker etudiant
groups
docker ps
```

![groupe docker](IMG%20docker/02-groupe-docker.png)

Versions installées :

```bash
docker -v
docker-compose -v
```

![versions](IMG%20docker/03-versions.png)

---

## 4. Images et containers

### 4.1 nginx

```bash
docker run -p 8080:80 -d nginx
docker ps
curl localhost:8080
```

| Option | Rôle |
| --- | --- |
| `run` | Crée et lance un container (télécharge l'image si besoin) |
| `-p 8080:80` | Mappe le port 8080 de l'hôte vers le port 80 du container |
| `-d` | Mode détaché (arrière-plan) |
| `nginx` | Image utilisée |

![nginx](IMG%20docker/04-nginx.png)

### 4.2 Fonction des commandes

| Commande | Fonction |
| --- | --- |
| `docker ps` | Liste les containers en cours d'exécution |
| `docker ps -a` | Liste tous les containers, y compris arrêtés |
| `docker images` | Liste les images présentes en local |

![ps et images](IMG%20docker/05-ps-a-images.png)

### 4.3 Arrêter et supprimer

| Action | Commande |
| --- | --- |
| Arrêter un container | `docker stop <nom_ou_id>` |
| Supprimer un container | `docker rm <nom_ou_id>` |
| Supprimer une image | `docker rmi <image>` |

Après l'arrêt, `docker ps` est vide alors que `docker ps -a` montre le container avec le statut `Exited (0)`.

![stop](IMG%20docker/06-stop.png)

![rm et rmi](IMG%20docker/07-rm-rmi.png)

### 4.4 Apache

```bash
docker pull httpd
docker run -d -p 8081:80 --name apache httpd
docker ps
curl localhost:8081
```

`docker pull` télécharge seulement l'image, alors que `docker run` crée et démarre un container.

![apache](IMG%20docker/08-apache.png)

---

## 5. Création d'une image

### 5.1 Image `btssio` basée sur Debian

```bash
mkdir ~/lab0 && cd ~/lab0
nano Dockerfile
```

```dockerfile
FROM debian:stable-slim
LABEL maintainer="BTS SIO2 SISR"
RUN apt-get update \
    && apt-get install -y vim git htop mc tree \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

> Le sujet annonce l'outil `tree` sans l'installer : il a été ajouté. `MAINTAINER`, déprécié, est remplacé par `LABEL maintainer`.

![Dockerfile Debian](IMG%20docker/09-dockerfile-debian.png)

Construction et vérification de la taille :

```bash
docker build -t btssio:v1.0 .
docker images
```

![fin du build Debian](IMG%20docker/10a-build-debian.png)

![taille de l'image Debian](IMG%20docker/10b-images-debian.png)

Lancement et test des outils :

```bash
docker run -tid --name sisr btssio:v1.0
docker exec -ti sisr bash
```

![outils Debian](IMG%20docker/11-outils-debian.png)

### 5.2 Image plus légère avec Alpine

L'indice « France en Formule 1 » renvoie à l'écurie **Alpine**, donc à **Alpine Linux**, une distribution minuscule.

```bash
mkdir -p ~/lab1 && cd ~/lab1
nano Dockerfile
```

```dockerfile
FROM alpine:latest
LABEL maintainer="BTS SIO2 SISR"
RUN apk add --no-cache vim git htop mc tree
```

![Dockerfile Alpine](IMG%20docker/12-dockerfile-alpine.png)

```bash
docker build -t btssio:v2.0 .
docker images
docker run -tid --name sisr2 btssio:v2.0
docker exec -ti sisr2 sh
```

> Alpine n'a pas `bash` par défaut : on utilise `sh`.

### 5.3 Comparaison des tailles

| Image | Base | Taille |
| --- | --- | --- |
| `btssio:v1.0` | `debian:stable-slim` (78,9 MB) | 266 MB |
| `btssio:v2.0` | `alpine:latest` (8,43 MB) | 66 MB |

L'image Alpine est environ quatre fois plus légère, avec les mêmes outils.

![build Alpine](IMG%20docker/13a-build-alpine.png)

![comparaison des tailles](IMG%20docker/13b-comparaison-tailles.png)

![container Alpine](IMG%20docker/14a-container-alpine.png)

![outils Alpine](IMG%20docker/14b-outils-alpine.png)

Nettoyage de fin de partie :

```bash
docker rm -f sisr sisr2
docker rmi btssio:v1.0 btssio:v2.0
docker ps -a
```

![nettoyage](IMG%20docker/15-nettoyage.png)

---

## 6. Docker Compose

### 6.1 Application

L'application Flask affiche le hostname, un nom lu dans la variable d'environnement `NOM` et un compteur de visites stocké dans Redis.

Fichiers du dossier `~/projet` : `app.py`, `requirements.txt` (Flask, Redis) et le `Dockerfile` :

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . /app
RUN pip install -r requirements.txt
EXPOSE 80
ENV NOM="BTS SIO2 SISR"
CMD ["python", "app.py"]
```

> Le sujet utilise `python:2.7-slim`. Python 2.7 étant obsolète et incompatible avec les versions récentes de Flask et Redis, l'image `python:3.11-slim` a été utilisée.

| Instruction | Rôle |
| --- | --- |
| `FROM` | Image source |
| `WORKDIR` | Répertoire de travail dans le container |
| `COPY` | Copie les fichiers du projet dans l'image |
| `RUN` | Exécute une commande à la construction |
| `EXPOSE` | Port d'écoute par défaut |
| `ENV` | Variable d'environnement |
| `CMD` | Commande lancée au démarrage |

```bash
ls
docker build -t monimage .
```

![fichiers et build](IMG%20docker/16-fichiers-build.png)

Sans Redis, la page s'affiche mais le compteur est désactivé : il faut un second container relié au premier.

### 6.2 Fichier `docker-compose.yml`

```yaml
version: "3"
services:
  monapp:
    image: monimage
    hostname: SaintLuc
    depends_on:
      - redis
    ports:
      - "80:80"
    networks:
      - monreseau
    environment:
      - NOM=les SISR
  redis:
    image: redis
    networks:
      - monreseau
networks:
  monreseau:
```

- `depends_on` : `monapp` démarre après `redis`.
- `networks` : les deux services partagent `monreseau`, ce qui permet à Flask de joindre Redis par son nom de service.
- `environment` : `NOM=les SISR` remplace la valeur du Dockerfile.
- `hostname: SaintLuc` : modification demandée en fin de sujet, le hostname affiché est `SaintLuc`.

![docker-compose.yml](IMG%20docker/17-docker-compose-yml.png)

### 6.3 Lancement et validation

```bash
docker-compose up -d
docker ps
```

![compose up](IMG%20docker/18-compose-up.png)

```bash
curl localhost
curl localhost
```

La page affiche `Hello les SISR!`, `Hostname: SaintLuc` et un compteur qui augmente à chaque appel.

![validation](IMG%20docker/19-validation-curl.png)

---

## 7. Nettoyage et conclusion

```bash
docker-compose down
docker rmi monimage redis python:3.11-slim
docker ps -a
docker images
```

![nettoyage final](IMG%20docker/20-nettoyage-final.png)

Ce TP a permis de lancer des containers depuis Docker Hub, de construire deux images avec un Dockerfile (266 MB avec Debian, 66 MB avec Alpine) et d'orchestrer une application Flask et sa base Redis avec Docker Compose. Le choix de l'image de base a un effet direct sur la taille, et le réseau défini dans le fichier Compose permet aux containers de communiquer par leur nom de service.
