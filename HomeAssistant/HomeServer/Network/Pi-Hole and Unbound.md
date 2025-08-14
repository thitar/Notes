---
title: Pi-Hole and Unbound
updated: 2025-07-20 16:21:49Z
created: 2025-07-20 16:21:29Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

Pi-Hole and Unbound

This guide will walk you through installing Unbound and then configuring your existing Pi-hole to use it as its sole upstream DNS resolver.

**What you'll need:**

- **Your Raspberry Pi 2:** With Raspberry Pi OS (formerly Raspbian) Lite installed and Pi-hole already set up and running.
    
- **Static IP Address for Pi-hole:** (e.g., `192.168.1.50`)
    
- **SSH Client:** (e.g., PuTTY for Windows, Terminal for macOS/Linux) to connect to your Pi 2.
    
- **A Good Power Supply:** Essential for stability, especially for 24/7 operation.
    

* * *

## Guide: Configuring Pi-hole with Unbound on Raspberry Pi 2

This guide assumes you have a working Pi-hole installation on your Raspberry Pi 2.

* * *

### Step 1: Install Unbound

1.  **SSH into your Raspberry Pi 2:**
    
    Bash
    
    ```
    ssh pi@your_pi_ip_address
    ```
    
    (Replace `your_pi_ip_address` with your Pi-hole's static IP. The default username is `pi`).
    
2.  **Update your system's package list:**
    
    Bash
    
    ```
    sudo apt update
    ```
    
3.  **Install Unbound:**
    
    Bash
    
    ```
    sudo apt install unbound -y
    ```
    
    This will install Unbound and its dependencies.
    

* * *

### Step 2: Configure Unbound

We'll create a new configuration file for Unbound that's specifically tailored for use with Pi-hole. This keeps your custom settings separate from the default Unbound configuration files, making updates easier.

1.  **Create the Unbound configuration file for Pi-hole:**
    
    Bash
    
    ```
    sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
    ```
    
2.  **Paste the following configuration into the `nano` editor:**
    
    ```
    server:
        # If no logfile is specified, syslog is used
        # logfile: "/var/log/unbound/unbound.log"
        verbosity: 0
    
        port: 5335
        do-ip4: yes
        do-udp: yes
        do-tcp: yes
    
        # May be set to yes if you have IPv6 connectivity
        do-ip6: no
    
        # You want to leave this to no unless you have *native* IPv6.
        # Raspberry Pi OS (formerly Raspbian) does not have native IPv6 enabled by default.
        # If you later enable native IPv6 on your Pi, change this to 'yes'.
    
        # Unbound will not resolve queries for private use networks, i.e. 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, etc.
        private-address: 192.168.0.0/16
        private-address: 169.254.0.0/16
        private-address: 172.16.0.0/12
        private-address: 10.0.0.0/8
        private-address: fd00::/8
        private-address: fe80::/10
    
        # Use the root hints file downloaded by the package manager
        auto-trust-anchor-file: "/var/lib/unbound/root.key"
    
        # Ensure DNSSEC data is validated
        harden-glue: yes
        harden-dnssec-stripped: yes
        harden-referral-path: no # Set to no if you experience issues with local network resolution
    
        # Do not insert authority/additional sections into response messages when
        # those sections are not required. This reduces response size
        # significantly, and may avoid TCP fallback for some responses. This may
        # cause a slight speedup.
        minimal-responses: yes
    
        # Perform prefetching of close to expired message cache entries
        # This only applies to domains that have been frequently queried
        # This flag updates the cached domains
        prefetch: yes
        prefetch-key: yes
    
        # Send minimum amount of information to upstream servers to enhance privacy
        qname-minimisation: yes
    
        # Do not allow recursion for queries coming from outside your local network
        # This is critical for security.
        access-control: 127.0.0.1/32 allow
        access-control: 192.168.0.0/16 allow # Adjust to your local network range! Example: 192.168.1.0/24 or 192.168.0.0/16 for larger networks
    
        # Set the total number of unwanted replies to keep track of in every thread.
        # When it reaches the threshold, a defensive action of clearing the rrset
        # and message caches is taken, hopefully flushing away any poison.
        # Unbound suggests a value of 10 million.
        unwanted-reply-threshold: 10000000
    
        # Number of bytes size to advertise as the EDNS reassembly buffer
        # size. This only affects the UDP side of the DNS protocol.
        # DNS servers can switch from UDP to TCP when a DNS response is too big
        # to fit in this limited buffer size. This value has also been suggested
        # in DNS Flag Day 2020.
        edns-buffer-size: 1472
    
        # Ensure kernel buffer is large enough to not lose messages in traffic spikes
        so-rcvbuf: 1m
        so-sndbuf: 1m
    ```
    
    **Important Adjustments:**
    
    - `port: 5335`: This is the port Unbound will listen on. Pi-hole uses port 53, so we use a different one for Unbound to avoid conflicts.
        
    - `do-ip6: no`: Unless your Pi 2 has native IPv6 internet connectivity and you specifically want Unbound to resolve IPv6 addresses, keep this as `no`. For most home users, `no` is appropriate.
        
    - `access-control: 192.168.0.0/16 allow`: **Change `192.168.0.0/16` to match your actual local network range.** For example:
        
        - If your network is `192.168.1.x`, use `192.168.1.0/24`.
            
        - If your network uses `10.0.0.x`, use `10.0.0.0/8`.
            
        - The `/16` is a CIDR notation for the subnet mask. `192.168.0.0/16` covers `192.168.0.1` through `192.168.255.254`.
            
3.  **Save the file:**
    
    - Press `Ctrl + X`
        
    - Press `Y` to confirm saving.
        
    - Press `Enter` to confirm the filename.
        
4.  Disable unbound-resolvconf.service (Important for Debian/Raspberry Pi OS):
    
    This service can interfere with Unbound's configuration, so it's best to disable it.
    
    Bash
    
    ```
    sudo systemctl disable --now unbound-resolvconf.service
    ```
    
    Bash
    
    ```
    sudo rm /etc/unbound/unbound.conf.d/resolvconf_resolvers.conf
    ```
    
5.  **Check Unbound configuration for errors:**
    
    Bash
    
    ```
    sudo unbound-checkconf
    ```
    
    You should see `unbound-checkconf: no errors in /etc/unbound/unbound.conf.d/pi-hole.conf` (and other files if they exist). If you see errors, carefully re-check your `pi-hole.conf` file for typos.
    
6.  **Restart Unbound service:**
    
    Bash
    
    ```
    sudo systemctl restart unbound
    ```
    
7.  **Check Unbound service status:**
    
    Bash
    
    ```
    sudo systemctl status unbound
    ```
    
    It should show `Active: active (running)`. Press `q` to exit the status view.
    

* * *

### Step 3: Test Unbound (Optional but Recommended)

You can test Unbound directly to ensure it's resolving queries correctly and performing DNSSEC validation.

1.  **Query a DNSSEC-enabled domain:**
    
    Bash
    
    ```
    dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335
    ```
    
    In the output, look for a line like `flags: qr rd ra ad;` (the `ad` flag stands for "authenticated data," indicating DNSSEC validation passed). Also, check the `status: NOERROR`.
    
2.  **Query a DNSSEC-bogus domain (should fail):**
    
    Bash
    
    ```
    dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335
    ```
    
    This query should return `status: SERVFAIL` and the `ad` flag should be absent, indicating that Unbound correctly identified the bogus (tampered) record.
    
3.  **Query a regular domain:**
    
    Bash
    
    ```
    dig pi-hole.net @127.0.0.1 -p 5335
    ```
    
    This should return the IP address for `pi-hole.net`. The first query for a domain will be slower (as Unbound recursively resolves it), but subsequent queries should be much faster due to caching.
    

* * *

### Step 4: Configure Pi-hole to Use Unbound

Now, tell Pi-hole to use your local Unbound instance as its upstream DNS server.

1.  **Access Pi-hole Web Interface:** Open a web browser and go to `http://your_pi_hole_ip_address/admin` (e.g., `http://192.168.1.50/admin`).
    
2.  **Log in:** Use your Pi-hole admin password.
    
3.  **Go to Settings:** In the left sidebar, click `Settings`.
    
4.  **Go to DNS Tab:**
    
    - **Uncheck all Upstream DNS Servers** in the left column (Google, Cloudflare, OpenDNS, etc.).
        
    - Under Custom 1 (IPv4) in the right column, enter:
        
        127.0.0.1#5335
        
        This tells Pi-hole to send DNS queries to Unbound, which is listening on the loopback address (127.0.0.1) on port 5335.
        
    - **Uncheck "Use DNSSEC"** in Pi-hole. Unbound is handling DNSSEC validation, so Pi-hole doesn't need to do it again (this prevents potential conflicts and ensures Unbound is the sole validator).
        
    - Ensure **"Never forward reverse lookups for private IP ranges"** and **"Never forward non-FQDNs"** are checked under "Advanced DNS settings." These are typically default and recommended.
        
    - Scroll down and click the **Save** button.
        

### Step 5: Test Your Combined Setup

1.  **Clear your device's DNS cache:**
    
    - **Windows:** Open Command Prompt as Administrator and type `ipconfig /flushdns`
        
    - **macOS:** Open Terminal and type `sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder`
        
    - **Linux:** Depends on your distribution/resolver. Often `sudo systemctl restart systemd-resolved` or similar.
        
    - **For mobile devices/routers:** A quick way is to restart Wi-Fi on the device, or reboot the device/router.
        
2.  **Browse the internet from a device using Pi-hole for DNS:**
    
    - Open a web browser and visit a few common websites.
        
    - Try visiting websites that typically have many ads (e.g., news sites, blogs). Pi-hole should still be blocking ads.
        
3.  **Check Pi-hole's Query Log:**
    
    - Go back to your Pi-hole web interface (`http://your_pi_hole_ip_address/admin`).
        
    - Go to `Query Log`.
        
    - You should now see queries being answered by `localhost#5335` (or `127.0.0.1#5335`) in the "Reply" column, indicating that Unbound is successfully processing the requests.
        
4.  **Verify recursion:** For a deeper dive, you can use a command like `dig pi-hole.net` from a client device (that's using Pi-hole as its DNS). In the output, you won't directly see Unbound's recursive steps, but the overall speed and the presence of the `ad` flag (authenticated data) from `dig +dnssec` on certain queries will confirm DNSSEC validation is working through Unbound.
    

Congratulations! Your Raspberry Pi 2 is now running a powerful, private, and secure DNS resolver stack with Pi-hole for ad-blocking and Unbound for recursive, validating lookups. This significantly enhances your network's privacy and security.