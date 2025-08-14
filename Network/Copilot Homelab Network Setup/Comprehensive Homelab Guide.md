---
title: Comprehensive Homelab Guide
updated: 2025-08-05 20:16:50Z
created: 2025-08-05 20:16:47Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

# Comprehensive Homelab Guide

This guide provides a **step-by-step**, integrated walkthrough for building and documenting a homelab that includes local DNS with Pi-hole, reverse proxy setups using NGINX and Traefik, TLS certificate management via XCA, internal network segmentation with VLANs, and secure remote access with ZeroTier. The goal is a **clear**, **scalable**, and **secure** environment suitable for self-hosted services, experimentation, and continuous learning.  

---

## Table of Contents

- Overview and Objectives  
- Internal Network Organization  
  - VLAN Segmentation and Purpose  
  - Hardware Requirements and Topology  
- Pi-hole DNS Setup  
  - Installing Pi-hole  
  - Integrating Unbound for Recursive DNS  
  - Troubleshooting Common DNS Issues  
- NGINX Reverse Proxy Configuration  
  - Installing and Enabling NGINX  
  - Server Blocks and Snippets  
  - SSL Termination and HTTP→HTTPS Redirects  
  - Common Pitfalls and Troubleshooting  
- Traefik Reverse Proxy Configuration  
  - Deploying Traefik via Docker Compose  
  - Static vs. Dynamic Configuration  
  - Container Labels and Middleware  
  - ACME DNS Challenge and Certificate Resolvers  
  - Security Best Practices for Traefik  
- TLS Certificate Management with XCA  
  - Setting Up the XCA Database  
  - Creating Root, Intermediate, and End-Entity Certificates  
  - Exporting Certificates and Keys  
  - Integrating XCA Certificates with NGINX and Traefik  
- Secure Remote Access with ZeroTier  
  - Why ZeroTier for Homelab VPN  
  - Installing and Joining a ZeroTier Network  
  - Authorizing and Managing Members  
  - Advanced Routing and DNS Integration  
- Markdown Documentation Best Practices  
  - Heading Structure and Paragraphs  
  - Code Blocks, Tables, and Emphasis  
  - Extended Syntax Tips  
- References  

---

## Overview and Objectives

Building a home lab often involves combining multiple components—networking, DNS, reverse proxies, VPNs, and certificate management—into a **cohesive**, **secure**, and **maintainable** setup. Our objectives in this guide are:

1. Establish **local DNS** services with **Pi-hole** and **Unbound** for ad blocking, privacy, and performance.  
2. Configure **NGINX** and **Traefik** as reverse proxies to handle HTTP(S) traffic, load balancing, and TLS termination.  
3. Manage a **private PKI** with **XCA**, creating root, intermediate, and end-entity certificates for internal services.  
4. Organize the internal network with **VLAN segmentation** for isolation, security, and traffic control.  
5. Implement **ZeroTier** to provide secure, peer-to-peer VPN connectivity across NAT boundaries.  
6. Document configurations using **Markdown**, following best practices for clarity and reusability.  

With these pieces in place, you’ll have a fully segmented, secure, and remotely accessible homelab environment.

---

## Internal Network Organization

### VLAN Segmentation and Purpose

Segmenting your homelab network into VLANs (Virtual LANs) allows you to isolate and secure traffic based on device function and trust level. Each VLAN is assigned a unique subnet and purpose. This table summarizes a recommended VLAN scheme:

| VLAN ID | Name             | Subnet           | Purpose                                                | Trust Level |
| ------- | ---------------- | ---------------- | ------------------------------------------------------ | ----------- |
| 1       | Management       | 192.168.1.0/24   | Host management (Routers, Switches, Controller UIs)    | High        |
| 10      | Infrastructure   | 192.168.10.0/24  | Core services: DNS, DHCP, NTP, Monitoring              | High        |
| 20      | Internal Apps    | 192.168.20.0/24  | Internal services: GitLab, Home Assistant, Syncthing   | Medium      |
| 30      | Edge Services    | 192.168.30.0/24  | Public / DMZ services: Reverse Proxy, Web, T-Pot       | Medium      |
| 40      | Servers & VMs    | 192.168.40.0/24  | Virtualization host networks (Proxmox, ESXi vSAN)      | High        |
| 50      | IoT              | 192.168.50.0/24  | IoT devices and smart home                             | Low         |
| 60      | Guests           | 192.168.60.0/24  | Guest Wi-Fi, isolated from corporate/personal devices  | Low         |

