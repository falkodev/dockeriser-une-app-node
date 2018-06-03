# Dockeriser et déployer une app Node \(1/2\)

L'intérêt de mettre une application au sein d'un conteneur Docker, c'est de pouvoir développer, partager, déployer sans se soucier de l'environnement du serveur. Nous allons prendre l'exemple d'une application Apostrophe, basée sur Node et Mongo, auquel nous ajouterons un serveur web Nginx pour démontrer l'intérêt d'un réseau de conteneurs.

## Apostrophe dans Docker

[Punk'Avenue](https://punkave.com/), le créateur du CMF Apostrophe, explique [comment bâtir une image Docker pour Apostrophe](https://apostrophecms.org/docs/tutorials/howtos/docker.html). Cependant, c'est assez sommaire et pas vraiment pratique en vue de déployer régulièrement l'application en production.

M'étant inspiré de l'exemple utilisé par Punk'Avenue, j'ai ajouté l'installation de `libpng-dev` pour éviter une erreur d'upload d'images lors de l'utilisation de l'appli :

```docker
FROM node:boron-slim

RUN apt-get update -y && apt-get install -y --no-install-recommends gcc make libpng-dev

# Create app directory
RUN mkdir -p /app
WORKDIR /app

# Install node modules
COPY package*.json /app/
RUN cd /app && npm install --registry=https://registry.npmjs.org/ --only=production
 
# Install application
COPY dist /app

# Mount persistent storage
VOLUME /app/data
VOLUME /app/public/uploads

EXPOSE 3000
CMD [ "npm", "start" ]

```

## Configuration de développement et de production

Plutôt que de rester sur de la ligne de commande comme dans l'exemple d'Apostrophe, je propose des fichiers de configuration.

Pour le développement : 

```yaml
falkodev-site-db:
  container_name: falkodev-site-db
  image: mongo
  command: --smallfiles --rest
  ports:
  - 27017:27017
  volumes:
  - mongodata:/data/db

falkodev-site-dev:
  container_name: falkodev-site-dev
  build: .
  command: sh ./scripts/docker-dev.sh
  ports:
  - 3000:3000
  - 3001:3001
  links:
  - falkodev-site-db
  volumes:
  - ./:/app
  - ./data:/app/data
  - publicdata:/app/public
  - ./public/uploads:/app/public/uploads
  - /app/node_modules
  working_dir: /app
  environment:
    MONGODB: mongodb://falkodev-site-db:27017/site
    NODE_ENV: development
    NODE_APP_INSTANCE: docker
```

J'expose usuellement le port 3000 pour la production et le port 3001 pour le développement. La commande `sh ./scripts/docker-dev.sh` est un simple fichier bash qui lance une tâche npm. Chez moi, il s'agit de `npm run dev` qui démarre la version de développement de mon application.

Pour lancer l'image Docker, on lance la commande `docker-compose up` et s'il s'agit d'un autre fichier de configuration que celui par défaut (`docker-compose.yml`), on passe la commande suivante `docker-compose -f docker-compose-dev.yml up` où `docker-compose-dev.yml` est le fichier de configuration utilisé. 


## Ajout serveur Nginx

Pour cette appli, j'ai besoin d'un serveur [Nginx](https://nginx.org/en/), qui me servira de serveur de fichiers statiques (plus rapide que Node dans ce cas) et de reverse-proxy vers mon app. Pour faire simple, Nginx en face d'Apache est un peu ce qu'est Node face à PHP : asynchrone pour des connexions non-bloquantes, moins consommateur de RAM et capable d'absorber plus de connexions simultanées. 

Voici la configuration Nginx dont je me sers : 

```nginx
worker_processes  1;

error_log  /etc/nginx/logs/error.log;
error_log  /etc/nginx/logs/error.log  notice;
error_log  /etc/nginx/logs/error.log  info;

pid        /etc/nginx/logs/nginx.pid;

events {
    worker_connections  1024;
}

 http {
     include       mime.types;
     default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                     '$status $body_bytes_sent "$http_referer" '
                     '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /etc/nginx/logs/access.log  main;

    sendfile        on;
    sendfile_max_chunk 5m;
    tcp_nopush     on;
    keepalive_timeout  65;

    gzip  on;
    gzip_types      text/plain text/css application/javascript application/xml application/json application/octet-stream image/png image/svg+xml;
    gzip_proxied    no-cache no-store private expired auth;
    gzip_min_length 1000;

    proxy_set_header   Host $host;
    proxy_set_header   X-Real-IP $remote_addr;
    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Host $server_name;
    proxy_read_timeout 5m;

    server {
        listen 80;
        server_name .tarlao.fr;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2 default_server;
        listen [::]:443 ssl http2 default_server;
        ssl_certificate /etc/letsencrypt/live/tarlao.fr/fullchain.pem;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
        ssl_ecdh_curve secp384r1;
        ssl_session_cache shared:SSL:10m;
        ssl_session_tickets off;
        ssl_stapling on;
        ssl_stapling_verify on;
        ssl_dhparam /etc/nginx/certs/dhparam.pem;
        resolver 8.8.8.8 8.8.4.4 valid=300s;
        resolver_timeout 5s;
        add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";
        add_header Cache-Control "public, max-age=86400";
        add_header Pragma public;
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        expires 5d;

        root /usr/share/nginx/html;
        sendfile        on;
        sendfile_max_chunk 5m;
        tcp_nopush     on;
        keepalive_timeout  65;

        location / {
            proxy_pass http://falkodev-site:3000/;
        }

        location /css/ {
            root  /usr/share/nginx/html;
        }
        location /js/ {
            root  /usr/share/nginx/html;
        }
        location /fonts/ {
            root  /usr/share/nginx/html;
        }
        location /img/ {
            root  /usr/share/nginx/html;
        }
        location /uploads/ {
            root  /usr/share/nginx/html;
        }
    }
 }

```

`falkodev-site` est le nom que j'ai donné au conteneur Docker pour l'application Apostrophe, c'est pourquoi on le retrouve ici dans cette configuration.

## Création du réseau de conteneurs

Enfin, pour la configuration de production, on assemble les différents conteneurs (base de données Mongo, serveur Node pour l'application Apostrophe, serveur web Nginx)  dans un "network" Docker (que j'ai appelé "siteperso" dans l'exemple ci-dessous), c'est-à-dire un réseau de conteneurs que Docker gèrera comme des micro-services indépendants et communiquant entre eux.

```yaml
version: '3'

services:
  falkodev-site-db:
    container_name: falkodev-site-db
    image: mongo
    command: --smallfiles
    ports:
      - 27017:27017

  falkodev-site:
    container_name: falkodev-site
    build: .
    ports:
      - 3000:3000
    depends_on:
      - falkodev-site-db
    volumes:
      - /app/node_modules
    environment:
      MONGODB: mongodb://falkodev-site-db:27017/site
      NODE_ENV: production
      NODE_APP_INSTANCE: docker

  falkodev-web:
    container_name: falkodev-web
    image: nginx:alpine
    restart: always
    volumes:
      - ./dist/public:/usr/share/nginx/html:ro
      - ./dist/nginx/site.conf:/etc/nginx/nginx.conf:ro
      - ./dist/nginx/certs:/etc/nginx/certs:ro
      - ./dist/nginx/logs:/etc/nginx/logs:rw
      - ./dist/letsencrypt:/etc/letsencrypt:ro
    depends_on:
      - falkodev-site
    ports:
      - 80:80
      - 443:443

networks:
  default:
    external:
      name: siteperso
```

Sur le serveur de production, ne reste plus qu'à lancer la commande `docker-compose up -d` pour construire et exécuter l'image Docker en arrière plan. Cela crée le réseau de conteneurs. Nginx écoute sur les ports 80 (HTTP) et 443 (HTTPS) pour accepter les connexions entrantes et les traiter.

Il y a aussi quelques étapes supplémentaires à traiter avant un déploiement complet. Ceci sera détaillé [au prochain épisode.](/dockeriser-et-deployer-une-app-node-2-2)

