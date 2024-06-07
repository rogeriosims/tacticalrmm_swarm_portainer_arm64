# Deploy do TacticalRMM no Portainer usando Docker Swarm em Ambiente ARM64

Este guia fornece um passo a passo para realizar o deploy do TacticalRMM no Portainer usando Docker Swarm em um ambiente ARM64.

## Passo 1: Clonar o Repositório

Clone o repositório do TacticalRMM para o seu ambiente local.

```sh
git clone https://github.com/amidaware/tacticalrmm.git

cd tacticalrmm/docker
```

## Passo 2: Copiar Dockerfiles

Copie os Dockerfiles contidos na pasta docker/containers/ para a raiz do repositório.

```sh
cp docker/containers/tactical/dockerfile ./dockerfile.tactical

cp docker/containers/tactical-frontend/dockerfile ./dockerfile.frontend

cp docker/containers/tactical-meshcentral/dockerfile ./dockerfile.meshcentral

cp docker/containers/tactical-nats/dockerfile ./dockerfile.nats

cp docker/containers/tactical-nginx/dockerfile ./dockerfile.nginx
```

## Passo 3: Alterar Arquivos Necessários

### 3.1 Alterar entrypoint.sh do Meshcentral

Edite o arquivo docker/containers/tactical-meshcentral/entrypoint.sh para substituir o IP pelo host.

```sh
nano docker/containers/tactical-meshcentral/entrypoint.sh
```
Alterar de:
NGINX_HOST_IP:=172.20.0.20

Para:
NGINX_HOST_IP:=tactical-nginx

### 3.2 Alterar Dockerfile do NATS
Edite o arquivo dockerfile.nats para usar o binário ARM64.

```sh
nano dockerfile.nats
```

Alerar de:
COPY natsapi/bin/nats-api /usr/local/bin

Para:
COPY natsapi/bin/nats-api-arm64 /usr/local/bin/nats-api/

## Passo 4: Construir as Imagens
Construa as imagens Docker para cada componente do TacticalRMM.

```sh

docker build -t tactical:latest -f dockerfile.tactical .

docker build -t tactical.frontend:latest -f dockerfile.frontend .

docker build -t tactical.meshcentral:latest -f dockerfile.meshcentral . 

docker build -t tactical.nats:latest -f dockerfile.nats .

docker build -t tactical.nginx:latest -f dockerfile.nginx .

```

## Passo 5: Configurar o Ambiente

Copiar e Editar o Arquivo .env

Edite o arquivo .env para definir as variáveis de ambiente como POSTGRES_USER, POSTGRES_PASS, APP_HOST, API_HOST, MESH_USER, MESH_HOST, TRMM_USER, TRMM_PASS, etc.

### 5.1 Criar certificado curinga

Configurar o dominio para o ip do serviço.

```sh
sudo certbot certonly --manual -d *.tecnologiacorporativa.com.br --agree-tos --no-bootstrap --preferred-challenges dns
```

Incluia uma entrada txt no dns com o valor retornado pelo certbot, e clique em ok.

```sh
sudo echo "CERT_PUB_KEY=$(sudo base64 -w 0 /etc/letsencrypt/live/tecnologiacorporativa.com.br/fullchain.pem)" >> .env

sudo echo "CERT_PRIV_KEY=$(sudo base64 -w 0 /etc/letsencrypt/live/tecnologiacorporativa.com.br/privkey.pem)" >> .env
```

## Passo 6: Deploy com Docker Stack no Portainer

Crie uma stack no portainer e cole a stack abaixo. Não esqueça de importar o arquivo .env ou configurar as váriaves diretamente no portainer.


