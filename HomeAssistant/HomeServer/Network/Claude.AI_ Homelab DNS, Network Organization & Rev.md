---
title: 'Claude.AI: Homelab DNS, Network Organization & Reverse Proxy Guide'
updated: 2025-08-05 17:48:39Z
created: 2025-08-05 17:48:14Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

# Claude.AI: Homelab DNS, Network Organization & Reverse Proxy Guide

## Table of Contents

1.  [Pi-hole Local DNS Configuration](#pi-hole-local-dns-configuration)
2.  [Network Organization Best Practices](#network-organization-best-practices)
3.  [Reverse Proxy Implementation](#reverse-proxy-implementation)
4.  [Domain Integration Strategy](#domain-integration-strategy)
5.  [Security Considerations](#security-considerations)
6.  [Troubleshooting](#troubleshooting)

* * *

## Pi-hole Local DNS Configuration

### Understanding Pi-hole DNS Resolution

Pi-hole can act as both an ad-blocker and a local DNS resolver. For local DNS entries, Pi-hole uses two main methods:

1.  **Local DNS Records** (recommended for static entries)
2.  **Custom DNS entries** via the web interface

### Method 1: Local DNS Records (Recommended)

#### Configuration via Web Interface

1.  **Access Pi-hole Admin Panel**
    
    - Navigate to `http://pi-hole-ip/admin`
    - Login with your admin password
2.  **Add Local DNS Records**
    
    - Go to **Local DNS** → **DNS Records**
    - Add entries in format: `hostname.local` → `IP address`
3.  **Example Entries**
    
    ```
    proxmox.local       → 192.168.1.10
    docker.local        → 192.168.1.20
    nas.local           → 192.168.1.30
    homeassistant.local → 192.168.1.40
    ```
    

#### Configuration via File (Advanced)

Edit `/etc/hosts` on the Pi-hole server:

```bash
sudo nano /etc/hosts
```

Add your local entries:

```
192.168.1.10    proxmox.local proxmox
192.168.1.20    docker.local docker
192.168.1.30    nas.local nas
192.168.1.40    homeassistant.local homeassistant
```

### Method 2: Custom DNS via dnsmasq

For more advanced configurations, edit the dnsmasq configuration:

```bash
sudo nano /etc/dnsmasq.d/02-local.conf
```

Add custom entries:

```bash
# Local domain resolution
address=/proxmox.lab/192.168.1.10
address=/docker.lab/192.168.1.20
address=/nas.lab/192.168.1.30

# Wildcard subdomains for services
address=/.apps.lab/192.168.1.20
```

Restart Pi-hole:

```bash
sudo systemctl restart pihole-FTL
```

* * *

## Network Organization Best Practices

### Recommended Network Segmentation

#### VLAN Structure

```
VLAN 10 - Management (192.168.10.0/24)
├── Router/Firewall: 192.168.10.1
├── Proxmox Hosts: 192.168.10.10-19
├── Pi-hole: 192.168.10.53
└── Network Equipment: 192.168.10.240-254

VLAN 20 - Servers (192.168.20.0/24)
├── Production VMs: 192.168.20.10-99
├── Docker Hosts: 192.168.20.100-149
└── Database Servers: 192.168.20.150-199

VLAN 30 - Services (192.168.30.0/24)
├── Web Services: 192.168.30.10-99
├── Media Services: 192.168.30.100-149
└── IoT/Home Automation: 192.168.30.150-199

VLAN 40 - DMZ (192.168.40.0/24)
├── Reverse Proxy: 192.168.40.10
├── Public-facing services: 192.168.40.20-99
└── VPN Endpoints: 192.168.40.100-149
```

### DNS Naming Convention

#### Recommended Scheme

```
# Infrastructure
proxmox01.mgmt.lab      → 192.168.10.10
proxmox02.mgmt.lab      → 192.168.10.11
pihole.mgmt.lab         → 192.168.10.53

# Services
docker01.srv.lab        → 192.168.20.100
nas.srv.lab             → 192.168.20.150
db01.srv.lab            → 192.168.20.160

# Applications
nginx.web.lab           → 192.168.30.10
jellyfin.media.lab      → 192.168.30.100
homeassistant.iot.lab   → 192.168.30.150

# External-facing (DMZ)
proxy.dmz.lab           → 192.168.40.10
```

### Pi-hole Configuration for Multiple VLANs

#### Enable Pi-hole to Listen on Multiple Interfaces

Edit `/etc/pihole/setupVars.conf`:

```bash
PIHOLE_INTERFACE=eth0
PIHOLE_DNS_1=1.1.1.1
PIHOLE_DNS_2=1.0.0.1
DNSMASQ_LISTENING=all
```

#### Configure Conditional Forwarding

In Pi-hole admin panel:

- **Settings** → **DNS** → **Conditional Forwarding**
    
- Enable and configure:
    
    ```
    Local network in CIDR: 192.168.0.0/16IP address of DHCP server: 192.168.10.1Local domain name: lab
    ```
    

* * *

## Reverse Proxy Implementation

### What is a Reverse Proxy?

A reverse proxy sits between clients and your backend services, providing:

- **SSL Termination**: Handle HTTPS certificates centrally
- **Load Balancing**: Distribute traffic across multiple backends
- **Authentication**: Centralized login/access control
- **Subdomain Routing**: Route different subdomains to different services
- **Security**: Hide backend infrastructure, provide WAF capabilities

### Recommended Reverse Proxy Solutions

#### Option 1: Nginx Proxy Manager (Easiest)

**Docker Compose Setup:**

```yaml
version: '3'
services:
  nginx-proxy-manager:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    environment:
      - PUID=1000
      - PGID=1000
```

**Configuration Steps:**

1.  Access web UI at `http://server-ip:81`
2.  Default login: `admin@example.com` / `changeme`
3.  Add proxy hosts for each service
4.  Configure SSL certificates (Let’s Encrypt integration)

#### Option 2: Traefik (Docker-Native)

**Docker Compose Setup:**

```yaml
version: '3.7'
services:
  traefik:
    image: traefik:v2.10
    container_name: traefik
    restart: unless-stopped
    command:
      - --api.dashboard=true
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.letsencrypt.acme.email=your@email.com
      - --certificatesresolvers.letsencrypt.acme.storage=/acme.json
      - --certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./acme.json:/acme.json
    labels:
      - traefik.enable=true
      - traefik.http.routers.dashboard.rule=Host(`traefik.yourdomain.com`)
      - traefik.http.services.dashboard.loadbalancer.server.port=8080
```

### Service Configuration Examples

#### Proxmox Web Interface

```yaml
# Nginx Proxy Manager
Scheme: https
Forward Hostname/IP: 192.168.10.10
Forward Port: 8006
Websockets Support: Yes
SSL Certificate: Request new (Let's Encrypt)
```

#### Docker Services with Traefik

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    labels:
      - traefik.enable=true
      - traefik.http.routers.jellyfin.rule=Host(`jellyfin.cyber-bronkc.com`)
      - traefik.http.routers.jellyfin.tls.certresolver=letsencrypt
      - traefik.http.services.jellyfin.loadbalancer.server.port=8096
```

* * *

## Domain Integration Strategy

### Current Setup Analysis

- **Domain**: cyber-bronkc.com
- **Subdomain**: home.cyber-bronkc.com → Router IP
- **Goal**: Access internal services via subdomains

### Recommended Architecture

#### External DNS Configuration

Set up subdomains in your domain registrar:

```
*.home.cyber-bronkc.com → Your Router IP
```

#### Router Port Forwarding

Forward ports 80 and 443 to your reverse proxy:

```
Port 80 → 192.168.40.10:80
Port 443 → 192.168.40.10:443
```

#### Internal DNS Override (Pi-hole)

Configure Pi-hole to resolve subdomains internally:

```bash
# In /etc/dnsmasq.d/02-local.conf
address=/proxmox.home.cyber-bronkc.com/192.168.10.10
address=/docker.home.cyber-bronkc.com/192.168.20.100
address=/nas.home.cyber-bronkc.com/192.168.20.150
address=/jellyfin.home.cyber-bronkc.com/192.168.30.100
```

#### Service Access Matrix

```
External Access:
https://proxmox.home.cyber-bronkc.com → Router → Reverse Proxy → Proxmox

Internal Access:
https://proxmox.home.cyber-bronkc.com → Pi-hole → Direct to Proxmox
```

### Split-Brain DNS Benefits

- **Internal**: Direct access, faster response, no internet dependency
- **External**: Secure access through reverse proxy
- **Consistent URLs**: Same URLs work internally and externally

* * *

## Security Considerations

### Firewall Rules

#### Proxmox/Router Firewall

```bash
# Allow Pi-hole DNS
iptables -A INPUT -p udp --dport 53 -s 192.168.0.0/16 -j ACCEPT
iptables -A INPUT -p tcp --dport 53 -s 192.168.0.0/16 -j ACCEPT

# Allow reverse proxy
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Block external access to internal services
iptables -A INPUT -p tcp --dport 8006 -s ! 192.168.0.0/16 -j DROP
```

### SSL/TLS Configuration

#### Let’s Encrypt Certificates

- Use DNS-01 challenge for internal services
- Configure automatic renewal
- Implement HSTS headers

#### Example Nginx Configuration

```nginx
server {
    listen 443 ssl http2;
    server_name proxmox.home.cyber-bronkc.com;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000";
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    
    location / {
        proxy_pass https://192.168.10.10:8006;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Access Control

#### Network-Level Security

- Implement VLANs for segmentation
- Use firewall rules between VLANs
- Monitor access logs

#### Application-Level Security

- Enable 2FA where possible
- Use strong passwords
- Regular security updates
- Implement fail2ban for brute force protection

* * *

## Troubleshooting

### Common DNS Issues

#### Pi-hole Not Resolving Local Names

1.  Check Pi-hole logs: `tail -f /var/log/pihole.log`
2.  Verify dnsmasq configuration: `sudo pihole-FTL test`
3.  Restart Pi-hole: `sudo systemctl restart pihole-FTL`

#### Split-Brain DNS Not Working

1.  Verify client DNS settings point to Pi-hole
    
2.  Check conditional forwarding configuration
    
3.  Test with `nslookup` or `dig`:
    
    ```bash
    nslookup proxmox.home.cyber-bronkc.com 192.168.10.53
    ```
    

### Reverse Proxy Issues

#### SSL Certificate Problems

1.  Check certificate validity: `openssl x509 -in cert.pem -text -noout`
2.  Verify port forwarding on router
3.  Check Let’s Encrypt rate limits

#### Backend Connection Issues

1.  Verify backend service is running
2.  Check network connectivity: `telnet backend-ip port`
3.  Review proxy logs for errors

### Network Connectivity

#### VLAN Communication Issues

1.  Verify VLAN configuration on switches
2.  Check inter-VLAN routing rules
3.  Test with `ping` and `traceroute`

#### DNS Resolution Testing

```bash
# Test internal resolution
dig @192.168.10.53 proxmox.home.cyber-bronkc.com

# Test external resolution
dig @8.8.8.8 home.cyber-bronkc.com

# Check which DNS server is being used
nslookup google.com
```

* * *

## Implementation Checklist

### Phase 1: Basic Setup

- [ ] Configure Pi-hole local DNS entries
- [ ] Test internal name resolution
- [ ] Document IP address assignments
- [ ] Set up basic VLAN structure

### Phase 2: Reverse Proxy

- [ ] Deploy reverse proxy solution
- [ ] Configure SSL certificates
- [ ] Set up service routing
- [ ] Test external access

### Phase 3: Integration

- [ ] Configure split-brain DNS
- [ ] Set up domain subdomains
- [ ] Configure router port forwarding
- [ ] Test both internal and external access

### Phase 4: Security & Monitoring

- [ ] Implement firewall rules
- [ ] Set up access logging
- [ ] Configure monitoring alerts
- [ ] Create backup procedures

* * *

## Maintenance Tasks

### Weekly

- Review Pi-hole query logs
- Check SSL certificate expiration dates
- Verify backup integrity

### Monthly

- Update reverse proxy software
- Review firewall logs
- Update DNS entries as needed

### Quarterly

- Security audit of exposed services
- Review and update documentation
- Test disaster recovery procedures

* * *

## Additional Resources

- [Pi-hole Documentation](https://docs.pi-hole.net/)
- [Nginx Proxy Manager](https://nginxproxymanager.com/)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Let’s Encrypt](https://letsencrypt.org/)

* * *

*Last Updated: \[Current Date\]* *Author: \[Your Name\]*