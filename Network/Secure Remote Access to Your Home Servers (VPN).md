---
title: Secure Remote Access to Your Home Servers (VPN)
updated: 2025-07-20 16:00:56Z
created: 2025-07-20 16:00:29Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

## Secure Remote Access to Your Home Servers (VPN)

This guide focuses on setting up a Virtual Private Network (VPN) to securely access your home network and servers from anywhere. This is the recommended method for personal access as it encrypts all traffic and makes your remote device "part of" your home network.

**What you'll need:**

- **A dedicated device for the VPN server:** This can be your Raspberry Pi (the same one running Pi-hole, if it has enough resources), a dedicated mini-PC, or a robust router that supports VPN server functionality.
- **Static IP Address for VPN Server:** Ensure this device has a static IP on your local network (e.g., `192.168.1.50` if it's your Pi-hole).
- **Dynamic DNS (DDNS) Service:** Your home's public IP address usually changes. A DDNS service will give you a consistent hostname for your home network.
- **Router Access:** To configure port forwarding.
- **VPN Client Software:** For your remote devices (phone, laptop).

* * *

### Step 1: Obtain a Dynamic DNS (DDNS) Hostname

Your Internet Service Provider (ISP) typically assigns your home a dynamic public IP address, meaning it can change. A DDNS service maps a consistent hostname to this changing IP.

1.  **Choose a DDNS Provider:** Popular free options include:
    - **DuckDNS:** Simple, free, good for home users.
    - **No-IP:** Offers a free tier with some limitations.
    - **FreeDNS:** Another good free option.
2.  **Sign Up and Create a Hostname:**
    - Go to your chosen DDNS provider's website.
    - Sign up for an account.
    - Create a hostname (e.g., `myhome.duckdns.org`).
3.  **Configure DDNS Client:**
    - **Router Integration (Easiest):** Many routers have built-in DDNS clients. Look for "DDNS" or "Dynamic DNS" settings in your router's interface. Enter your DDNS provider, hostname, username, and password. This is ideal as your router will automatically update your IP.
    - **Software Client:** If your router doesn't support DDNS, you can run a small client application on your Pi-hole device or another always-on server that periodically updates your DDNS hostname with your current public IP. DuckDNS, for example, provides simple scripts for this.

* * *

### Step 2: Choose and Set Up Your VPN Server

**Option A: WireGuard (Recommended for Speed and Simplicity with PiVPN)**

1.  **Install PiVPN:** If you're using a Raspberry Pi or a Debian-based Linux server, PiVPN makes WireGuard (and OpenVPN) setup very easy.
    
    - SSH into your VPN server device.
        
    - Run the installer:
        
        ```bash
        curl -L https://install.pivpn.io | bash
        ```
        
2.  **Follow PiVPN Prompts:**
    
    - **Choose VPN Protocol:** Select `WireGuard`.
    - **Local User:** Select the local user to own the VPN configuration.
    - **Static IP:** Confirm the static IP of your VPN server.
    - **Upstream DNS:** PiVPN will ask for an upstream DNS server. **Crucially, enter your Pi-hole's static IP address here (e.g., `192.168.1.50`).** This ensures that when you connect to the VPN, your Pi-hole handles DNS resolution for both internal and external queries.
    - **Public IP or DNS:** Select `DNS Entry` and enter your DDNS hostname (e.g., `myhome.duckdns.org`).
    - **Port:** Accept the default WireGuard port (UDP 51820) or choose another.
    - **Unattended Upgrades:** Recommended to enable.
    - **Reboot:** Reboot when prompted.
3.  **Generate Client Configurations:**
    
    - After rebooting and reconnecting via SSH, run: `pivpn add`
    - Give the client a name (e.g., `myphone`, `mylaptop`).
    - PiVPN will generate a `.conf` file and a QR code (for mobile clients).

**Option B: Tailscale (Easiest, No Port Forwarding)**

1.  **Sign Up for Tailscale:** Go to [tailscale.com](https://tailscale.com/) and sign up for a free account.
    
2.  **Install Tailscale on your Home Server (e.g., Pi-hole device):**
    
    - SSH into your Pi-hole device.
        
    - Follow the Tailscale installation instructions for your OS (usually a simple `curl` command).
        
    - When running `tailscale up`, use the `--accept-dns=false` flag if your Pi-hole is already handling DNS for the device itself.
        
        ```bash
        curl -fsSL https://tailscale.com/install.sh | sh
        sudo tailscale up --accept-dns=false # Use this if Pi-hole is on the same device
        # Or simply: sudo tailscale up # if Pi-hole is on a different device
        ```
        
    - This will give you a URL to authenticate the device with your Tailscale account.
        
3.  **Install Tailscale on your Remote Devices:** Download and install the Tailscale app on your phone, laptop, etc., and log in with the same account.
    
4.  **Configure MagicDNS (for Pi-hole integration):**
    
    - Go to the Tailscale admin console (https://login.tailscale.com/admin/dns).
    - Under "Nameservers," add your Pi-hole's Tailscale IP address (you can find this on the "Machines" page in the admin console, it's usually in the `100.x.y.z` range).
    - Enable "Override global DNS settings" or "Override local DNS" if you want all DNS queries to go through your Pi-hole when connected to Tailscale.
    - Under "MagicDNS," enable it and add your local domain (e.g., `home`). This allows you to resolve `myserver.home` via Tailscale.

* * *

### Step 3: Configure Router Port Forwarding (Only for WireGuard/OpenVPN)

**If you chose WireGuard or OpenVPN (not Tailscale/ZeroTier):**

1.  **Log in to your Router:** Access your router's administration interface.
2.  **Find Port Forwarding Settings:** Look for "Port Forwarding," "NAT," or "Virtual Servers."
3.  **Create a New Rule:**
    - **Service/Application Name:** Give it a descriptive name (e.g., `WireGuard VPN`).
    - **External Port (or Port Range):** Enter `51820` (or the port you chose for WireGuard).
    - **Internal Port (or Port Range):** Enter `51820`.
    - **Protocol:** Select `UDP`.
    - **Internal IP Address:** Enter the static local IP address of your VPN server (e.g., `192.168.1.50`).
    - **Enable/Save:** Enable the rule and save the settings.

* * *

### Step 4: Connect from Your Remote Device

**For WireGuard (via PiVPN):**

1.  **Download WireGuard Client:** Get the official WireGuard app for your phone, laptop, or tablet.
2.  **Import Configuration:**
    - **Mobile:** Open the WireGuard app, tap the `+` icon, and scan the QR code generated by `pivpn add`.
    - **Desktop:** Copy the `.conf` file generated by `pivpn add` to your desktop and import it into the WireGuard client.
3.  **Activate Connection:** Turn on the VPN connection in the WireGuard app.

**For Tailscale:**

1.  **Install Tailscale Client:** Download and install the Tailscale app on your remote device.
2.  **Log In:** Log in with the same Tailscale account you used for your home server.
3.  **Connect:** The device will automatically join your Tailscale network. You should now be able to access your home servers by their local IP or the custom hostnames you set up in Pi-hole (e.g., `myserver.home`).

* * *

### Step 5: Test Remote Access

1.  **Disconnect from Home Wi-Fi:** On your remote device, disconnect from your home Wi-Fi and use mobile data or another Wi-Fi network.
2.  **Activate VPN:** Turn on your VPN connection (WireGuard or Tailscale).
3.  **Test Access:**
    - Try pinging your servers by their local IP addresses (e.g., `ping 192.168.1.100`).
    - Try accessing your servers by their custom Pi-hole hostnames (e.g., `ping myserver.home`, `http://plex.home:32400`).
    - You should now be able to reach them as if you were at home.

* * *

### Important Security Considerations for Remote Access:

- **Strong Passwords & 2FA:** Use strong, unique passwords for all your server logins, router, Pi-hole, and VPN services. Enable Two-Factor Authentication (2FA) wherever possible.
- **Keep Software Updated:** Regularly update your Pi-hole, VPN software, server operating systems, and router firmware.
- **Minimal Port Forwarding:** If using WireGuard/OpenVPN, only forward the *single* VPN port. Never forward ports for individual services directly unless absolutely necessary and you understand the risks.
- **Firewall:** Ensure your home router's firewall is active. On your Linux servers, use a firewall (like `ufw` on Ubuntu/Debian) to only allow necessary incoming connections (e.g., SSH from your local network, VPN traffic from the VPN interface).
- **Reverse Proxy (for public web services):** If you want to expose web services to the internet via a domain name (e.g., `photos.yourdomain.com`) without a VPN, a reverse proxy like Nginx Proxy Manager or Cloudflare Tunnel is much safer than direct port forwarding for each service. This allows you to centralize SSL, security, and only expose ports 80/443.
- **Consider Zero Trust Networks:** Solutions like Tailscale and Cloudflare Tunnel are part of a "Zero Trust" approach, which is a modern security model that assumes no user or device is inherently trustworthy, even if they are inside the network perimeter. This offers a higher level of security for remote access.

By following these guides, you'll have a robust and secure home network setup for both local and remote access!