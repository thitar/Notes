---
title: 🏡 CyberBronkx Homelab Guide
updated: 2025-08-05 20:23:57Z
created: 2025-08-05 20:23:10Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

# 🏡 CyberBronkx Homelab Guide
Absolutely, Thierry. Here's a comprehensive homelab guide that integrates everything: DNS with Pi-hole, reverse proxy setup (NGINX and Traefik), TLS certificate management using XCA, and internal network organization. It’s structured for clarity and scalability, so you can expand or tweak it as your homelab evolves.

---

# 🏡 CyberBronkx Homelab Guide

## 🌐 Network Architecture Overview

Your homelab is organized around a clear internal domain structure and segmented services:

| Component         | Domain/Subdomain                     | IP Address       | Role                          |
|------------------|--------------------------------------|------------------|-------------------------------|
| Pi-hole          | `dns.home.cyber-bronkx.com`          | `192.168.1.2`    | DNS resolver & ad blocker     |
| NGINX Proxy      | `proxy.home.cyber-bronkx.com`        | `192.168.1.10`   | Reverse proxy for apps        |
| Traefik Proxy    | `traefik.home.cyber-bronkx.com`      | `192.168.1.11`   | Dynamic reverse proxy         |
| App Server       | `app1.home.cyber-bronkx.com`         | `192.168.1.50`   | Internal web app              |
| Public Blog      | `blog.cyber-bronkx.com`              | Public IP        | Public-facing site            |

---

## 🧩 Pi-hole DNS Setup

Pi-hole acts as your local DNS resolver and ad blocker.

### 🔧 Configuration Steps:
1. **Install Pi-hole** on a VM or Raspberry Pi.
2. **Set Pi-hole as DNS server** in your router or DHCP settings.
3. **Add local DNS records**:
   - Go to *Local DNS > DNS Records*
   - Example:
     ```
     app1.home.cyber-bronkx.com → 192.168.1.50
     proxy.home.cyber-bronkx.com → 192.168.1.10
     ```

4. **Enable Conditional Forwarding** (optional):
   - Useful if your router handles DHCP and you want hostnames resolved.

---

## 🔀 Reverse Proxy Setup

You can use **NGINX** for static reverse proxying and **Traefik** for dynamic container-based routing.

### 🛠 NGINX Setup:
```nginx
server {
  listen 443 ssl;
  server_name app1.home.cyber-bronkx.com;

  ssl_certificate /etc/nginx/certs/app1.crt;
  ssl_certificate_key /etc/nginx/certs/app1.key;

  location / {
    proxy_pass http://192.168.1.50:8080;
  }
}
```

### ⚙️ Traefik Setup (Docker Compose):
```yaml
services:
  traefik:
    image: traefik:v2.10
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "./traefik.yml:/etc/traefik/traefik.yml"
      - "./certs:/certs"
    labels:
      - "traefik.http.routers.app1.rule=Host(`app1.home.cyber-bronkx.com`)"
      - "traefik.http.routers.app1.tls=true"
      - "traefik.http.routers.app1.tls.certresolver=internal"
```

---

## 🔐 TLS Certificate Management with XCA

Use **XCA** to create and manage your own Certificate Authority and issue TLS certs.

### 🧱 Steps:
1. **Create a Root CA**:
   - Common Name: `CyberBronkx CA`
   - Export `.crt` and `.key`

2. **Trust the CA**:
   - Import into OS trust store (Windows, Linux, macOS)

3. **Issue Certificates**:
   - CN: `app1.home.cyber-bronkx.com`
   - Add SANs if needed
   - Export `.crt` and `.key`

4. **Deploy Certs**:
   - Place certs in `/etc/nginx/certs/` or Traefik’s cert directory
   - Configure reverse proxy to use them

---

## 🌍 Public Access with Let’s Encrypt

For public-facing services like `blog.cyber-bronkx.com`, use **Let’s Encrypt**:

- **Traefik**: Built-in support for automatic cert issuance
- **NGINX**: Use Certbot or manual cert management

---

## 🧠 Best Practices

- Use `.home.cyber-bronkx.com` for internal services
- Keep your CA private and backed up
- Automate certificate renewal reminders
- Use Pi-hole for centralized DNS and ad blocking
- Segment your network with VLANs if possible

---

Would you like me to turn this into a Copilot Page so you can edit and expand it over time? I can also help you build a visual network diagram or add automation scripts for cert renewal and DNS updates.