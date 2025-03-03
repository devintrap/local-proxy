# 🔄 Local Proxy

Ingress-like solution for local development with automatic SSL certificates via Cloudflare DNS challenge. Perfect for developers who want Kubernetes-style service routing without the complexity of a full K8s setup.

### Key Features
- 🔐 Automatic SSL certificates via Let's Encrypt
- 🌐 Cloudflare DNS integration
- 🚦 Ingress-like traffic routing
- 🔄 Automatic HTTPS redirection
- 📊 Web dashboard for monitoring

## 📑 Table of Contents
- [⚡ Requirements](#requirements)
- [🛠️ Setup](#setup)
- [📝 Usage](#usage)
- [🔑 Access](#access)

## ⚡ Requirements

- Docker/Podman
- Docker Compose / Podman Compose
- Cloudflare DNS account

## 🛠️ Setup

1. Copy environment file:
```bash
cp .env.example .env
```

2. Configure your `.env` file based on [`.env.example`](.env.example):

3. Configure container socket mounting in `traefik/compose.yml`:

For Docker:
```yaml
volumes:
  - ./traefik/data/letsencrypt:/letsencrypt:Z
  - /var/run/docker.sock:/var/run/docker.sock:z
```
For Podman:
```yaml
volumes:
  - ./traefik/data/letsencrypt:/letsencrypt:Z
  - /run/user/1000/podman/podman.sock:/var/run/docker.sock:z
```

4. Start the proxy using the main `local-proxy/compose.yml`:
```bash
cd local-proxy
docker-compose up -d   # for Docker
# or
podman-compose up -d  # for Podman
```

## 📝 Usage

1. Create your service directory and its own `compose.yml`:

```bash
mkdir myapp
touch myapp/compose.yml
```

2. Add your service configuration in `myapp/compose.yml`:
```yaml
// filepath: /home/pawel/Projects/gitlab/devintr.app/local-proxy/myapp/compose.yml
services:
  myapp:
    image: nginx:${APP_VERSION}
    restart: always
    networks:
      internal:
        ipv4_address: ${APP_IP}
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`${APP_SUBDOMAIN}.${DOMAIN_NAME}`)"
      - "traefik.http.routers.myapp.tls=true"
      - "traefik.http.routers.myapp.entrypoints=web,websecure"
      - "traefik.http.routers.myapp.tls.certresolver=letsencrypt"
      # Security headers
      - "traefik.http.middlewares.myapp.headers.SSLRedirect=true"
      - "traefik.http.middlewares.myapp.headers.STSSeconds=315360000"
      - "traefik.http.middlewares.myapp.headers.browserXSSFilter=true"
      - "traefik.http.middlewares.myapp.headers.contentTypeNosniff=true"
      - "traefik.http.middlewares.myapp.headers.forceSTSHeader=true"
      - "traefik.http.middlewares.myapp.headers.SSLHost=${DOMAIN_NAME}"
      - "traefik.http.middlewares.myapp.headers.STSIncludeSubdomains=true"
      - "traefik.http.middlewares.myapp.headers.STSPreload=true"
      - "traefik.http.routers.myapp.middlewares=myapp@docker"
```

3. Add your service to the main `compose.yml`:
```yaml
version: '3'
services:
  traefik:
    extends:
      file: ./traefik/compose.yml
      service: traefik

  myapp:
    extends:
      file: /path-to-app/compose.yml
      service: myapp

networks:
  internal:
    ipam:
      config:
        - subnet: ${SUBNET}
          gateway: ${GATEWAY}
```

4. Add your app configuration to `.env`:
```env
# Example App configs
APP_VERSION=latest
APP_IP=10.0.0.3
APP_SUBDOMAIN=myapp
```

> **Note**: Always use the main `compose.yml` in the root directory to manage your services. Each service should have its own directory with its compose file, which is then referenced in the main compose file using `extends`.

## 🔑 Access

- 🔐HTTPS: `https://${DOMAIN_NAME}`
- 🔓️HTTP: `http://${DOMAIN_NAME}` (redirects to HTTPS)
- 📊Dashboard: `https://traefik.${DOMAIN_NAME}` or `https://localhost:8080`