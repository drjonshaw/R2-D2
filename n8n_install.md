# Server Setup

based on this guide <https://www.youtube.com/watch?v=UHHkehy3SQk>

## Google Cloud Creation

Log into Google Cloud
<https://console.cloud.google.com/>
Create a new project and provide a billing account

Enable the "Compute Engine" service in Google Cloud

### Create a free tier VM

Keep it on one of the 3 free regions
Change "Machine Type" to E2-micro
Change "Boot Disk" settings
- Ubuntu
- Select Standard Persistant Disk
- 30Gb
Change Firewall settings and select all

Once created, connect by clicking the SSH link

### update server

<https://docs.n8n.io/hosting/installation/server-setups/docker-compose/#1-install-docker>
sudo apt update

## Install Docker

sudo apt install docker.io
sudo apt-get install ca-certificates curl gnup lsb-release
sudo mkdir -p /etc/apt/keyrings
sudo apt-get update

sudo systemctl start docker
sudo systemctl enable docker

### Install Docker Compose

- download
sudo curl -L "https://github.com/docker/compose/releases/download/v2.31.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
- make the file executable
sudo chmod +x /usr/local/bin/docker-compose
- verify it has been installed
sudo docker-compose --version

### Create docker-compose.yaml for N8N and Traefik

- use a nano editor to write the yaml file:

sudo nano docker-compose.yaml

- the following is the YAML file (this uses a SQL lite database)

```
services:
  traefik:
    image: "traefik"
    restart: always
    command:
      - "--api=true"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
      - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro

  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "127.0.0.1:5678:5678"
    labels:
      - traefik.enable=true
      - traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)
      - traefik.http.routers.n8n.tls=true
      - traefik.http.routers.n8n.entrypoints=web,websecure
      - traefik.http.routers.n8n.tls.certresolver=mytlschallenge
      - traefik.http.middlewares.n8n.headers.SSLRedirect=true
      - traefik.http.middlewares.n8n.headers.STSSeconds=315360000
      - traefik.http.middlewares.n8n.headers.browserXSSFilter=true
      - traefik.http.middlewares.n8n.headers.contentTypeNosniff=true
      - traefik.http.middlewares.n8n.headers.forceSTSHeader=true
      - traefik.http.middlewares.n8n.headers.SSLHost=${DOMAIN_NAME}
      - traefik.http.middlewares.n8n.headers.STSIncludeSubdomains=true
      - traefik.http.middlewares.n8n.headers.STSPreload=true
      - traefik.http.routers.n8n.middlewares=n8n@docker
    environment:
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  traefik_data:
    external: true
  n8n_data:
    external: true

```

- copy this into the nano editor
- then Ctrl O and hit enter
- then Ctrl X to exit

### create .env file for the variables

sudo nano .env
- use the following for the .env file

```
# The top level domain to serve from
DOMAIN_NAME=example.com

# The subdomain to serve from
SUBDOMAIN=n8n

# DOMAIN_NAME and SUBDOMAIN combined decide where n8n will be reachable from
# above example would result in: https://n8n.example.com

# Optional timezone to set which gets used by Cron-Node by default
# If not set New York time will be used
GENERIC_TIMEZONE=Europe/Berlin

# The email address to use for the SSL certificate creation
SSL_EMAIL=user@example.com
```

### create the docker volumes

- traefik
sudo docker volume create traefik_data

sudo docker volume create n8n_data

## setup DNS

- Go to GoDaddy domain name manager, select domain and select "Manage DNS"
- Add new A record
- type "n8n" and then get the external IP address of the new Google server (just the IP value)

## run the containers inc N8N

- run it in attached mode initially (so you can see the logs when it installs etc) from the SSH terminal
sudo docker-compose up

- this will download and install

Network jonandsarahshaw_default      Created                                                            2.3s 
 ✔ Container jonandsarahshaw-n8n-1      Created                                                            4.2s 
 ✔ Container jonandsarahshaw-traefik-1  Created                                                            4.0s 
Attaching to n8n-1, traefik-1
n8n-1      | No encryption key found - Auto-generated and saved to: /home/node/.n8n/config
n8n-1      | Permissions 0644 for n8n settings file /home/node/.n8n/config are too wide. This is ignored for now, but in the future n8n will attempt to change the permissions automatically. To automatically enforce correct permissions now set N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true (recommended), or turn this check off set N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=false.


- open a browser and go to https://n8n.virglai.com

- but then to exit the terminal Ctrl C will also shut down the containers

### run in detached mode

sudo docker-compose up -d

- to check containers

sudo docker-compose ps

- to stop

sudo docker-compose down

### checking logs

sudo docker-compose logs n8n