```yaml

version: "3.7"

# networks
networks:
  proxy:
    driver: overlay
    ipam:
      driver: default
      config:
        - subnet: 172.20.0.0/24
  api-db:
    driver: overlay
  redis:
    driver: overlay
  mesh-db:
    driver: overlay

# docker managed persistent volumes
volumes:
  tactical_data:
    external: true
  postgres_data:
    external: true
  mongo_data:
    external: true
  mesh_data:
    external: true
  redis_data:
    external: true


services:
  # postgres database for api service
  tactical-postgres:
    image: postgres:13-alpine
    restart: always
    environment:
      POSTGRES_DB: tacticalrmm
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASS}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - api-db

  # redis container for celery tasks
  tactical-redis:
    image: redis:6.0-alpine
    user: 1000:1000
    command: redis-server
    restart: always
    volumes:
      - redis_data:/data
    networks:
      - redis

  # used to initialize the docker environment
  tactical-init:
    image: tactical:${VERSION}
    restart: on-failure
    command: ["tactical-init"]
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASS: ${POSTGRES_PASS}
      APP_HOST: ${APP_HOST}
      API_HOST: ${API_HOST}
      MESH_USER: ${MESH_USER}
      MESH_HOST: ${MESH_HOST}
      TRMM_USER: ${TRMM_USER}
      TRMM_PASS: ${TRMM_PASS}
    depends_on:
      - tactical-postgres
      - tactical-meshcentral
      - tactical-redis
    networks:
      - api-db
      - proxy
      - redis
    volumes:
      - tactical_data:/opt/tactical
      - mesh_data:/meshcentral-data
      - mongo_data:/mongo/data/db
      - redis_data:/redis/data

  # nats
  tactical-nats:
    image: tactical-nats:${VERSION}
    user: 1000:1000
    restart: always
    environment:
      API_HOST: ${API_HOST}
    volumes:
      - tactical_data:/opt/tactical
    networks:
      api-db: null
      proxy:
        aliases:
          - ${API_HOST}

  # meshcentral container
  tactical-meshcentral:
    image: tactical-mesh:${VERSION}
    user: 1000:1000
    restart: always
    environment:
      MESH_HOST: ${MESH_HOST}
      MESH_USER: ${MESH_USER}
      MESH_PASS: ${MESH_PASS}
      MONGODB_USER: ${MONGODB_USER}
      MONGODB_PASSWORD: ${MONGODB_PASSWORD}
      MESH_PERSISTENT_CONFIG: ${MESH_PERSISTENT_CONFIG}
    networks:
      proxy:
        aliases:
          - ${MESH_HOST}
      mesh-db: null
    volumes:
      - tactical_data:/opt/tactical
      - mesh_data:/home/node/app/meshcentral-data
    depends_on:
      - tactical-mongodb

  # mongodb container for meshcentral
  tactical-mongodb:
    image: mongo:4.4
    restart: always
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGODB_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGODB_PASSWORD}
      MONGO_INITDB_DATABASE: meshcentral
      PUID: 1000
      PGID: 1000  
    networks:
      - mesh-db
    volumes:
      - mongo_data:/data/db

    
  # container that hosts vue frontend
  tactical-frontend:
    image: tactical-frontend:${VERSION}
    user: 1000:1000
    restart: always
    networks:
      - proxy
    volumes:
      - tactical_data:/opt/tactical
    environment:
      API_HOST: ${API_HOST}

  # container for django backend
  tactical-backend:
    image: tactical:${VERSION}
    user: 1000:1000
    command: ["tactical-backend"]
    restart: always
    networks:
      - proxy
      - api-db
      - redis
    volumes:
      - tactical_data:/opt/tactical
    depends_on:
      - tactical-postgres

  # container for django websockets connections
  tactical-websockets:
    image: tactical:${VERSION}
    user: 1000:1000
    command: ["tactical-websockets"]
    restart: always
    networks:
      - proxy
      - api-db
      - redis
    volumes:
      - tactical_data:/opt/tactical
    depends_on:
      - tactical-postgres
      - tactical-backend

  # container for tactical reverse proxy
  tactical-nginx:
    image: tactical-nginx:${VERSION}
    user: 1000:1000
    restart: always
    environment:
      APP_HOST: ${APP_HOST}
      API_HOST: ${API_HOST}
      MESH_HOST: ${MESH_HOST}
      CERT_PUB_KEY: ${CERT_PUB_KEY}
      CERT_PRIV_KEY: ${CERT_PRIV_KEY}
    networks:
      - proxy
    ports:
      - "${TRMM_HTTP_PORT-80}:8080"
      - "${TRMM_HTTPS_PORT-443}:4443"
    volumes:
      - tactical_data:/opt/tactical

  # container for celery worker service
  tactical-celery:
    image: tactical:${VERSION}
    user: 1000:1000
    command: ["tactical-celery"]
    restart: always
    networks:
      - redis
      - proxy
      - api-db
    volumes:
      - tactical_data:/opt/tactical
    depends_on:
      - tactical-postgres
      - tactical-redis

  # container for celery beat service
  tactical-celerybeat:
    image: tactical:${VERSION}
    user: 1000:1000
    command: ["tactical-celerybeat"]
    restart: always
    networks:
      - proxy
      - redis
      - api-db
    volumes:
      - tactical_data:/opt/tactical
    depends_on:
      - tactical-postgres
      - tactical-redis
```
