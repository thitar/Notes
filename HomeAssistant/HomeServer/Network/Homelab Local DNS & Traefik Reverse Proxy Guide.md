---
title: Homelab Local DNS & Traefik Reverse Proxy Guide
updated: 2025-08-05 21:33:01Z
created: 2025-08-05 21:32:57Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

# Homelab Local DNS & Traefik Reverse Proxy Guide

## Overview of Your Infrastructure

Before diving into the configuration, let’s understand how your network components work together. You have three main hosts: **Dietpi** (Pi Zero 2) running Pi-hole and Unbound for DNS services, **Docker** (Debian VM on Proxmox) running most containerized services, and **Intelnuc** (WSL) running media containers like Jellyfin. The goal is to create a seamless local network where you can access services using friendly names like `jellyfin.local` instead of remembering IP addresses and ports.

The `.local` domain approach is particularly elegant because it leverages mDNS (Multicast DNS) principles while being overridden by your Pi-hole for centralized control. This means your devices will naturally look for `.local` addresses through your DNS server rather than broadcasting on the network.

## Understanding the Reverse Proxy Concept

Think of Traefik as a smart doorman for your services. When you type `jellyfin.local` in your browser, Traefik receives this request and knows to forward it to your Intelnuc server on port 8096. This eliminates the need to remember port numbers and provides a single point for SSL certificate management. Traefik can automatically obtain and renew SSL certificates, making HTTPS effortless across all your services.

The beauty of this setup is that Traefik watches your Docker containers and automatically configures routing based on labels you add to your containers. When you start a new service, Traefik immediately knows how to route traffic to it without manual configuration files.

## Phase 1: Pi-hole Local DNS Configuration

### Setting Up .local Domain Resolution

Your Pi-hole installation needs to handle `.local` domain queries instead of letting them fall back to mDNS. This gives you centralized control over all local name resolution.

**Step 1: Configure Pi-hole via Web Interface**

Access your Pi-hole admin panel at `http://dietpi.local/admin` (assuming your Pi Zero is accessible via mDNS initially). Navigate to **Local DNS** and then **DNS Records**. Here you’ll add entries for each of your hosts and services.

Start with these basic host entries:

```
dietpi.local     → [Pi Zero IP address]
docker.local     → [Docker VM IP address]
intelnuc.local   → [Intel NUC IP address]
```

The reason we start with host-level entries is to establish the foundation. Later, we’ll add service-specific entries that point to the same IPs but will be handled differently by Traefik based on the hostname.

**Step 2: Add Service Entries**

For each service you want to access, add DNS entries that point to the host running Traefik. If you’re running Traefik on the Docker VM, then all service entries should point to the Docker VM’s IP address:

```
jellyfin.local      → [Docker VM IP address]
sonarr.local        → [Docker VM IP address]
radarr.local        → [Docker VM IP address]
portainer.local     → [Docker VM IP address]
```

This might seem counterintuitive at first - why do all services point to the same IP? The answer lies in how Traefik works. It receives all incoming requests on ports 80 and 443, examines the hostname in the request, and then forwards the request to the appropriate backend service, which might be running on the same host or a different host entirely.

### Understanding DNS Resolution Flow

When you type `jellyfin.local` in your browser, here’s what happens:

1.  Your device asks Pi-hole “What’s the IP address for jellyfin.local?”
2.  Pi-hole responds with the Docker VM’s IP address
3.  Your browser connects to the Docker VM on port 80 or 443
4.  Traefik receives the request, sees the hostname is “jellyfin.local”
5.  Traefik forwards the request to intelnuc:8096 where Jellyfin is actually running
6.  The response flows back through Traefik to your browser

This separation of concerns allows for incredible flexibility. You can move services between hosts, change ports, or add load balancing without ever changing DNS entries.

## Phase 2: Traefik Configuration and Deployment

### Understanding Traefik’s Architecture

