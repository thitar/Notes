---
title: Migrating Paperless-ngx from LXC to Docker VM with iGPU Passthrough
updated: 2025-08-10 19:53:13Z
created: 2025-08-01 22:17:56Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

&nbsp;

# Migrating Paperless-ngx from LXC to Docker VM with iGPU Passthrough

This guide provides a thorough, step-by-step process for moving your existing Paperless-ngx LXC container to your ‘docker’ VM, enabling both Paperless-ngx (via its OCR pipeline) and PaddleOCR to leverage your Intel N100’s integrated GPU (iGPU) for accelerated performance.

**Prerequisites:**

- A running Proxmox VE host with an Intel N100 processor.
    
- Your existing Paperless-ngx LXC container.
    
- Your existing ‘docker’ VM with Docker installed and running.
    
- SSH access to your Proxmox host, your Paperless-ngx LXC, and your ‘docker’ VM.
    
- Sufficient free disk space on your ‘docker’ VM for Paperless-ngx data and Docker images.
    
- **Crucially, understand that the Intel N100 has only one iGPU.** Passing it through to your ‘docker’ VM means the Proxmox host itself will no longer use it for display output, and no other VMs/LXCs will have direct iGPU access.
    

## Step 1: Backup Your Current Paperless-ngx LXC Data

This is the most critical step. Data loss is not an option.

1.  **Stop your Paperless-ngx services in the LXC:**
    
    - SSH into your Paperless-ngx LXC.
        
    - Navigate to your Paperless-ngx `docker-compose.yml` directory (e.g., `/opt/paperless-ngx`).
        
    - Stop the running Paperless-ngx containers:
        
        Bash
        
        ```
        docker compose down
        ```
        
    - Verify services are down: `docker ps` (should not show paperless containers).
        
2.  **Perform a full Paperless-ngx data export using `document_exporter`:**
    
    - This tool exports all your documents, their metadata, and the database into a portable format.
        
    - Ensure your Paperless-ngx Docker Compose services are still in the directory.
        
    - Create a temporary directory for the export *outside* your Paperless-ngx data volumes, but accessible by Docker:
        
        Bash
        
        ```
        mkdir ~/paperless_export_temp
        ```
        
    - Run the `document_exporter` command. This will create an `export` directory within `~/paperless_export_temp` containing your data.
        
        Bash
        
        ```
        docker compose run --rm webserver document_exporter /usr/src/paperless/export
        ```
        
        *Note: The `/usr/src/paperless/export` path is the *internal* path within the `webserver` container. Docker Compose will map this to the `~/paperless_export_temp/export` directory on your LXC host.*
        
3.  **Copy the exported data to a safe, external location:**
    
    - Once the `document_exporter` finishes, copy the entire `~/paperless_export_temp/export` directory from your LXC to your Proxmox host or a network share. This is your primary backup.
        
    - Example (from LXC to Proxmox host, assuming LXC ID is 101 and Proxmox user is `youruser`):
        
        Bash
        
        ```
        scp -r ~/paperless_export_temp/export youruser@<Proxmox_Host_IP>:/path/to/safe/backup/location/
        ```
        
4.  **Backup Paperless-ngx Configuration Files:**
    
    - Copy your `docker-compose.yml` and `.env` files from your Paperless-ngx directory (e.g., `/opt/paperless-ngx`) to the same safe location as your exported data.
        
    - Also, copy any custom scripts you were using (e.g., `/opt/paperless-scripts` from the previous guide) to this safe location.
        

## Step 2: Configure Intel iGPU Passthrough to Your ‘docker’ VM

This step dedicates your Intel N100’s iGPU to your ‘docker’ VM.

1.  **Proxmox Host Configuration:**
    
    - **Enable IOMMU and Blacklist iGPU Driver:**
        
        - Edit your GRUB configuration file on the Proxmox host:
            
            Bash
            
            ```
            sudo nano /etc/default/grub
            ```
            
        - Find the line `GRUB_CMDLINE_LINUX_DEFAULT` and add the following parameters inside the quotes. This enables IOMMU and blacklists the Intel graphics driver (`i915`) on the host, dedicating the iGPU to the VM.
            
            ```
            GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt modprobe.blacklist=i915 video=simplefb:off video=vesafb:off video=efifb:off video=vesa:off disable_vga=1"
            ```
            
            *Warning: Adding `video=...` and `disable_vga=1` will cause you to lose console output on your Proxmox host via the iGPU. You will need to access Proxmox via SSH or a separate display adapter if available.*
            
        - Save the file (Ctrl+O, Enter) and exit (Ctrl+X).
            
    - **Update GRUB and `initramfs`:**
        
        Bash
        
        ```
        sudo update-grub
        sudo update-initramfs -u -k all
        ```
        
    - **Reboot your Proxmox host:**
        
        Bash
        
        ```
        sudo reboot
        ```
        
    - **Verify IOMMU is enabled:** After reboot, SSH back into your Proxmox host and check:
        
        Bash
        
        ```
        dmesg | grep -e DMAR -e IOMMU
        ```
        
        You should see a line like `DMAR: IOMMU enabled`.
        
