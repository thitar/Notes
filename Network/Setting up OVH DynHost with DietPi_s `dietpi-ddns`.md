---
title: Setting up OVH DynHost with DietPi's `dietpi-ddns`
updated: 2025-07-25 18:11:14Z
created: 2025-07-25 18:11:03Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

# Setting up OVH DynHost with DietPi's `dietpi-ddns`

This guide will walk you through configuring Dynamic DNS (DynDNS) for your OVH domain using DietPi's user-friendly `dietpi-ddns` utility. This is the recommended method for DietPi users as it simplifies the process significantly compared to manual scripting.

You will achieve the following:

  * Your chosen subdomain (e.g., `home.yourdomain.com`) will automatically update its public IP address with OVH whenever your home network's public IP changes.
  * You can then use this subdomain to access services on your home network from outside (provided you also configure port forwarding on your router).

## Part 1: Configure DynHost in OVHcloud Control Panel (Mandatory First Step)

Before touching DietPi, you must prepare your domain within the OVHcloud Control Panel by creating a DynHost user and an associated DNS record.

1.  **Log in to your OVHcloud Control Panel:**

      * Go to [https://www.ovhcloud.com/manager/](https://www.ovhcloud.com/manager/)
      * Enter your credentials.

2.  **Navigate to Your DNS Zone:**

      * In the left-hand menu, under "Web Cloud," click on **"DNS zones"**.
      * Select the **domain name** (e.g., `yourdomain.com`) that you want to configure for DynDNS.

3.  **Access the DynHost Tab:**

      * Once your domain's DNS zone is displayed, click on the **"DynHost"** tab.

4.  **Create a DynHost Identifier (User):**

      * This user will be authorized to update the specific DynHost record.
      * Click the **"Manage access"** button (or similar).
      * Click **"Create a username"** (or "+ Add an identifier").
      * Fill in the details:
          * **Description:** A descriptive name (e.g., `DietPi-DDNS`, `Home-Network-DDNS`).
          * **Username suffix:** This will be appended to your domain name to form the full DynHost username. For example, if your domain is `yourdomain.com` and you enter `pihole`, your full username will be `yourdomain.com-pihole`.
          * **Sub-domain:** Specify the *exact subdomain* this user is allowed to update (e.g., `home`). If you want to update the root domain (e.g., `yourdomain.com` directly), you might enter `*` or leave it blank, but it's generally recommended to use a specific subdomain for DDNS.
          * **Password:** Create a strong, unique password for this DynHost user. **SAVE THIS PASSWORD and the full username; you'll need it for DietPi.**
      * Click **"Create"** or **"Validate"**.

5.  **Create a DynHost Record:**

      * After creating the user, return to the **"DynHost"** tab.
      * Click **"Add a DynHost"** (or "+ Add a DynHost entry").
      * Fill in the details:
          * **Subdomain:** Enter the *exact same subdomain* you specified when creating the DynHost user (e.g., `home`).
          * **Target:** Enter your *current public IP address*. You can find this by visiting a website like [https://whatismyip.com/](https://whatismyip.com/). This is just an initial placeholder; DietPi will automatically update it.
      * Click **"Add"** or **"Validate"**.

    *Self-Check:* You should now see an entry under "DynHost" for your chosen subdomain (e.g., `home.yourdomain.com`) with the initial IP address you provided.

## Part 2: Configure DietPi's `dietpi-ddns`

Now that your OVH DynHost is ready, you can configure your DietPi system.

1.  **Connect to your DietPi:**

      * Open an SSH client (like PuTTY on Windows, or use Terminal on macOS/Linux).
      * Connect to your DietPi:
        ```bash
        ssh dietpi@your_pi_hole_ip_address
        ```
      * Enter your DietPi password when prompted.

2.  **Start `dietpi-ddns`:**

      * Once logged in, run the DietPi DDNS configuration tool:
        ```bash
        dietpi-ddns
        ```

3.  **Navigate the Interactive Menu:**

      * You will see an interactive menu.
      * Select the option to **"Add/Configure a new provider"** or **"DDNS Control"** (the exact wording may vary slightly depending on your DietPi version).
      * When prompted to select a DDNS provider, choose **"OVH"**.

4.  **Enter OVH DynHost Credentials:**

      * The tool will then prompt you for the specific details you set up in Part 1:
          * **Domain:** Enter your full domain name (e.g., `yourdomain.com`).
          * **Subdomain:** Enter the exact subdomain you created for DynHost (e.g., `home`).
          * **Username:** Enter the *full DynHost username* you created in the OVH control panel (e.g., `yourdomain.com-pihole`).
          * **Password:** Enter the strong password you set for the DynHost user.
          * **Update Interval:** Select how frequently `dietpi-ddns` should check for IP changes and update OVH (e.g., `5 minutes`, `10 minutes`, `15 minutes`). A 5-10 minute interval is usually sufficient.

5.  **Save and Apply Configuration:**

      * Follow the on-screen prompts to confirm your settings and save the configuration. DietPi will then automatically configure the underlying `ddclient` service and set up a `cron` job to run it at your specified interval.

## Part 3: Verify and Troubleshoot

1.  **Check Status:**

      * After setup, you can check the status of `ddclient` (which `dietpi-ddns` configures) by running:
        ```bash
        sudo systemctl status ddclient
        ```
      * Look for "Active: active (running)" and recent log entries to ensure it's starting without errors.

2.  **Monitor Logs:**

      * You can view `ddclient`'s log for detailed updates:
        ```bash
        sudo tail -f /var/log/syslog | grep ddclient
        ```
      * This command will show live log entries related to `ddclient`. Look for messages indicating successful updates or IP matches.

3.  **Test External Access:**

      * From a device *outside* your home network (e.g., using mobile data, or another Wi-Fi network), try to access the service you intend to expose using your new DynDNS subdomain (e.g., `http://home.yourdomain.com`).
      * Remember, for external access, you will **also need to configure port forwarding on your home router** to direct incoming traffic on specific ports (e.g., 80 for web, 443 for HTTPS, a custom VPN port) to the internal IP address of the device hosting that service.

## Important Notes:

  * **Port Forwarding:** The DynDNS setup only ensures your domain points to your home's public IP. To access services *within* your home network from outside, you *must* set up **port forwarding** on your router.
  * **Security:** Exposing services to the internet carries inherent security risks. Ensure any exposed services are properly secured, use strong passwords, and are regularly updated. Consider using a VPN for more secure remote access to your entire home network.
  * **Initial A Record:** The DynHost setup in OVH requires you to initially create the A record for your subdomain (e.g., `home.yourdomain.com`). `dietpi-ddns` will then keep this record updated. If you try to configure `dietpi-ddns` for a subdomain that doesn't have an A record or isn't set up as a DynHost in OVH, it will likely fail.
  * **Static IP for Pi-Hole:** Ensure your DietPi (running Pi-Hole) has a static IP address within your local network. This is crucial for consistent operation of your Pi-Hole and DDNS client.