Traefik operates on the concept of **providers**, **routers**, **services**, and **middlewares**. The Docker provider watches your Docker daemon for containers with specific labels. When it finds them, it automatically creates routes based on those labels. This dynamic configuration means you rarely need to edit configuration files.

**Step 1: Create Traefik Directory Structure**

On your Docker VM, create a dedicated directory for Traefik:

```bash
mkdir -p ~/traefik/{config,certificates}
cd ~/traefik
```

The certificates directory will store your SSL certificates, while the config directory will hold any static configuration files we might need.

**Step 2: Create Docker Compose File**

Create a `docker-compose.yml` file that sets up Traefik with all the features you need:

```yaml
version: '3.8'

services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    
    # Command line arguments configure Traefik's behavior
    command:
      # Enable the web dashboard - you'll access this via traefik.local
      - --api.dashboard=true
      - --api.insecure=true  # Remove this in production
      
      # Configure Docker as a provider
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --providers.docker.network=traefik
      
      # Set up entrypoints (ports Traefik listens on)
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      
      # Configure automatic HTTPS with Let's Encrypt
      - --certificatesresolvers.letsencrypt.acme.email=your-email@example.com
      - --certificatesresolvers.letsencrypt.acme.storage=/certificates/acme.json
      - --certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web
      
      # Enable access logs for troubleshooting
      - --accesslog=true
      - --log.level=INFO
      
    ports:
      # HTTP port - Traefik will redirect to HTTPS automatically
      - "80:80"
      # HTTPS port - where your secured services will be available
      - "443:443"
      # Dashboard port (optional if using labels below)
      - "8080:8080"
      
    volumes:
      # Give Traefik access to Docker socket to watch for containers
      - /var/run/docker.sock:/var/run/docker.sock:ro
      # Persistent storage for SSL certificates
      - ./certificates:/certificates
      
    networks:
      - traefik
      
    labels:
      # Enable Traefik for this container
      - traefik.enable=true
      
      # Dashboard configuration
      - traefik.http.routers.dashboard.rule=Host(`traefik.local`)
      - traefik.http.routers.dashboard.service=api@internal
      - traefik.http.routers.dashboard.tls.certresolver=letsencrypt

networks:
  traefik:
    external: true
```

**Step 3: Create the Traefik Network**

Before starting Traefik, create the network that will connect all your services:

```bash
docker network create traefik
```

This network allows containers to communicate with Traefik even if they’re defined in separate Docker Compose files. It’s a best practice that keeps your configurations modular and manageable.

**Step 4: Start Traefik**

```bash
docker-compose up -d
```

After starting, you should be able to access the Traefik dashboard by adding `traefik.local` to your Pi-hole DNS records pointing to your Docker VM, then visiting `http://traefik.local` in your browser.

### Understanding SSL Certificate Management

Traefik’s automatic SSL certificate management is one of its most powerful features. When a new service starts with proper labels, Traefik automatically requests a certificate from Let’s Encrypt, stores it securely, and renews it before expiration.

The ACME (Automatic Certificate Management Environment) protocol handles this entire process. Traefik proves domain ownership using the HTTP challenge method, where Let’s Encrypt sends a verification request to your domain, and Traefik responds appropriately.

For local services using `.local` domains, Let’s Encrypt cannot issue certificates since it cannot verify ownership of `.local` domains over the internet. We’ll address this by using self-signed certificates or a local Certificate Authority for internal services, while using Let’s Encrypt certificates for any services exposed via `home.cyber-bronckx.com`.

## Phase 3: Configuring Services with Traefik

### Understanding Service Configuration

Each Docker container that you want to access through Traefik needs specific labels that tell Traefik how to route traffic to it. These labels define the hostname, port, and SSL settings for each service.

**Step 1: Configure Jellyfin on Intel NUC**