Every VLAN is blocked by default from other VLANs using Layer 3 firewall rules, except as expressly permitted. For example, VLAN 20 (Internal Apps) may reach VLAN 10 (Infrastructure) for DNS and monitoring, while VLAN 50 (IoT) can only access the Internet via the router.

### Hardware Requirements and Topology

A resilient homelab network needs VLAN-aware hardware and proper cabling. Here’s a typical hardware bill of materials:

| Device                           | Model / Role                | Ports           | Notes                                                     |
| -------------------------------- | --------------------------- | --------------- | --------------------------------------------------------- |
| Edge Router / Firewall           | pfSense / UDM-Pro           | 8 LAN, 2 WAN    | Layer 3 routing, DHCP, DNS, Firewall, VLAN segmentation  |
| Managed Ethernet Switch (Leaf)   | UniFi USW-Nano — 8×1GbE     | 1×10GbE (UPLINK) | Access layer, connects servers and internal devices       |
| Managed Ethernet Switch (Spine)  | UniFi USW-Pro 24×2.5GbE     | 4×10GbE         | Aggregates leaf switches, uplink to router and NAS        |
| Wireless Access Point            | UniFi U6-Mesh               | dual-band 4×4   | Supports multiple SSIDs mapped to VLANs                   |
| NAS (File / Backup)              | Synology RS822+             | 4×HDD, 2×SSD    | NFS/SMB, VM datastore, snapshots, Active Backup           |
| Compute Nodes                     | Proxmox / ESXi Hosts        | 2×10GbE each    | Virtualization with vSAN, VM, LXC                         |
| IP-based KVM                      | PiKVM on Raspberry Pi       | 1×USB, 1×HDMI   | Remote console access to hypervisors                      |

**Topology**: Leaf switches connect directly to the router (Layer 3 device). Spine switches aggregate leaf uplinks. The NAS connects to the spine for high-speed storage. Access switches connect desktops, laptops, and IoT devices. Wireless APs map SSIDs to guest or internal VLANs.

---

## Pi-hole DNS Setup

### Installing Pi-hole

Pi-hole blocks ads, provides local DNS, and can integrate with Unbound to become your **private recursive resolver**. Install on a Raspberry Pi or VM:

1. Update system:  
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```  
2. Install Pi-hole via automated script or Docker:  
   ```bash
   curl -sSL https://install.pi-hole.net | bash
   ```  
   – Choose desired blocklists, disable IPv6 if unsupported, and set admin UI password.  
3. Point your router’s DHCP DNS to Pi-hole’s IP (e.g., 192.168.10.2).

Pi-hole’s web UI on `http://192.168.10.2/admin` shows query logs, blocklist status, and ad metrics.

### Integrating Unbound for Recursive DNS

Using **Unbound** behind Pi-hole provides **DNS privacy** and **independence** from upstream providers.  

1. Install Unbound:  
   ```bash
   sudo apt install unbound -y
   ```  
2. Configure Unbound with Pi-hole’s `pi-hole.conf`:  
   ```bash
   sudo tee /etc/unbound/unbound.conf.d/pi-hole.conf << 'EOF'
   server:
     interface: 127.0.0.1
     port: 5335
     do-ip4: yes
     do-udp: yes
     do-tcp: yes
     root-hints: "/var/lib/unbound/root.hints"
     harden-glue: yes
     prefetch: yes
     private-address: 192.168.0.0/16
   EOF
   ```  
3. Download root hints and enable auto-update:  
   ```bash
   sudo curl -s https://www.internic.net/domain/named.root -o /var/lib/unbound/root.hints
   ```  
4. Point Pi-hole’s DNS to Unbound (`Custom 1: 127.0.0.1#5335`).

Restart services:  
```bash
sudo systemctl restart unbound
sudo pihole restartdns
```  

This yields **encrypted**, **recursive** lookups with DNSSEC validation.

### Troubleshooting Common DNS Issues

Common Pi-hole problems include DNS overrides by routers and IPv6 bypassing Pi-hole. Refer to the Pi-hole Troubleshooting Guide for solutions:

