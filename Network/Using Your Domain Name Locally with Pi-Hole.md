---
title: Using Your Domain Name Locally with Pi-Hole
updated: 2025-07-25 18:06:32Z
created: 2025-07-25 18:06:15Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

# Using Your Domain Name Locally with Pi-Hole

This guide explains how to configure your Pi-Hole to resolve your domain name (e.g., `yourdomain.com`) to a local IP address within your home network. This setup, known as "Split-Horizon DNS," allows you to access local services using your domain name without interfering with public services like your OVH email.

## 1\. Understanding Split-Horizon DNS

- **Public DNS (OVH):** When someone from the internet accesses your domain (e.g., for email or your public website), OVH's DNS servers provide the correct public IP addresses. This remains unchanged.
    
- **Local DNS (Pi-Hole):** When devices *on your home network* access your domain, your Pi-Hole will provide a *local* IP address (e.g., the IP address of a server or device on your network).
    

## 2\. Configure Your Devices to Use Pi-Hole as DNS

For Pi-Hole to manage your local DNS, your network devices must be configured to use it as their primary DNS server.

### Option A: Router Configuration (Recommended)

This is the most efficient method as it applies to all devices connected to your network.

1.  **Access Router Admin Interface:** Open a web browser and navigate to your router's IP address (e.g., `192.168.1.1`, `192.168.0.1`, `10.0.0.1`). Refer to your router's manual for the exact address and login credentials.
    
2.  **Locate DNS Settings:** Look for sections like "LAN Settings," "DHCP Settings," "Network Settings," or "DNS Servers."
    
3.  **Enter Pi-Hole's IP:** In the primary DNS server field, enter the static IP address of your Pi-Hole (e.g., `192.168.1.10`).
    
4.  **Save Changes:** Apply or save the changes and reboot your router if prompted.
    
5.  **Verify:** After your router reboots, ensure your devices are receiving the Pi-Hole's IP as their DNS server. You can check this in your device's network settings.
    

### Option B: Individual Device Configuration (Less Common for Home Networks)

You can manually set the DNS server on specific devices.

- **Windows:** Network and Sharing Center > Change adapter settings > Right-click network adapter > Properties > Internet Protocol Version 4 (TCP/IPv4) > Properties > Use the following DNS server addresses.
    
- **macOS:** System Settings > Network > Wi-Fi/Ethernet > Details > DNS.
    
- **Linux:** Often configured in `/etc/resolv.conf` or via network manager.
    
- **Mobile Devices:** Check Wi-Fi network settings.
    

## 3\. Add Local DNS Records in Pi-Hole

Now, tell Pi-Hole how to resolve your domain locally.

1.  **Access Pi-Hole Admin Interface:** Open a web browser and go to `http://pi.hole/admin` or `http://[your_pi_hole_ip_address]/admin`.
    
2.  **Navigate to Local DNS Records:** In the left-hand menu, go to **"Local DNS"** > **"DNS Records"**.
    
3.  **Add a New Record:** Click on **"Add A record"** or **"Add CNAME record"**.
    
    - **A Record (Address Record):** Maps a hostname directly to an IP address. Use this for the primary mapping.
        
        - **Domain:** Enter your full domain or a subdomain (e.g., `yourdomain.com`, `home.yourdomain.com`).
            
        - **IP Address:** Enter the *local* IP address of the device you want this domain to point to (e.g., `192.168.1.50` for a local web server, or your router's IP if you want to access its interface).
            
        - **Example:**
            
            - Domain: `yourdomain.com`
                
            - IP Address: `192.168.1.50` (your local web server)
                
            - Domain: `files.yourdomain.com`
                
            - IP Address: `192.168.1.60` (your local NAS)
                
    - **CNAME Record (Canonical Name Record):** Maps one hostname to another hostname. Useful for aliases.
        
        - **Domain:** Enter the alias you want to create (e.g., `www.yourdomain.com`).
            
        - **Target Domain:** Enter the domain it should point to (e.g., `yourdomain.com`).
            
        - **Example:**
            
            - Domain: `www.yourdomain.com`
                
            - Target Domain: `yourdomain.com` (so `www.yourdomain.com` also goes to `192.168.1.50`)
                
4.  **Save/Add:** Click "Add" (or similar button) to save the record.
    

## 4\. Test Your Local Domain Resolution

From a device on your network that's using Pi-Hole for DNS:

1.  Open a web browser.
    
2.  Type in your domain name (e.g., `http://yourdomain.com` or `http://files.yourdomain.com`).
    
3.  It should now direct you to the local IP address and service you configured in Pi-Hole, not any public website or service associated with your domain via OVH.
    

## Important Notes:

- **Local IP Addresses:** Always use private, local IP addresses (e.g., `192.168.x.x`, `10.x.x.x`, `172.16.x.x` to `172.31.x.x`) in Pi-Hole for local resolution.
    
- **No Conflict:** This local setup does not interfere with your OVH-managed email or any public web presence. External users will still resolve your domain through OVH's DNS servers.
    
- **Pi-Hole Availability:** If your Pi-Hole is down, your local domain resolution will fail, and your devices might fall back to other configured DNS servers (if any). Ensure your Pi-Hole is always running.
    

* * *