Since Jellyfin runs on your Intel NUC with WSL, you have two approaches. The cleanest approach is to run Jellyfin as a Docker container with Traefik labels, but connected to the same Traefik network.

Create a `docker-compose.yml` file on your Intel NUC:

```yaml
version: '3.8'

services:
  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    restart: unless-stopped
    
    ports:
      # You can remove these ports since Traefik will handle routing
      # - "8096:8096"
      
    volumes:
      - ./config:/config
      - ./cache:/cache
      - /path/to/media:/media:ro
      
    networks:
      - traefik
      
    labels:
      # Enable Traefik for this container
      - traefik.enable=true
      
      # Define the hostname for this service
      - traefik.http.routers.jellyfin.rule=Host(`jellyfin.local`)
      
      # Specify which port Traefik should forward to
      - traefik.http.services.jellyfin.loadbalancer.server.port=8096
      
      # Enable automatic SSL
      - traefik.http.routers.jellyfin.tls=true
      - traefik.http.routers.jellyfin.tls.certresolver=letsencrypt
      
      # Optional: redirect HTTP to HTTPS
      - traefik.http.routers.jellyfin-insecure.rule=Host(`jellyfin.local`)
      - traefik.http.routers.jellyfin-insecure.entrypoints=web
      - traefik.http.routers.jellyfin-insecure.middlewares=redirect-to-https
      - traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https

networks:
  traefik:
    external: true
```

The key insight here is that all containers participating in Traefik routing must be on the same Docker network. This allows Traefik to communicate directly with your services using Docker’s internal networking, which is more secure and efficient than exposing ports to the host.

**Step 2: Configure Services on Docker VM**

For services running on your main Docker VM, the configuration is similar but simpler since they’re on the same host as Traefik:

```yaml
version: '3.8'

services:
  portainer:
    image: portainer/portainer-ce
    container_name: portainer
    restart: unless-stopped
    
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
      
    networks:
      - traefik
      
    labels:
      - traefik.enable=true
      - traefik.http.routers.portainer.rule=Host(`portainer.local`)
      - traefik.http.services.portainer.loadbalancer.server.port=9000
      - traefik.http.routers.portainer.tls=true

  sonarr:
    image: linuxserver/sonarr
    container_name: sonarr
    restart: unless-stopped
    
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Brussels
      
    volumes:
      - ./sonarr:/config
      - /path/to/downloads:/downloads
      - /path/to/tv:/tv
      
    networks:
      - traefik
      
    labels:
      - traefik.enable=true
      - traefik.http.routers.sonarr.rule=Host(`sonarr.local`)
      - traefik.http.services.sonarr.loadbalancer.server.port=8989
      - traefik.http.routers.sonarr.tls=true

volumes:
  portainer_data:

networks:
  traefik:
    external: true
```

### Understanding Cross-Host Container Communication

When Traefik on your Docker VM needs to communicate with Jellyfin on your Intel NUC, Docker’s networking handles this automatically if both containers are on the same network. However, since they’re on different physical hosts, you need to create an overlay network or use Docker Swarm mode.

A simpler approach for your setup is to configure Traefik to forward requests to external services using IP addresses and ports. Modify your Jellyfin configuration to use a different approach:

**Alternative Jellyfin Configuration (Recommended for Cross-Host)**

Instead of running Jellyfin in the Traefik network, configure Traefik to forward to the external IP:

```yaml
# Add this to your Traefik compose file as an additional service
  jellyfin-proxy:
    image: alpine:latest
    container_name: jellyfin-proxy
    restart: "no"
    networks:
      - traefik
    labels:
      - traefik.enable=true
      - traefik.http.routers.jellyfin.rule=Host(`jellyfin.local`)
      - traefik.http.services.jellyfin.loadbalancer.server.url=http://[Intel NUC IP]:8096
      - traefik.http.routers.jellyfin.tls=true
    command: tail -f /dev/null
```

This approach creates a “phantom” container that exists only to provide Traefik with routing information to your external service.