- Ensure devices use Pi-hole’s IP as primary DNS (`nslookup pi.hole`).  
- Block IPv6 if unsupported or configure Pi-hole for AAAA queries (`dig AAAA example.com @127.0.0.1 -p 5335`).  
- Prevent clients bypassing Pi-hole with DoH/DoT by firewall rules blocking port 53 to upstream servers.  
- Check Pi-hole’s `pihole -d` debug log for errors.

---

## NGINX Reverse Proxy Configuration

### Installing and Enabling NGINX

NGINX can proxy multiple internal services via **server blocks** (virtual hosts) and provides SSL termination. On Debian/Ubuntu:  
```bash
sudo apt install nginx -y  
sudo systemctl enable --now nginx  
```  
Verify status:  
```bash
sudo systemctl status nginx  
```  
NGINX listens on port 80 by default; adjust with `listen` directives in `/etc/nginx/nginx.conf`.

### Server Blocks and Snippets

Create re-usable snippets for proxy headers and SSL params in `/etc/nginx/snippets/`:

**`/etc/nginx/snippets/proxy-defaults.conf`**  
```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_redirect off;
```

**`/etc/nginx/snippets/ssl-params.conf`**  
```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:..."
ssl_dhparam /etc/ssl/certs/dhparam.pem;
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
```

Define a site in `/etc/nginx/sites-available/example.com`:

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl http2;
    server_name example.com;
    include snippets/ssl-params.conf;
    include snippets/proxy-defaults.conf;
    ssl_certificate /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;
    location / {
        proxy_pass http://192.168.20.10:8080;
    }
}
```

Enable and test:

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```  
Clients visiting `https://example.com` will be proxied securely.

### Common Pitfalls and Troubleshooting

- 502 Bad Gateway often results from backend failures or misconfigured `proxy_pass` URL or missing availability of upstream services.  
- 504 Gateway Timeout results from slow backends; tune `proxy_read_timeout`, `proxy_connect_timeout`, and `send_timeout` appropriately.  
- Ensure `proxy_http_version 1.1` and header updates for WebSockets (`Upgrade`, `Connection`) in proxy-defaults.conf.  
- Mismatched SSL protocols/ciphers between NGINX and backend leads to `502 Bad Gateway - sslv3 alert handshake failure` errors.

---

## Traefik Reverse Proxy Configuration

### Deploying Traefik via Docker Compose

Traefik v3 simplifies dynamic configuration for containerized environments. Sample `docker-compose.yml`:

```yaml
version: "3.8"
services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data/traefik.yml:/traefik.yml:ro
      - ./data/acme.json:/acme.json:rw
      - ./data/configs:/configs:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.rule=Host(`traefik.lab.local`)"
      - "traefik.http.routers.traefik.entrypoints=https"
      - "traefik.http.routers.traefik.middlewares=auth@file"
      - "traefik.http.routers.traefik.tls=true"
      - "traefik.http.routers.traefik.tls.certresolver=letsencrypt"
    networks:
      - proxy
networks:
  proxy:
    external: true
```

**`data/traefik.yml`**:

```yaml
api:
  dashboard: true
entryPoints:
  http:
    address: ":80"
  https:
    address: ":443"
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@lab.local
      storage: acme.json
      tlsChallenge: {}
```

Create `acme.json`, set permissions, and launch:

```bash
touch data/acme.json
chmod 600 data/acme.json
docker compose up -d
```

Traefik dashboard at `https://traefik.lab.local/dashboard/`.

### Container Labels and Middleware

Example for Traefik to proxy a simple service:

```yaml
  whoami:
    image: traefik/whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.entrypoints=http,https"
      - "traefik.http.routers.whoami.rule=Host(`whoami.lab.local`)"
      - "traefik.http.routers.whoami.middlewares=https-redirect@file"
      - "traefik.http.services.whoami.loadbalancer.server.port=80"
```

**`configs/middleware.yml`**:

```yaml
http:
  middlewares:
    https-redirect:
      redirectScheme:
        scheme: https
        permanent: true
    auth:
      basicAuth:
        users:
          - "admin:$2y$05$gQIjp..."  
```

### ACME DNS Challenge and Certificate Resolvers

Use the DNS challenge to request wildcard certificates:

```yaml
certificatesResolvers:
  dns:
    acme:
      email: admin@lab.local
      storage: acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
```

Enable environment variables for Cloudflare API token and mount volumes accordingly.

### Security Best Practices for Traefik

