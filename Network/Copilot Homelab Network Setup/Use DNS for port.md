---
title: Use DNS for port
updated: 2025-08-05 20:24:13Z
created: 2025-08-05 20:21:55Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

## Use DNS for port
Great question, Thierry — and it’s one that trips up a lot of homelab enthusiasts early on. DNS itself doesn’t handle ports. It only resolves domain names to IP addresses. So when you type `app.home.cyber-bronkx.com`, DNS tells your browser the IP address, but it’s up to your browser (or client) to decide which port to connect to — usually defaulting to port 80 (HTTP) or 443 (HTTPS).

So how do you get around the “port problem” when hosting multiple apps on the same IP? That’s where a **reverse proxy** comes in. Here's how it works:

---

## 🧭 Why DNS Can't Handle Ports

- DNS maps **hostnames to IP addresses**, not ports.
- You can't create a DNS record like `app1.home.cyber-bronkx.com:8080` — the `:8080` part is handled by the browser, not DNS.
- So if you have multiple services on the same IP but different ports, DNS alone can't help you route traffic cleanly.

---

## 🔀 Enter the Reverse Proxy

A **reverse proxy** acts as a smart traffic director. It listens on standard ports (80/443) and forwards requests to the correct internal service based on the hostname or path.

### Example Setup:
| Hostname | Internal Port | Docker Service |
|----------|---------------|----------------|
| `app1.home.cyber-bronkx.com` | 8081 | App 1 |
| `app2.home.cyber-bronkx.com` | 8082 | App 2 |

With a reverse proxy (like **NGINX**, **Traefik**, or **Caddy**), you can:
- Route `app1.home.cyber-bronkx.com` to `localhost:8081`
- Route `app2.home.cyber-bronkx.com` to `localhost:8082`
- All traffic comes in on port 80/443, and the proxy handles the rest.

---

## 🧰 Recommended Reverse Proxies

| Proxy | Pros | Docker-Friendly? |
|-------|------|------------------|
| **NGINX** | Powerful, widely used | ✅ |
| **Traefik** | Auto-discovers Docker containers, great for dynamic setups | ✅✅ |
| **Caddy** | Easiest HTTPS setup, automatic TLS | ✅ |

For Docker-heavy environments, **Traefik** is especially popular because it can read Docker labels and configure routes automatically.

---

## 🧠 How This Fits Into Your Homelab

- Use **Pi-Hole** to resolve `*.home.cyber-bronkx.com` to your reverse proxy’s IP.
- Use a reverse proxy to route traffic based on hostname.
- Keep your Docker services isolated and clean — no more port juggling.

---

Would you like me to include this explanation in the full Markdown guide I’m preparing for you? I can also walk you through setting up Traefik or NGINX with Docker if you want a hands-on example.