## Phase 4: SSL Certificate Management

### Understanding Local SSL Certificates

Since Let’s Encrypt cannot issue certificates for `.local` domains, you have several options for securing your internal services:

**Option 1: Self-Signed Certificates (Simplest)**

Traefik can generate self-signed certificates automatically. Add this configuration to your Traefik command section:

```yaml
# Add to your Traefik command section
- --certificatesresolvers.local.acme.caserver=https://localhost:4001/directory
```

This tells Traefik to use a local certificate authority. You’ll get browser warnings, but the traffic will be encrypted.

**Option 2: Local Certificate Authority (Recommended)**

Create your own Certificate Authority for your homelab. This requires more setup but provides trusted certificates across all your devices.

**Step 1: Create CA on Traefik Host**

```bash
# Install mkcert for easy local CA management
curl -JLO "https://dl.filippo.io/mkcert/latest?for=linux/amd64"
chmod +x mkcert-v*-linux-amd64
sudo mv mkcert-v*-linux-amd64 /usr/local/bin/mkcert

# Create local CA
mkcert -install

# Generate wildcard certificate for .local domain
mkcert "*.local" localhost 127.0.0.1
```

**Step 2: Configure Traefik to Use Local Certificates**

Create a dynamic configuration file that tells Traefik about your certificates:

```yaml
# Create ~/traefik/config/dynamic.yml
tls:
  certificates:
    - certFile: /certificates/_wildcard.local.pem
      keyFile: /certificates/_wildcard.local-key.pem
```

Add this volume mount to your Traefik container:

```yaml
volumes:
  - ./config:/config:ro
```

And add this command:

```yaml
command:
  - --providers.file.directory=/config
  - --providers.file.watch=true
```

**Step 3: Install CA on Client Devices**

Copy the CA certificate to all devices that need to trust your certificates. The CA certificate is typically located at `~/.local/share/mkcert/rootCA.pem`.

### Understanding External SSL with Let’s Encrypt

For services you want to access externally via `home.cyber-bronckx.com`, Let’s Encrypt can issue proper certificates. Configure additional routers in your service labels:

```yaml
labels:
  # Internal access
  - traefik.http.routers.jellyfin-internal.rule=Host(`jellyfin.local`)
  - traefik.http.routers.jellyfin-internal.tls=true
  
  # External access
  - traefik.http.routers.jellyfin-external.rule=Host(`jellyfin.home.cyber-bronckx.com`)
  - traefik.http.routers.jellyfin-external.tls=true
  - traefik.http.routers.jellyfin-external.tls.certresolver=letsencrypt
```

This dual configuration allows the same service to be accessible both internally with your local certificates and externally with Let’s Encrypt certificates.

## Phase 5: Security and Network Considerations

### Understanding Network Security

Your approach of keeping external threats out while maintaining internal convenience is sound. The key is to ensure that only the services you explicitly want to expose externally are accessible from the internet.

**Step 1: Configure Router Port Forwarding**

Only forward ports 80 and 443 to your Traefik instance:

```
Router Port 80 → Docker VM:80
Router Port 443 → Docker VM:443
```

**Step 2: Configure Traefik Access Control**

Use Traefik middlewares to add authentication to sensitive services:

```yaml
labels:
  # Add authentication middleware
  - traefik.http.routers.portainer.middlewares=auth
  - traefik.http.middlewares.auth.basicauth.users=admin:$$2y$$10$$hashed-password
```

Generate the hashed password using:

```bash
htpasswd -nb admin yourpassword
```

**Step 3: Network Segmentation**

Consider creating separate Docker networks for different types of services:

```bash
docker network create traefik-internal
docker network create traefik-external
```

Connect only externally-facing services to the external network, while keeping purely internal services on the internal network.

## Phase 6: Monitoring and Troubleshooting

### Understanding Traefik’s Observability

