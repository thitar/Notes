---
title: Emulating DynDNS with OVH using a Script
updated: 2025-07-25 18:09:09Z
created: 2025-07-25 18:06:54Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

# Emulating DynDNS with OVH using a Script

This guide explains how to automatically update your domain's public IP address registered with OVH, similar to how a DynDNS service works. This is useful if your Internet Service Provider (ISP) gives you a dynamic public IP address that changes periodically, and you want to access services on your home network (like a personal web server or VPN) from outside.

## 1\. The Challenge: Dynamic Public IP Addresses

Your home network's public IP address (the one the rest of the internet sees) can change. If it does, your domain name (e.g., `home.yourdomain.com`), which is configured to point to your old IP, will no longer work from outside your network. We'll use a script to automatically update this.

## 2\. Prerequisites

1.  **OVH API Credentials:** You need to generate API keys from your OVH Control Panel.

      * Log in to your [OVH Control Panel](https://www.ovh.com/manager/).
      * Click on your name/avatar in the top right corner and select **"My account"**.
      * Go to **"API access"** (or similar, it might be under "SSH Keys & API" or "API tokens").
      * Click **"Create a new token"** or **"Generate a new API credential"**.
      * **Crucially, grant the following permissions:**
          * `GET /ip` (to get information about your public IP if needed, though `ipify.org` is used in the example)
          * `GET /domain/zone/*` (to read DNS zone records)
          * `PUT /domain/zone/*/record/*` (to update DNS records)
          * `POST /domain/zone/*/refresh` (to apply DNS zone changes)
      * Note down your **Application Key**, **Application Secret**, and **Consumer Key**. **Treat these like passwords; keep them secure.**

2.  **Python 3:** Ensure Python 3 is installed on the device where you'll run the script (e.g., your Pi-Hole Raspberry Pi).

3.  **OVH Python Library:** Install the OVH API client library:

    ```bash
    pip install ovh requests
    ```

## 3\. The DynDNS Update Script

Create a new file (e.g., `ovh_dyndns_updater.py`) on your Pi-Hole or another Linux machine and paste the following Python code.

```python
import ovh
import requests
import time
import sys

# --- OVH API Configuration ---
# IMPORTANT: Replace with your actual credentials from OVH Control Panel!
# DO NOT SHARE THESE.
APPLICATION_KEY = "YOUR_OVH_APPLICATION_KEY"
APPLICATION_SECRET = "YOUR_OVH_APPLICATION_SECRET"
CONSUMER_KEY = "YOUR_OVH_CONSUMER_KEY" # This is generated when you create the token.

# --- Domain Configuration ---
DOMAIN_NAME = "yourdomain.com"          # Your main domain (e.g., example.com)
SUBDOMAIN_TO_UPDATE = "home"            # The specific subdomain to update (e.g., home.yourdomain.com)
                                        # Use "" (empty string) if you want to update the root domain (yourdomain.com)

# --- Initialize OVH client ---
# Choose the correct endpoint for your OVH region (e.g., 'ovh-eu', 'ovh-ca', 'ovh-us')
client = ovh.Client(
    endpoint='ovh-eu',
    application_key=APPLICATION_KEY,
    application_secret=APPLICATION_SECRET,
    consumer_key=CONSUMER_KEY,
)

def get_public_ip():
    """Fetches the current public IP address from a public service."""
    try:
        response = requests.get('https://api.ipify.org?format=json', timeout=10)
        response.raise_for_status() # Raise an HTTPError for bad responses (4xx or 5xx)
        return response.json()['ip']
    except requests.exceptions.RequestException as e:
        print(f"Error getting public IP: {e}", file=sys.stderr)
        return None

def get_ovh_dns_record_info(domain_name, subdomain):
    """Fetches the current A record for the specified subdomain from OVH."""
    zone_name = domain_name
    record_type = "A" # We are updating an A record

    try:
        # Get all A records for the zone, filtered by subdomain
        record_ids = client.get(f'/domain/zone/{zone_name}/record', fieldType=record_type, subDomain=subdomain)

        if not record_ids:
            print(f"No A record found for '{subdomain}.{zone_name}'. Please ensure it exists in OVH DNS.", file=sys.stderr)
            return None, None

        # Assuming you want the first matching record if multiple exist for some reason
        record_id = record_ids[0]
        record = client.get(f'/domain/zone/{zone_name}/record/{record_id}')
        return record['target'], record_id
    except ovh.exceptions.ResourceNotFoundError:
        print(f"Domain zone '{zone_name}' not found or invalid API key/permissions.", file=sys.stderr)
        return None, None
    except ovh.exceptions.APIError as e:
        print(f"OVH API Error fetching record for '{subdomain}.{zone_name}': {e}", file=sys.stderr)
        return None, None

def update_ovh_dns_record(domain_name, record_id, new_ip):
    """Updates the A record with the new IP address via OVH API."""
    zone_name = domain_name
    try:
        client.put(f'/domain/zone/{zone_name}/record/{record_id}', target=new_ip)
        print(f"Successfully updated DNS record ID {record_id} to {new_ip}")
        return True
    except ovh.exceptions.APIError as e:
        print(f"OVH API Error updating record ID {record_id} to {new_ip}: {e}", file=sys.stderr)
        return False

def refresh_ovh_dns_zone(domain_name):
    """Refreshes the DNS zone to apply changes. Crucial for changes to propagate."""
    zone_name = domain_name
    try:
        client.post(f'/domain/zone/{zone_name}/refresh')
        print(f"DNS zone '{zone_name}' refreshed successfully.")
        return True
    except ovh.exceptions.APIError as e:
        print(f"OVH API Error refreshing zone '{zone_name}': {e}", file=sys.stderr)
        return False

def main():
    print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] Starting OVH DynDNS update for {SUBDOMAIN_TO_UPDATE or DOMAIN_NAME}...")

    current_public_ip = get_public_ip()
    if not current_public_ip:
        print("Failed to get current public IP. Aborting.", file=sys.stderr)
        return

    ovh_ip, record_id = get_ovh_dns_record_info(DOMAIN_NAME, SUBDOMAIN_TO_UPDATE)

    if ovh_ip is None or record_id is None:
        print("Could not retrieve current OVH IP or record ID. Aborting.", file=sys.stderr)
        return

    if ovh_ip == current_public_ip:
        print(f"IP address for {SUBDOMAIN_TO_UPDATE or DOMAIN_NAME} is already up to date ({current_public_ip}).")
    else:
        print(f"IP mismatch! Current public IP: {current_public_ip}, OVH IP: {ovh_ip}")
        print(f"Updating DNS record for {SUBDOMAIN_TO_UPDATE or DOMAIN_NAME}...")
        if update_ovh_dns_record(DOMAIN_NAME, record_id, current_public_ip):
            time.sleep(5) # Give OVH a moment before refreshing, often not strictly necessary but good practice
            if refresh_ovh_dns_zone(DOMAIN_NAME):
                print("DNS update process completed successfully.")
            else:
                print("DNS update successful, but zone refresh failed. Changes might take longer.", file=sys.stderr)
        else:
            print("Failed to update DNS record.", file=sys.stderr)
    print("------------------------------------------------------------------")

if __name__ == "__main__":
    main()
```

## 4\. How to Use the Script

1.  **Save the Script:** Save the code above to a file named `ovh_dyndns_updater.py` (or any other `.py` name) on your Raspberry Pi. A good location might be `/home/pi/scripts/ovh_dyndns_updater.py`.

2.  **Make it Executable (Optional but Good Practice):**

    ```bash
    chmod +x /home/pi/scripts/ovh_dyndns_updater.py
    ```

3.  **Test Manually:**

    ```bash
    python3 /home/pi/scripts/ovh_dyndns_updater.py
    ```

    Check the output for any errors. The first time you run it, if your public IP differs from what's currently set at OVH, it should update.

4.  **Schedule with `cron` (for automatic updates):**
    `cron` is a job scheduler on Linux. We'll use it to run your script regularly.

      * Open your cron table:

        ```bash
        crontab -e
        ```

        (If asked, choose a text editor like `nano`).

      * Add the following line at the end of the file to run the script every 10 minutes:

        ```cron
        */10 * * * * /usr/bin/python3 /home/pi/scripts/ovh_dyndns_updater.py >> /var/log/ovh_dyndns.log 2>&1
        ```

          * `*/10 * * * *`: This means "run every 10 minutes."
          * `/usr/bin/python3`: This is the full path to your Python 3 interpreter. You can find it with `which python3`.
          * `/home/pi/scripts/ovh_dyndns_updater.py`: This is the full path to your script.
          * `>> /var/log/ovh_dyndns.log 2>&1`: This redirects all output (standard output and errors) to a log file, which is very useful for troubleshooting.

      * Save and exit the cron editor (Ctrl+X, Y, Enter for nano).

## 5\. Important Considerations for DynDNS:

  * **Firewall/Port Forwarding:** Even after your domain points to your correct public IP, you'll still need to configure your router's firewall and set up **port forwarding** to allow external traffic to reach specific services on your internal network (e.g., port 80 for a web server, port 443 for HTTPS, a custom port for a VPN).
  * **Security:** Exposing services to the internet carries security risks. Ensure any services you expose are secure and up-to-date. Consider using a VPN for remote access to your home network instead of directly exposing services.
  * **Log File:** Regularly check the log file (`/var/log/ovh_dyndns.log`) for any errors or messages to ensure your script is running correctly.
  * **OVH Rate Limits:** Be mindful of OVH API rate limits. Running the script every 5-10 minutes is usually fine, but don't run it excessively.
  * **Initial A Record:** Ensure you have an A record created for `home.yourdomain.com` (or whatever subdomain you choose) in your OVH DNS zone *before* running the script. The script won't create it, only update it. You can point it to a dummy IP (e.g., `127.0.0.1`) initially.