2.  **Configure ‘docker’ VM in Proxmox:**
    
    - Log in to your Proxmox VE web interface.
        
    - Navigate to your ‘docker’ VM (`VM ID` -> `Hardware`).
        
    - Click “Add” -> “PCI Device”.
        
    - From the “PCI Device” dropdown, select your Intel iGPU. It usually appears as `00:02.0 VGA compatible controller`.
        
    - Check “All functions” and “Primary GPU” (if available).
        
    - Ensure “PCI-Express” is checked.
        
    - Click “Add”.
        
    - **Start your ‘docker’ VM.**
        
3.  **Inside Your ‘docker’ VM:**
    
    - SSH into your ‘docker’ VM.
        
    - **Install Intel GPU Drivers and Tools:**
        
        Bash
        
        ```
        sudo apt update
        sudo apt install -y intel-gpu-tools intel-media-va-driver-non-free vainfo
        ```
        
    - **Verify iGPU Detection:**
        
        Bash
        
        ```
        vainfo
        ```
        
        You should see detailed VA-API information, confirming the iGPU is recognized and accessible within the VM.
        
    - **Add your user to `video` and `render` groups:** (Replace `your_username` with your actual username on the ‘docker’ VM)
        
        Bash
        
        ```
        sudo usermod -aG video,render your_username
        ```
        
        *Log out and back in to your ‘docker’ VM for group changes to take effect.*
        

## Step 3: Install Paperless-ngx on Your ‘docker’ VM

Now, you’ll set up a fresh Paperless-ngx installation on your ‘docker’ VM.

1.  **Create a dedicated directory** for Paperless-ngx on your ‘docker’ VM (e.g., `/opt/paperless-ngx`):
    
    Bash
    
    ```
    sudo mkdir -p /opt/paperless-ngx
    cd /opt/paperless-ngx
    ```
    
2.  **Download Paperless-ngx Docker Compose files:**
    
    - It’s recommended to use PostgreSQL as the database backend for new installations.
        
    - Download the `docker-compose.postgres.yml` and `docker-compose.env` files:
        
        Bash
        
        ```
        wget https://raw.githubusercontent.com/paperless-ngx/paperless-ngx/main/docker/compose/docker-compose.postgres.yml -O docker-compose.yml
        wget https://raw.githubusercontent.com/paperless-ngx/paperless-ngx/main/docker/compose/docker-compose.env -O docker-compose.env
        ```
        
