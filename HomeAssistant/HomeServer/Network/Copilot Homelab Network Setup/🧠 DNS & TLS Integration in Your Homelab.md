---
title: 🧠 DNS & TLS Integration in Your Homelab
updated: 2025-08-05 20:24:07Z
created: 2025-08-05 20:22:28Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---


# 🧠 DNS & TLS Integration in Your Homelab
Perfect, Thierry — I’ll integrate everything into your homelab guide: Pi-hole DNS, reverse proxy setup (NGINX and Traefik), domain strategy, and TLS certificate management using **XCA**. Here's the next section of your guide, focused on DNS and TLS integration:

---

# 🧠 DNS & TLS Integration in Your Homelab

## 🧩 Pi-hole as Local DNS Resolver

Pi-hole can act as your **authoritative DNS** for internal domains like `.home` or `.lan`.

### ✅ Benefits:
- Blocks ads and trackers
- Resolves internal hostnames
- Centralizes DNS management

### 🔧 Setup Steps:
1. **Enable Conditional Forwarding** (optional):
   - Go to *Settings > DNS > Conditional Forwarding*
   - Point Pi-hole to your router if it handles DHCP

2. **Add Local DNS Records**:
   - Go to *Local DNS > DNS Records*
   - Add entries like:
     ```
     app1.home.cyber-bronkx.com → 192.168.1.50
     app2.home.cyber-bronkx.com → 192.168.1.50
     ```

3. **Use Pi-hole as Primary DNS**:
   - Set Pi-hole’s IP as the DNS server in your router or DHCP settings
   - All devices will resolve internal domains via Pi-hole

---

## 🔐 TLS Certificates with XCA

Using **XCA** gives you full control over certificate creation and trust management — ideal for internal services.

### 🛠 Steps to Create a Local CA and Issue Certs

1. **Create a Certificate Authority (CA)**:
   - In XCA, go to *Certificates > New Certificate*
   - Choose “CA” template
   - Set common name like `CyberBronkx CA`
   - Save and export the CA certificate

2. **Trust the CA on Devices**:
   - Import the CA cert into:
     - Windows: *Trusted Root Certification Authorities*
     - Linux: `/usr/local/share/ca-certificates/`
     - macOS: *Keychain Access > System*

3. **Issue Certificates for Internal Apps**:
   - Create a new certificate signed by your CA
   - Set CN to `app1.home.cyber-bronkx.com`
   - Add SANs (Subject Alternative Names) if needed
   - Export as `.crt` and `.key`

4. **Configure Reverse Proxy to Use TLS**:
   - For **NGINX**:
     ```nginx
     server {
       listen 443 ssl;
       server_name app1.home.cyber-bronkx.com;

       ssl_certificate /etc/nginx/certs/app1.crt;
       ssl_certificate_key /etc/nginx/certs/app1.key;

       location / {
         proxy_pass http://app1:8080;
       }
     }
     ```
   - For **Traefik**:
     ```yaml
     labels:
       - "traefik.http.routers.app1.tls=true"
       - "traefik.http.routers.app1.tls.certresolver=internal"
     ```

     You’ll need to define a custom certificate resolver in Traefik’s config to use your XCA certs.

---

## 🧠 Tips for Managing TLS Internally

- Use short expiration periods for internal certs (e.g. 90 days)
- Automate renewal with scripts or reminders
- Keep your CA private and backed up
- Use different certs for internal vs public-facing apps

---

## 🧪 Optional: Let’s Encrypt for Public Apps

If you expose `blog.cyber-bronkx.com` to the internet, you can use **Let’s Encrypt** for free TLS certs.

- Traefik supports automatic Let’s Encrypt integration
- NGINX requires Certbot or manual setup

---

Would you like me to now compile all of this into a Copilot Page so you can edit and expand it over time? I’ll include headings, code blocks, and examples so it’s easy to maintain.