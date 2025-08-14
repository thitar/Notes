---
title: 'Llama: Using Pi-Hole for Local DNS Resolution'
updated: 2025-08-05 18:07:49Z
created: 2025-08-05 18:06:11Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

## Llama: Using Pi-Hole for Local DNS Resolution

```

```

Pi-Hole is a powerful tool for ad blocking and DNS management. Here’s a step-by-step guide on how to use Pi-Hole to resolve DNS entries locally.

```

```

Step 1: Configure Pi-Hole

```

```

- Access your Pi-Hole web interface by navigating to `http://pihole-ip/admin` (replace `pihole-ip` with the IP address of your Pi-Hole server).
    
- Go to **Settings** > **DNS** and select the DNS server you want to use (e.g., Google, Cloudflare, or OpenDNS).
    
- Ensure that **Never forward non-FQDNs** and **Never forward reverse lookups for private IP ranges** are enabled.
    

```

```

Step 2: Add Local DNS Entries

```

```

- Go to **Local DNS** > **DNS Records**.
    
- Click on **Add a new domain** and enter the domain name you want to resolve locally (e.g., `cyber-bronkc.local` or `home.cyber-bronkc.com`).
    
- Enter the IP address of the server or device you want to point the domain to.
    
- Click **Add** to save the record.
    

```

```

Step 3: Configure DHCP Settings (Optional)

```

```

If you want Pi-Hole to assign IP addresses to devices on your network, you’ll need to configure DHCP settings.

```

```

- Go to **Settings** > **DHCP**.
    
- Enable **DHCP server enabled**.
    
- Set the IP range, gateway, and other settings according to your network requirements.
    

```

```

Recommended Network Organization

```

```

- **Use a consistent naming convention**: Use a consistent naming convention for your devices, servers, and DNS entries to avoid confusion.
    
- **Use a local domain**: Use a local domain (e.g., `.local` or `.lan`) for your internal network to avoid conflicts with public DNS entries.
    
- **Use static IP addresses**: Assign static IP addresses to servers and devices that require consistent access.
    
- **Segment your network**: Segment your network into different subnets or VLANs to improve security and organization.
    

```

```

Reverse Proxy

```

```

A reverse proxy is a server that sits between the internet and your internal network, forwarding requests to your servers or devices. It’s commonly used for:

```

```

- **Load balancing**: Distribute incoming traffic across multiple servers.
    
- **Security**: Hide internal IP addresses and protect servers from external attacks.
    
- **Caching**: Cache frequently accessed content to improve performance.
    

```

```

In your configuration, you can use a reverse proxy to:

```

```

- Expose your internal servers or services to the internet securely.
    
- Route traffic to different servers or services based on domain names or URLs.
    

```

```

Some popular reverse proxy solutions include:

```

```

- **NGINX**: A popular web server and reverse proxy solution.
    
- **Traefik**: A modern reverse proxy solution designed for containerized environments.
    
- **HAProxy**: A high-performance load balancer and reverse proxy solution.
    

```

```

Using Reverse Proxy with Pi-Hole

```

```

You can use a reverse proxy in conjunction with Pi-Hole to:

```

```

- Route traffic to your Pi-Hole server securely.
    
- Expose your internal services to the internet while keeping your internal IP addresses private.
    

```

```

To set up a reverse proxy with Pi-Hole, you’ll need to:

```

```

- Configure your reverse proxy solution to forward requests to your Pi-Hole server.
    
- Update your Pi-Hole configuration to use the reverse proxy’s IP address or domain name.
    

```

```

Example Configuration

```

```

Let’s say you want to expose your Pi-Hole server to the internet using a reverse proxy. You can configure NGINX to forward requests to your Pi-Hole server like this:

```

```

Nginx

```

```

```
http {
    upstream pihole {
        server pihole-ip:80;
    }

    server {
        listen 80;
        server_name home.cyber-bronkc.com;

        location / {
            proxy_pass http://pihole;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

```

```

In this example, NGINX listens for requests on port 80 and forwards them to your Pi-Hole server.

```

```

Using Your Domain Name

```

```

Since your domain name `home.cyber-bronkc.com` points to your internet router, you’ll need to configure port forwarding or a similar setup to forward requests to your Pi-Hole server or reverse proxy.

```

```

You can configure your router to forward requests to your Pi-Hole server or reverse proxy like this:

```

```

- Forward requests from `home.cyber-bronkc.com:80` to your Pi-Hole server’s IP address and port (e.g., `pihole-ip:80`).

```

```

By following these steps and configuring your network and Pi-Hole setup accordingly, you can use Pi-Hole to resolve DNS entries locally and securely expose your internal services to the internet using a reverse proxy.

```

```

**Markdown Checklist**

```

```

- Configure Pi-Hole DNS settings
    
- Add local DNS entries
    
- Configure DHCP settings (optional)
    
- Use a consistent naming convention
    
- Use a local domain
    
- Use static IP addresses
    
- Segment your network
    
- Configure reverse proxy (optional)
    
- Configure port forwarding on your router (optional)
    

```

```

**Further Reading**

```

```

- Pi-Hole documentation: [https://docs.pi-hole.net/](https://l.meta.ai/?u=https%3A%2F%2Fdocs.pi-hole.net%2F&h=AT31ykbBMBHj_YAmz4qk-wWcA3sLAUkjejOCvfQUxsOxBfoc_pUlBhuYaMD8D6CdwPYnoiCBut6jKyE4fuSC4i2y7PxzwVsA88xYATAur6s0wqrrfJPJiREJG2mvZJSeyGmVEs4zKfrrpIq184wPe8-Ph6n8_zqSG62eKw)
    
- NGINX documentation: [https://nginx.org/en/docs/](https://l.meta.ai/?u=https%3A%2F%2Fnginx.org%2Fen%2Fdocs%2F&h=AT31ykbBMBHj_YAmz4qk-wWcA3sLAUkjejOCvfQUxsOxBfoc_pUlBhuYaMD8D6CdwPYnoiCBut6jKyE4fuSC4i2y7PxzwVsA88xYATAur6s0wqrrfJPJiREJG2mvZJSeyGmVEs4zKfrrpIq184wPe8-Ph6n8_zqSG62eKw)
    
- Traefik documentation: [https://doc.traefik.io/traefik/](https://l.meta.ai/?u=https%3A%2F%2Fdoc.traefik.io%2Ftraefik%2F&h=AT31ykbBMBHj_YAmz4qk-wWcA3sLAUkjejOCvfQUxsOxBfoc_pUlBhuYaMD8D6CdwPYnoiCBut6jKyE4fuSC4i2y7PxzwVsA88xYATAur6s0wqrrfJPJiREJG2mvZJSeyGmVEs4zKfrrpIq184wPe8-Ph6n8_zqSG62eKw)
    

```

```