# 🛠️ Tutoriel : Installer Wiki.js avec Docker, PostgreSQL et Nginx sur Debian

## 📌 **Table des matières**

1. [Présentation](#-pr%C3%A9sentation)
2. [Prérequis](#-1-pr%C3%A9requis)
3. [Préparer les fichiers Docker](#-2-pr%C3%A9parer-les-fichiers-docker)
4. [Démarrer Wiki.js avec Docker](#-3-d%C3%A9marrer-wikijs-avec-docker)
5. [Migrer une installation existante](#-4-migrer-une-installation-existante)
6. [Configurer Nginx comme reverse proxy](#-5-configurer-nginx-comme-reverse-proxy)
7. [Finalisation et vérifications](#-6-finalisation-et-v%C3%A9rifications)

---

## 📌 **Présentation**

Ce guide explique comment installer **Wiki.js** sur un serveur **Debian** en utilisant **Docker** et **Docker Compose**. Nous allons :

- Déployer **Wiki.js** dans un conteneur Docker
- Utiliser **PostgreSQL** comme base de données
- Configurer **Nginx** comme reverse proxy avec **Let's Encrypt** pour le HTTPS
- Migrer une installation existante vers Docker sans perdre de données

---

## ✅ **1. Prérequis**

Avant de commencer, assure-toi d'avoir :

- Un serveur **Debian** à jour
- **Docker** et **Docker Compose** installés
- Un nom de domaine configuré (ex : `wiki.fenris-shay.net`)

### **Installation de Docker et Docker Compose**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
```

📌 **Vérifier l'installation**

```bash
docker --version
docker-compose --version
```

Ajouter l'utilisateur courant au groupe Docker pour éviter de devoir utiliser `sudo` à chaque commande :

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## 📂 **2. Préparer les fichiers Docker**

Nous allons créer un dossier pour Wiki.js et configurer **Docker Compose**.

```bash
mkdir ~/wiki-docker && cd ~/wiki-docker
mkdir db-data wiki-data
```

### **Créer le fichier `docker-compose.yml`**

```bash
nano docker-compose.yml
```

Ajoute ce contenu :

```yaml
services:
  wiki:
    image: requarks/wiki:latest
    restart: always
    ports:
      - "3000:3000"
    environment:
      DB_TYPE: postgres
      DB_HOST: db
      DB_PORT: 5432
      DB_USER: wikijs
      DB_PASS: mypassword
      DB_NAME: wikidb
    depends_on:
      - db
    volumes:
      - ./wiki-data:/wiki-data

  db:
    image: postgres:13
    restart: always
    environment:
      POSTGRES_USER: wikijs
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: wikidb
    volumes:
      - ./db-data:/var/lib/postgresql/data

volumes:
  db-data:
  wiki-data:
```

💾 **Enregistre et ferme (`CTRL + X`, `Y`, `Entrée`).**

---

## 🚀 **3. Démarrer Wiki.js avec Docker**

Exécute cette commande pour démarrer tout le stack Docker :

```bash
docker-compose up -d
```

📌 **Vérifier les conteneurs en cours d'exécution :**

```bash
docker ps
```

Après quelques instants, Wiki.js devrait être accessible sur [**http://localhost:3000**](http://localhost:3000).

---

## 🔄 **4. Migrer une installation existante**

### **Sauvegarde de la base de données actuelle**

Si Wiki.js était déjà installé, il faut sauvegarder la base PostgreSQL existante :

```bash
pg_dump -U wikijs -h localhost -d wikidb > sauvegarde.sql
```

### **Importer la base dans Docker**

Déplacer la sauvegarde dans le dossier `db-data` :

```bash
mv sauvegarde.sql ~/wiki-docker/db-data/
```

Puis importer la base :

```bash
docker exec -i $(docker ps -qf "name=db") psql -U wikijs -d wikidb < ~/wiki-docker/db-data/sauvegarde.sql
```

Vérifier si les utilisateurs sont bien restaurés :

```bash
docker exec -it $(docker ps -qf "name=db") psql -U wikijs -d wikidb -c "SELECT id, email FROM users;"
```

---

## 🌍 **5. Configurer Nginx comme reverse proxy**

### **Créer la configuration Nginx**

```bash
sudo nano /etc/nginx/sites-available/wiki
```

Ajoute ce contenu :

```nginx
server {
    listen 80;
    server_name wiki.fenris-shay.net;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name wiki.fenris-shay.net;

    ssl_certificate /etc/letsencrypt/live/fenris-shay.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/fenris-shay.net/privkey.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

📌 **Activer la configuration et redémarrer Nginx**

```bash
sudo ln -s /etc/nginx/sites-available/wiki /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## 🎯 **6. Finalisation et vérifications**

🚀 **Félicitations, Wiki.js est maintenant installé avec Docker et sécurisé avec Nginx !**