- Mount Docker socket read-only and consider a socket proxy for limiting API scope.  
- Run container as `read_only: true` and drop unnecessary capabilities (`no-new-privileges`).  
- Isolate middleware definitions in files and reference via `@file` to reduce coupling.  

---

## TLS Certificate Management with XCA

### Setting Up the XCA Database

XCA provides a **graphical interface** for creating and managing X.509 certificates, keys, and CSRs. Install XCA and create a new database:

1. Download and install XCA from http://hohnstaedt.de/xca.  
2. Launch XCA, **File → New Database**, choose `homelab.xdb`, set a password for the encrypted key store.

### Creating Root and Intermediate CAs

In XCA:

1. **Certificates → New Certificate → Create a CA certificate**.  
2. Under **Subject**, fill out organization details. Generate a new RSA key (2048 bits).  
3. Under **Extensions → Type: Certification Authority**, set key usage to allow certificate signing. Click **OK**.  
4. For Intermediate CA, highlight Root CA, click **New Certificate**, repeat process, selecting **CA** type.  

### Creating Server and Client Certificates

1. **New Certificate** under Intermediate CA to create end-entity certs.  
2. Under **Extensions → Type: End Entity**, set key usage (Digital Signature, Key Encipherment).  
3. Generate server key, fill in CN as `*.lab.local`, use **SubjectAltName** template for multiple domain names.  

### Exporting Certificates and Keys

1. **Certificates tab → Export → PEM** for public cert (`labCA.crt`).  
2. For private server certs, **Export → PKCS#12** (`webserver.p12`), protect with a passphrase.  
3. Export client certs similarly for PiKVM and internal services.  

### Integrating XCA Certificates with NGINX and Traefik

**NGINX**:
```nginx
ssl_certificate     /etc/ssl/certs/webserver.chained.crt;
ssl_certificate_key /etc/ssl/private/webserver.key;
```
Use the concatenated `server + intermediate` chain file as `webserver.chained.crt`.

**Traefik (dynamic)**:
```yaml
tls:
  options:
    default:
      minVersion: VersionTLS12
      cipherSuites:
        - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
```
Reference server certs via `tls.certificates` in dynamic config or leverage ACME resolvers as shown earlier.

---

## Secure Remote Access with ZeroTier

### Why ZeroTier for Homelab VPN

ZeroTier builds a **software-defined** virtual Ethernet network that spans NAT boundaries, ideal for remote device connectivity behind CGNAT or strict firewalls. It requires zero port forwarding, uses **end-to-end encryption**, and automatically handles NAT traversal via its global root servers.

### Installing and Joining a ZeroTier Network

**Install**:
```bash
curl -s https://install.zerotier.com | sudo bash
```
(Or use distro-specific packages).

**Create Network**:
1. Sign up at my.zerotier.com → Create a new private network.  
2. Copy the 16-digit Network ID.

**Join Network**:
```bash
sudo zerotier-cli join <NETWORK_ID>
```
**Authorize** on the ZeroTier Central dashboard by toggling the `Auth` checkbox for the new node.

### Authorizing and Managing Members

Use the ZeroTier API for automation:
```bash
curl -H "Authorization: Bearer $API_TOKEN" \
  -X POST -d '{"config":{"authorized":true}}' \
  https://my.zerotier.com/api/network/$NET_ID/member/$NODE_ID
```
Hide or ban nodes by setting `"hidden": true` in the same endpoint.

### Advanced Routing and DNS Integration

- **Managed Routes**: Expose your homelab LAN CIDR via a gateway node.  
- **DNS**: Run `dnsmasq` on a ZeroTier member and auto-update `/etc/hosts` via scripts from the ZeroTier API, ensuring `.zt` hostnames resolve internally.  

---

## Markdown Documentation Best Practices

### Heading Structure and Paragraphs

- Use a single `#` for the document title, `##` for main sections, and `###` for subsections.  
- Keep paragraphs **short** (3–5 sentences, ≤150 words).  
- Ensure a **blank line** before and after each paragraph.  

### Code Blocks, Tables, and Emphasis

