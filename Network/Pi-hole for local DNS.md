---
title: Pi-hole for local DNS
updated: 2025-07-20 16:00:46Z
created: 2025-07-20 15:58:45Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

&nbsp;

## Setting Up Pi-hole for Local DNS and Ad-Blocking

This guide will walk you through setting up Pi-hole, a network-wide ad blocker that also functions as an excellent local DNS server for your home network.

**What you'll need:**

- **A dedicated device:** A Raspberry Pi (any model, even older ones work well), an old PC, or a virtual machine running a Debian-based Linux distribution (like Ubuntu Server or Raspberry Pi OS Lite). This device needs to be always on.
- **Static IP Address:** The device running Pi-hole needs a static IP address on your local network so other devices can always find it.
- **Ethernet connection (recommended):** For stability, connect your Pi-hole device via Ethernet if possible.
- **SSH Client:** (e.g., PuTTY for Windows, Terminal for macOS/Linux) to connect to your Pi-hole device.

* * *

### Step 1: Prepare Your Pi-hole Device

1.  **Install Operating System:** If you haven't already, install your chosen Linux distribution on your device. For Raspberry Pi, use Raspberry Pi Imager to flash Raspberry Pi OS Lite (64-bit recommended) to an SD card.
2.  **Connect and Boot:** Connect your device to your network via Ethernet and power it on.
3.  **Find its IP Address:**
    - Check your router's connected devices list.
    - Use a network scanner app (e.g., Fing on mobile).
    - On Linux, after logging in, type `ip a` or `ifconfig` (if installed) to find the IP address of your Ethernet interface (e.g., `eth0`).
4.  **SSH into the Device:**
    - Open your SSH client.
    - Connect using the username (e.g., `pi` for Raspberry Pi OS) and the IP address: `ssh pi@your_pi_ip_address`
    - Enter the default password (e.g., `raspberry` for Raspberry Pi OS, you'll be prompted to change it).
5.  **Update Your System:**
    - Once logged in, run:
        
        ```bash
        sudo apt update
        sudo apt upgrade -y
        ```
        
    - Reboot if prompted: `sudo reboot` (then reconnect via SSH).
        
6.  **Set a Static IP Address for Pi-hole:**
    - **Router DHCP Reservation (Recommended):** This is generally easier. Log into your router's administration interface, find the DHCP settings, and create a "DHCP Reservation" or "Static Lease" for your Pi-hole device's MAC address, assigning it a specific IP address (e.g., `192.168.1.50`). This way, your router always gives your Pi-hole the same IP.
    - **Manual Static IP (Advanced):** If your router doesn't support reservations or you prefer to set it on the device, you'll need to edit network configuration files (e.g., `/etc/dhcpcd.conf` on Raspberry Pi OS). Consult specific guides for your Linux distribution.

* * *

### Step 2: Install Pi-hole

1.  **Run the Installer:** From your SSH session, execute the Pi-hole automated installer:
    
    ```bash
    curl -sSL https://install.pi-hole.net | bash
    ```
    
2.  **Follow the On-Screen Prompts:**
    
    - The installer will guide you through the setup. Press `Enter` to proceed through the welcome screens.
    - **Static IP Address:** It will detect your current IP. Confirm it's the static IP you want to use for Pi-hole.
    - **Upstream DNS Provider:** Choose an upstream DNS provider (e.g., Cloudflare, Google, OpenDNS). Pi-hole will forward queries it can't answer (like external websites) to this provider.
    - **Blocklists:** Select the default blocklists. You can add more later.
    - **Admin Web Interface:** Ensure you select to install the web admin interface.
    - **Logging:** Choose your preferred logging options.
    - **Installation Complete:** Note down the IP address of your Pi-hole and the password for the web interface.

* * *

### Step 3: Configure Your Network to Use Pi-hole

This is the crucial step to make all devices on your network use Pi-hole for DNS.

1.  **Log in to your Router:** Access your router's administration interface (usually `192.168.1.1` or `192.168.0.1`).
2.  **Find DHCP/LAN Settings:** Look for sections related to "DHCP Server," "LAN Setup," or "DNS Settings."
3.  **Change DNS Server:**
    - Set the **Primary DNS Server** to the static IP address of your Pi-hole (e.g., `192.168.1.50`).
    - If there's a Secondary DNS Server field, you can either leave it blank (forcing all DNS through Pi-hole) or set it to a public DNS server (e.g., `1.1.1.1` for Cloudflare) as a fallback, though this can sometimes bypass Pi-hole's ad-blocking. For maximum blocking, leave it blank or set it to your Pi-hole's IP again.
4.  **Save and Apply:** Save the settings and reboot your router if prompted.
5.  **Renew IP Lease on Devices:** For devices already connected, you might need to disconnect and reconnect to Wi-Fi, reboot them, or manually renew their DHCP lease for them to pick up the new DNS settings.

* * *

### Step 4: Configure Local DNS Records in Pi-hole

Now, let's make your custom server names work!

1.  **Access Pi-hole Web Interface:** Open a web browser and go to `http://your_pi_hole_ip_address/admin` (e.g., `http://192.168.1.50/admin`).
2.  **Log in:** Use the password provided during installation.
3.  **Navigate to Local DNS:** In the left sidebar, go to `Local DNS` > `DNS Records`.
4.  **Add Your Records:**
    - Click `Add`.
    - **Domain:** Enter your custom hostname (e.g., `myserver.home`, `plex.home`, `files.home`). You can choose any local domain suffix like `.home`, `.lan`, or `.local`.
    - **IP Address:** Enter the static local IP address of the server or device you want to map (e.g., `192.168.1.100`).
    - Click `Add`.
    - Repeat for all your servers and apps.

**Example:**

| Domain | IP Address |
| :--- | :--- |
| `myserver.home` | `192.168.1.100` |
| `plex.home` | `192.168.1.101` |
| `nas.home` | `192.168.1.102` |

* * *

### Step 5: Test Your Setup

1.  **On a device connected to your network:**
    - Open a command prompt (Windows) or terminal (macOS/Linux).
    - Try pinging your custom hostname: `ping myserver.home`
    - You should see replies from the correct IP address (`192.168.1.100`).
    - Try accessing your services in a web browser using the custom names: `http://plex.home:32400` (remember, DNS only resolves the IP, you still need the port unless you set up a reverse proxy).
2.  **Check Pi-hole Dashboard:** You should see DNS queries being processed and ads being blocked in the Pi-hole web interface.