Traefik provides excellent visibility into your network traffic through its dashboard and logs. The dashboard shows all configured routes, services, and their health status in real-time.

**Step 1: Enable Detailed Logging**

Add these commands to your Traefik configuration for better troubleshooting:

```yaml
command:
  - --log.level=DEBUG  # Change to INFO for production
  - --accesslog=true
  - --accesslog.filepath=/var/log/traefik/access.log
  - --metrics.prometheus=true
```

**Step 2: Common Troubleshooting Steps**

When a service isn’t accessible, check these items in order:

1.  **DNS Resolution**: Can you resolve the hostname from your client?
    
    ```bash
    nslookup jellyfin.local
    ```
    
2.  **Network Connectivity**: Can you reach Traefik?
    
    ```bash
    curl -I http://docker.local
    ```
    
3.  **Service Discovery**: Does Traefik see your service? Check the dashboard at `traefik.local`.
    
4.  **Backend Health**: Is your actual service running?
    
    ```bash
    curl -I http://intelnuc.local:8096
    ```
    

### Understanding Pi-hole Integration

Your Pi-hole logs will show DNS queries, which helps troubleshoot resolution issues. Access logs at `http://dietpi.local/admin` and look for queries to your `.local` domains.

If clients aren’t using Pi-hole for DNS, check that your DHCP server (likely your router) is configured to hand out Pi-hole’s IP address as the primary DNS server.

## Implementation Roadmap

### Phase 1: Foundation (Week 1)

Start by configuring your basic DNS entries in Pi-hole for your three hosts. Test that you can access each host by name from any device on your network. This establishes the foundation for everything else.

### Phase 2: Traefik Deployment (Week 2)

Deploy Traefik on your Docker VM and get the dashboard working. This proves that Traefik is receiving requests and can route them appropriately. Don’t worry about SSL certificates yet - focus on getting HTTP routing working first.

### Phase 3: Service Integration (Week 3)

Add one service at a time to Traefik, starting with something simple like Portainer that runs on the same host as Traefik. Once you have local services working, tackle the cross-host communication for Jellyfin.

### Phase 4: SSL Implementation (Week 4)

Implement your chosen SSL solution. Start with self-signed certificates for simplicity, then upgrade to a local CA if you want to eliminate browser warnings.

### Phase 5: External Access (Week 5)

Configure external access for services you want to reach from outside your network. Test thoroughly to ensure security boundaries are maintained.

This phased approach ensures that each component works before adding complexity. If something breaks, you’ll know exactly which change caused the issue.

## Maintenance and Best Practices

### Regular Maintenance Tasks

**Weekly**: Check Traefik logs for any unusual activity or failed certificate renewals. Review Pi-hole query logs to ensure DNS resolution is working correctly across all devices.

**Monthly**: Update Traefik and other container images. Test external access to ensure DDNS and port forwarding are still working correctly.

**Quarterly**: Review and update SSL certificates, especially if using a local CA. Audit which services are exposed externally and whether they still need external access.

### Backup Considerations

Your critical configuration includes Pi-hole settings, Traefik certificates, and Docker Compose files. Consider backing up:

1.  Pi-hole configuration: `/etc/pihole/` directory
2.  Traefik certificates: `~/traefik/certificates/` directory
3.  All Docker Compose files and their associated configuration directories

### Growth and Scaling

This architecture scales well as you add more services. Each new Docker container needs only the appropriate labels to be automatically integrated into your routing system. The centralized DNS management through Pi-hole means adding new services requires only a single DNS entry and proper container labels.

As your homelab grows, consider implementing monitoring solutions like Prometheus and Grafana, which integrate excellently with Traefik’s built-in metrics. This provides visibility into performance and usage patterns across all your services.

The modular nature of this setup means you can also easily migrate services between hosts by simply changing the IP address in your service configuration - the DNS entries and SSL certificates remain unchanged, minimizing disruption to users.