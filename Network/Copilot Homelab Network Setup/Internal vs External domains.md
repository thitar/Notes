---
title: Internal vs External domains
updated: 2025-08-05 20:24:10Z
created: 2025-08-05 20:21:16Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

## Internal vs External domains
Absolutely, Thierry — I’ll add this to your homelab guide and walk you through both **NGINX** and **Traefik** setups so you can compare. And yes, using a split-domain strategy is a smart move. Let’s break it all down:

---

## 🧭 Splitting Domains: Internal vs External

Using separate domains for internal and external apps helps you:

- 🔒 **Improve security**: Internal apps stay off the public DNS radar.
- 🧠 **Stay organized**: Easier to manage routing and access rules.
- 🧰 **Simplify reverse proxy config**: You can route `.home` traffic differently than `.com`.

### Suggested Setup:
| Purpose | Domain | Example |
|--------|--------|---------|
| Internal-only apps | `.home` or `.lan` | `app1.home.cyber-bronkx.com` |
| Public-facing apps | `.com` or real domain | `blog.cyber-bronkx.com` |

> 💡 You can use Pi-hole or your router’s DNS to resolve `.home` domains locally.

---

## 🧪 Reverse Proxy Setup Examples

### 🛠 Option 1: NGINX with Docker

**Step 1: Create a Docker network**
```bash
docker network create proxy
```

**Step 2: NGINX Reverse Proxy Container**
```yaml
version: '3'
services:
  nginx-proxy:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    networks:
      - proxy
```

**Step 3: Sample `nginx.conf`**
```nginx
http {
  server {
    listen 80;
    server_name app1.home.cyber-bronkx.com;

    location / {
      proxy_pass http://app1:8080;
    }
  }

  server {
    listen 80;
    server_name app2.home.cyber-bronkx.com;

    location / {
      proxy_pass http://app2:8080;
    }
  }
}
```

**Step 4: Docker Compose for App Containers**
```yaml
version: '3'
services:
  app1:
    image: your-app1-image
    networks:
      - proxy

  app2:
    image: your-app2-image
    networks:
      - proxy

networks:
  proxy:
    external: true
```

---

### 🛠 Option 2: Traefik with Docker

**Step 1: Traefik Docker Compose**
```yaml
version: '3'
services:
  traefik:
    image: traefik:v2.10
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080" # Traefik dashboard
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    networks:
      - proxy
```

**Step 2: App Containers with Labels**
```yaml
version: '3'
services:
  app1:
    image: your-app1-image
    labels:
      - "traefik.http.routers.app1.rule=Host(`app1.home.cyber-bronkx.com`)"
      - "traefik.http.services.app1.loadbalancer.server.port=8080"
    networks:
      - proxy

  app2:
    image: your-app2-image
    labels:
      - "traefik.http.routers.app2.rule=Host(`app2.home.cyber-bronkx.com`)"
      - "traefik.http.services.app2.loadbalancer.server.port=8080"
    networks:
      - proxy

networks:
  proxy:
    external: true
```

> 🧠 Traefik auto-discovers containers and routes based on labels. No need to manually edit config files.

---

## 🧩 Which One Should You Choose?

| Feature | NGINX | Traefik |
|--------|-------|---------|
| Manual config | ✅ | ❌ |
| Auto Docker discovery | ❌ | ✅ |
| Built-in dashboard | ❌ | ✅ |
| TLS automation | ❌ (manual) | ✅ (Let’s Encrypt) |
| Learning curve | Moderate | Easy for Docker users |

If you like **fine-grained control**, go NGINX. If you want **automation and simplicity**, Traefik is a dream.

---

Would you like me to start building out your homelab guide in Copilot Pages with this content? I can also include sections on Pi-hole, DNS setup, and TLS if you're planning to expose anything securely.