- Use fenced code blocks (```) with language labels for syntax highlighting.  
- Insert horizontal rules (`---`) to separate major sections.  
- Create tables with pipes (`|`) and hyphens (`-`), aligning columns with colons (`:`).  

```markdown
| VLAN  | Purpose           | Subnet          |
|:-----:|:-----------------:|----------------:|
| 10    | Infrastructure    | 192.168.10.0/24 |
| 20    | Internal Services | 192.168.20.0/24 |
```

### Extended Syntax Tips

- Use **bold** (`**text**`) for key terms, _italics_ (`*text*`) for emphasis.  
- Add footnotes for clarifications (`[^1]`) and define them at the end.  
- Use task lists (`- [ ]`, `- [x]`) for TODOs and checklists.  
- Avoid inline HTML except for advanced table or code scenarios when necessary.

---

## References

This guide synthesizes insights and configurations from a wide array of reputable sources:

1. The Objective Dad. How I Built Robust Local DNS Services for My Homelab Cluster  
2. Akash Rajpurohit. Nginx — The reverse proxy in my Homelab  
3. Adam. Practical Configuration of Traefik as a Reverse Proxy For Docker - Updated for 2023 and Beyond  
4. Hohnstaedt. X Certificate and Key Management – Official XCA Tutorial  
5. DoingStuff.dev. Homelab Adventure - Part 3: Internal Network (ZeroTier)  
6. TimInTech. Pi-hole v6 – Troubleshooting Guide（GitHub）  
7. DjangoCas.dev. NGINX Reverse Proxy Configuration and Troubleshooting  
8. infinitejs. Overcoming Common Traefik Setup Issues for Continuous Delivery  
9. habitats-tech. Traefik 3.x comprehensive setup guide for LXC workloads  
10. Virtualization Howto. Best Home Lab Networking Architecture in 2025  
11. Pi-hole.net. Pi-hole as All-Around DNS Solution (Unbound)  
12. Pi My Life Up. Installing Unbound for Pi-Hole on the Raspberry Pi  
13. WunderTech. Use Unbound to Enhance the Privacy of Pi-Hole on a Raspberry Pi  
14. Pi-hole.net. OpenVPN Dynamic DNS – Pi-hole documentation  
15. GitHub Issue. Using Pi-hole’s DNS for local name resolution (Pi-hole #2443)  
16. Daniel Rampelt. Complete Guide to Setting up Pi-hole on Raspberry Pi with IPv6 Support on Docker  
17. djangocas.dev. More Nginx reverse proxy tips and common pitfalls  
18. DevOps Daily. Using Nginx-Proxy in Your Homelab: A Comprehensive Guide  
19. Homelab.blog. Nginx: Reverse Proxy  
20. NGINX Official Documentation. SSL Termination and HTTPS configuration  
21. RedSwitches. 6 Ways to Fix 502 Bad Gateway in NGINX  
22. Sling Academy. NGINX Error: 502 Bad Gateway – Causes and Solutions  
23. Stackify. Error 502 Bad Gateway in NGINX: What It Is and How to Fix It  
24. Sling Academy. NGINX Error: 504 Gateway Timeout – Causes and Solutions  
25. Traefik Documentation. Traefik Docker Compose Basic Example  
26. spad.uk. Practical Configuration of Traefik v3 for Docker  
27. Traefik Official Docs. Traefik 3.x basic entryPoints, certificate resolvers  
28. gamingrobot. Salt Formula for ZeroTier (GitHub)  
29. VirtualizationHowto. Best network architectures for 2025 – VLANs & segmentation  
30. SamKneppel. 2025 Homelab Overview – real-world segmentation and services  
31. LucasDuete.dev. My Homelab Diagram in 2025 – VLAN, VPN, and apps summary  
32. Liore.com. VLANs for the Homelab: A beginner’s guide to segmenting networks  
33. DarkCloud. ZeroTier in my homelab – peer-to-peer VPN evolution  
34. YouTube. ZeroTier: FREE and EASY Remote Access to Your Homelab  
35. Markdown Guide. Extended Syntax and Tables best practices in Markdown  
36. HTMLMarkdown.com. Markdown table tutorial and advanced tips  
37. MarcoEidinger Gist. Master GitHub Markdown tables with code blocks  
38. Google Developers. Markdown primer and best practices for docs  

---

This guide integrates all critical aspects of a professional-grade homelab: local DNS, reverse proxies, internal PKI, VLAN-based segmentation, and modern P2P VPN. Follow each section closely, adapt configurations to your environment, and continue iterating to match your evolving requirements. Good luck on your homelab journey!