3.  **Adjust `docker-compose.yml` and `docker-compose.env`:**
    
    - **Edit `docker-compose.yml`:**
        
        Bash
        
        ```
        nano docker-compose.yml
        ```
        
        - **Volumes:** Modify the `volumes` section for `webserver`, `consumer`, `db`, and `redis` services to point to persistent paths on your ‘docker’ VM. Using relative paths (`./data`, `./media`, etc.) is common.
            
            YAML
            
            ```
            services:
              webserver:
                volumes:
                  -./data:/usr/src/paperless/data
                  -./media:/usr/src/paperless/media
                  -./export:/usr/src/paperless/export # For future exports/imports
                  -./scripts:/usr/src/paperless/scripts:ro # For your pre-consume script
                # Add this section to pass iGPU to the webserver container
                devices:
                  - /dev/dri:/dev/dri
              consumer:
                volumes:
                  -./data:/usr/src/paperless/data
                  -./media:/usr/src/paperless/media
                  -./consume:/usr/src/paperless/consume # Your consumption directory
                  -./scripts:/usr/src/paperless/scripts:ro
                # Add this section to pass iGPU to the consumer container
                devices:
                  - /dev/dri:/dev/dri
              #... other services (db, redis)...
            ```
            
            *Ensure the local paths (`./data`, `./media`, etc.) exist or will be created by Docker Compose.*
            
        - **Ports:** Adjust the `ports` mapping for the `webserver` service if `8000:8000` conflicts with other services on your ‘docker’ VM. (e.g., `8080:8000`)
            
    - **Edit `docker-compose.env`:**
        
        Bash
        
        ```
        nano docker-compose.env
        ```
        
        - **User Mapping:** Set `USERMAP_UID` and `USERMAP_GID` to your user’s UID and GID on the ‘docker’ VM to ensure correct file permissions. You can find these with `id -u your_username` and `id -g your_username`. <sup>2</sup>
            
        - **Paths:** Verify or set the following paths (these are internal to the container, matching the `volumes` in `docker-compose.yml`): <sup>2</sup>
            
            ```
            PAPERLESS_CONSUMPTION_DIR=/usr/src/paperless/consume
            PAPERLESS_DATA_DIR=/usr/src/paperless/data
            PAPERLESS_MEDIA_ROOT=/usr/src/paperless/media
            ```
            
        - **Database Credentials:** If using PostgreSQL, ensure `PAPERLESS_DBHOST`, `PAPERLESS_DBNAME`, `PAPERLESS_DBUSER`, `PAPERLESS_DBPASS` are correctly configured. <sup>2</sup>
            
        - **Pre-Consumption Script:** Add the environment variable for your pre-consumption script. The path must be the *internal* path within the container: <sup>2</sup>
            
            ```
            PAPERLESS_PRE_CONSUME_SCRIPT=/usr/src/paperless/scripts/pre-consume-paddleocr.py
            ```
            
        - **Disable Internal OCR:** To rely solely on PaddleOCR for pre-processing, disable Paperless-ngx’s built-in OCR. <sup>2</sup>
            
            ```
            PAPERLESS_OCR_MODE=skip
            PAPERLESS_OCR_SKIP_ARCHIVE_FILE=true
            ```
            
            *Note: `PAPERLESS_OCR_MODE=skip` tells Paperless-ngx to skip OCR if a text layer is already present. `PAPERLESS_OCR_SKIP_ARCHIVE_FILE=true` ensures it doesn’t try to OCR the PDF/A version it creates.*
            
4.  **Initial Paperless-ngx Startup:**
    
    - From the `/opt/paperless-ngx` directory on your ‘docker’ VM:
        
        Bash
        
        ```
        docker compose up -d
        ```
        
    - This will pull the Docker images, create the initial directory structure, and set up the database.
        

## Step 4: Restore Paperless-ngx Data and Reapply Configuration

Now, you’ll bring your old data into the new installation.

1.  **Stop the new Paperless-ngx installation:**
    
    Bash
    
    ```
    docker compose down
    ```
    
2.  **Copy Backed-up Data to New Volumes:**
    
    - Copy the contents of your backed-up `data` directory (from Step 1) to the new `data` volume location on your ‘docker’ VM (e.g., `/opt/paperless-ngx/data`).
        
    - Copy the contents of your backed-up `media` directory (from Step 1) to the new `media` volume location on your ‘docker’ VM (e.g., `/opt/paperless-ngx/media`).
        
    - Copy the exported documents from `document_exporter` (the `export` directory you created in Step 1) to the `consume` directory of your new Paperless-ngx instance (e.g., `/opt/paperless-ngx/consume`). Paperless-ngx will re-consume these, but since they already have a text layer from the export, it will be fast. <sup>3</sup>
        
3.  **Restore Database:**
    
    - Start the Paperless-ngx containers again:
        
        Bash
        
        ```
        docker compose up -d
        ```
        
    - Access the shell of the `webserver` container:
        
        Bash
        
        ```
        docker compose exec webserver bash
        ```
        
    - Run the `document_importer` command. The path `/usr/src/paperless/export` is the path *inside the container* where you copied your exported data (if you copied it to `/opt/paperless-ngx/export` on the host and mounted it to `/usr/src/paperless/export` in the container, this path is correct).
        
        Bash
        
        ```
        document_importer /usr/src/paperless/export
        ```
        
    - Exit the container shell: `exit`.
        
4.  **Re-index Documents:** After migration, it’s crucial to re-index the search index for optimal performance:
    
    Bash
    
    ```
    docker compose run --rm webserver document_index reindex
    ```
    
    This process can take some time depending on the number of documents.
    
5.  **Clear Browser Cache:** After the migration, clear your web browser’s cache to ensure the Paperless-ngx UI loads correctly, as old redirects might persist.
    

Your Paperless-ngx instance is now migrated to your ‘docker’ VM, ready for the PaddleOCR integration.