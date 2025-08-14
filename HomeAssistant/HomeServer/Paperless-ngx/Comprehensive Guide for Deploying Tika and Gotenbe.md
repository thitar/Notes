---
title: >-
  Comprehensive Guide for Deploying Tika and Gotenberg on a Proxmox Debian 12
  Dock
updated: 2025-07-30 20:00:39Z
created: 2025-07-29 21:00:49Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

# Comprehensive Guide for Deploying Tika and Gotenberg on a Proxmox Debian 12 Docker Host

This report provides a detailed guide for deploying Apache Tika and Gotenberg within a Docker environment on a Debian 12 virtual machine (VM) hosted on Proxmox. It includes a Docker Compose file for the services, along with comprehensive best practices for configuring the Docker VM in Proxmox, hardening the Debian host, optimizing Docker performance, and establishing a robust backup strategy. The aim is to ensure a secure, efficient, and maintainable home server setup for email processing within Paperless-ngx.

## I. Docker Compose Configuration for Tika and Gotenberg

The integration of Apache Tika and Gotenberg with Paperless-ngx significantly enhances its document processing capabilities, particularly for emails and various file formats. Tika is instrumental for content analysis and metadata extraction, while Gotenberg provides essential document conversion services, enabling Paperless-ngx to handle office documents and other formats effectively.<sup>1</sup> These services are external to Paperless-ngx and are largely stateless, meaning they do not store critical, persistent user data themselves. This characteristic simplifies their deployment and backup considerations, as their primary function is to process data on demand rather than retain it long-term.

### A. Docker Compose File for Tika and Gotenberg

The following Docker Compose configuration defines the Tika and Gotenberg services. It is designed for clarity and ease of deployment on a Debian 12 VM.

YAML

```
version: '3.9' # Use a recent stable version for Docker Compose [2]

services:
  tika:
    image: apache/tika:latest # Utilizes the latest stable Tika image
    container_name: paperless_tika
    restart: unless-stopped # Ensures the service restarts automatically if it crashes or on system reboot
    ports:
      # Expose Tika on all network interfaces (0.0.0.0) on host port 9998, mapped to container port 9998.
      # This allows the Paperless-ngx LXC to access the service, as it resides on a separate VM.
      # Binding to 127.0.0.1 (localhost) would restrict access to only services within this VM.
      - "9998:9998"
    # Tika typically does not require specific persistent volumes for its operation beyond internal caching,
    # as it processes data ephemerally.
    # No specific environment variables are usually needed for Tika itself in this setup.

  gotenberg:
    image: thecodingmachine/gotenberg:latest # Utilizes the latest stable Gotenberg image
    container_name: paperless_gotenberg
    restart: unless-stopped # Ensures the service restarts automatically if it crashes or on system reboot
    ports:
      # Expose Gotenberg on all network interfaces (0.0.0.0) on host port 3000, mapped to container port 3000.
      # This allows the Paperless-ngx LXC to access the service, as it resides on a separate VM.
      # Binding to 127.0.0.1 (localhost) would restrict access to only services within this VM.
      - "3000:3000"
    # Gotenberg typically does not require specific persistent volumes for its operation beyond internal caching,
    # as it processes data ephemerally.
    # No specific environment variables are usually needed for Gotenberg itself in this setup.

networks:
  default:
    # By default, Docker Compose creates a bridge network, facilitating communication between 'tika' and 'gotenberg'
    # if they needed to interact (though not for this specific setup).
    # For a home server, direct IP access from the Paperless-ngx LXC to this Docker VM is a common approach.
    # No custom network definition is strictly needed unless specific isolation or advanced routing is required
    # within the Docker host itself.
```

The `ports` configuration is a critical detail for enabling communication between the Paperless-ngx LXC and these new Docker services. Initially, binding ports to `127.0.0.1` (localhost) would restrict access to only processes running *within the same Docker VM*. However, since Paperless-ngx resides in a *separate* Proxmox LXC, it acts as an external host from the perspective of the Docker VM. For the LXC to successfully connect to Tika and Gotenberg, their ports must be exposed on an interface accessible from the LXC's network. Therefore, the configuration uses `"9998:9998"` and `"3000:3000"`, which defaults to binding on `0.0.0.0` (all available network interfaces) on the host VM. This ensures the services are reachable from other machines on the local network, including the Paperless-ngx LXC.

### B. Tika and Gotenberg Service Definitions

The following table summarizes the key aspects of the Tika and Gotenberg service definitions:

| Service Name | Docker Image | Container Name | Host Port | Container Port | Purpose | Key Considerations |
| --- | --- | --- | --- | --- | --- | --- |
| `tika` | `apache/tika:latest` | `paperless_tika` | `9998` | `9998` | Content Analysis, Metadata Extraction | Generally stateless; `restart: unless-stopped` is sufficient. |
| `gotenberg` | `thecodingmachine/gotenberg:latest` | `paperless_gotenberg` | `3000` | `3000` | Document Conversion (e.g., Office to PDF/A) | Generally stateless; `restart: unless-stopped` is sufficient. |

### C. Deployment Instructions

To deploy Tika and Gotenberg using the provided Docker Compose file:

1.  **Save the Compose File:** Save the Docker Compose content provided above as `compose.yaml` (which is the preferred filename) or `docker-compose.yml` in your chosen project directory on the Debian 12 VM (refer to Section III for optimal directory placement).
    
2.  **Navigate to Directory:** Open a terminal and navigate to the directory where you saved `compose.yaml`.
    
3.  **Pull Images:** Execute `docker compose pull` to download the latest Docker images for Tika and Gotenberg.
    
4.  **Start Services:** Run `docker compose up -d` to start the services in detached mode (background).
    
5.  **Verify Status:** Confirm that the containers are running by executing `docker ps`.
    
6.  **Configure Paperless-ngx:** From your Paperless-ngx LXC, configure Paperless-ngx to connect to Tika and Gotenberg. This involves setting the following environment variables in your Paperless-ngx configuration:
    
    - `PAPERLESS_TIKA_ENABLED=true`
        
    - `PAPERLESS_GOTENBERG_ENABLED=true`
        
    - `PAPERLESS_TIKA_URL=http://<Debian_VM_IP>:9998`
        
    - PAPERLESS_GOTENBERG_URL=http://&lt;Debian_VM_IP&gt;:3000
        
        Replace &lt;Debian_VM_IP&gt; with the actual IP address of your Debian 12 Docker VM. These settings enable Paperless-ngx to leverage the external Tika and Gotenberg services for enhanced document processing.1
        

## II. Optimal Docker Compose Project Structure and File Location

Organizing Docker Compose projects systematically is fundamental for long-term maintainability, especially in a home server environment where multiple services might be deployed. A well-structured approach simplifies management, updates, and backup processes.

### A. Recommended Directory Structures for Home Servers

Docker Compose, by default, searches for `compose.yaml` (the preferred name) or `docker-compose.yml` in the current working directory from which the command is executed.<sup>4</sup> While this default is convenient for quick tests, a more structured approach is advisable for persistent home server deployments.

Common organizational strategies include:

- **Service-Based Organization:** This approach dedicates a separate directory for each service or a logical group of related services. It promotes clear separation of concerns, making it easier to manage individual applications. For instance, a structure like `/srv/docker/paperless-tika-gotenberg/` for the Tika and Gotenberg services, containing their `compose.yaml` and any related files, is highly effective.<sup>5</sup>
    
- **Centralized Organization:** This method consolidates all Docker configurations and data into a single, overarching location, which can simplify backup procedures. An example might be `/srv/docker/compose.yaml` with subdirectories for various application data.<sup>5</sup>
    
- **System-Wide Installation:** More typical for production environments, this involves separating configurations and data across standard system directories (e.g., `/etc/docker/containers` for configs, `/srv/data` for persistent data).<sup>5</sup> For home servers,
    
    `/srv` is a widely recommended location for application data, as it is distinct from user home directories (`/home`) and Docker's internal data directory (`/var/lib/docker`).<sup>6</sup>
    

For a home server, a service-based organization within `/srv/docker/` is strongly recommended. This approach offers clear separation of concerns, simplifies the identification of service-related files, and streamlines future backups or migrations of individual services. Adhering to this standard Linux filesystem hierarchy for server applications makes the system more predictable and easier to manage, aligning with more "production-like" practices for long-term stability and ease of operation.

### B. Managing Persistent Data Volumes

Docker volumes are the preferred mechanism for persistent storage over bind mounts for most containerized applications.<sup>8</sup> Volumes are managed by Docker, which simplifies their backup and migration, and they can be safely shared among multiple containers. They ensure that application data persists even if containers are removed or recreated.<sup>8</sup>

Bind mounts, while useful for development or injecting host files (such as configuration files), directly depend on the host's filesystem structure and permissions.<sup>8</sup> This can introduce complexities related to host-side permissions and makes them less portable.

For Tika and Gotenberg, which are largely stateless services, persistent named volumes are not typically required for their core operation. Any internal temporary files or caches generated by these services are usually ephemeral and can reside within the container's writable layer. For performance-critical temporary data, a `tmpfs` mount can be considered to avoid disk I/O.<sup>8</sup> This stateless nature means there is no critical user data

*within* these containers that needs separate, granular backup, simplifying the overall backup strategy for these particular services.

### C. Permissions and Ownership Best Practices

Proper management of permissions and ownership is crucial for Docker deployments to ensure security and prevent unauthorized access or privilege escalation.

For bind mounts, if they are utilized to share data between the host and containers, it is essential to ensure that the host directories have the correct ownership and permissions. This involves using commands like `chown -R <uid>:<gid>` to set ownership and `chmod -R <mode>` to define read, write, or execute permissions, thereby allowing the container's designated user to access the files.<sup>9</sup>

A fundamental security practice is to run container processes as a non-root user.<sup>10</sup> This principle of least privilege significantly limits the potential impact if a container is compromised, as an attacker would not immediately gain root privileges on the host. The user can be specified using the

`--user` flag in `docker run` commands or directly within the Dockerfile.<sup>9</sup>

A significant security consideration, often overlooked, is the implication of adding a user to the `docker` group. While convenient for avoiding `sudo` before every Docker command, this action effectively grants the user root-equivalent privileges on the host system.<sup>12</sup> This is a severe security risk because a user in the

`docker` group can:

- Mount the host's root directory (`/`) inside a container, thereby gaining unrestricted access to the entire host filesystem and the ability to modify any file.<sup>12</sup>
    
- Modify Docker daemon settings, potentially weakening the host's security posture.<sup>13</sup>
    
- Create privileged containers or bypass network isolation, leading to unauthorized network access or control.<sup>13</sup>
    
- Exploit container breakout vulnerabilities, which could allow an attacker to escape the container's isolation and gain root access on the host.<sup>11</sup>
    

Given these substantial risks, the following recommendations are critical for secure Docker management:

- **Limit `docker` group membership:** Only highly trusted administrators who absolutely require it for their daily work should be added to the `docker` group. This access should be treated with the same caution as granting direct root access.
    
- **Enable Rootless Docker Mode:** This is the most robust security alternative.<sup>10</sup> Rootless Docker allows the Docker daemon and containers to run as a non-root user, significantly reducing the risk of privilege escalation even if a vulnerability within Docker or a container is exploited. Although it might be more complex to set up initially and has some limitations, the security benefits for a home server are substantial.
    
- **User Namespace Remapping:** If full rootless mode is not utilized, user namespace remapping should be enabled.<sup>10</sup> This feature maps container UIDs (including the root user inside the container) to an unprivileged range of UIDs on the host system. This means that even if a process within a container manages to gain root privileges, those privileges will not translate to the host's actual root user, providing a strong defense against privilege escalation. This can be configured by adding
    
    `"userns-remap": "default"` to `/etc/docker/daemon.json`.<sup>13</sup>
    
- **Use `sudo` with granular permissions:** As a fallback if rootless mode is not feasible, using `sudo` for Docker commands is a viable alternative. This approach provides an audit trail of Docker commands and allows for more granular control over which specific Docker subcommands a user is permitted to run, enhancing accountability and security.<sup>13</sup>
    

## III. Proxmox VM Configuration for a Docker Host

Optimizing the Proxmox VM settings from its creation is crucial for ensuring optimal performance and stability for a Debian 12 Docker host. These configurations directly influence how the Docker environment interacts with the underlying Proxmox hypervisor and physical hardware.

### A. Essential VM Creation Settings

When creating the Debian 12 VM in Proxmox, consider the following settings:

- **General:** Assign a clear and descriptive name, such as `docker-host-debian12`, for easy identification.
    
- **OS:** Select the appropriate Debian 12 ISO image for installation.
    
- **System:** Enable the "Qemu Agent" option.<sup>14</sup> This agent is vital for Proxmox to communicate effectively with the guest operating system, enabling features like graceful shutdowns, consistent live backups, and accurate retrieval of guest IP addresses.
    
- **Disk:** Allocate sufficient disk space for your Docker images, containers, and volumes. An initial allocation of 60-100GB is often a good starting point, with the understanding that it can be expanded later if using thin-provisioned storage.
    
- **CPU:** Configure the CPU Type and allocate a suitable number of cores (see Section IV.B for details).
    
- **Memory:** Specify adequate RAM for your Docker workloads. A minimum of 4096 MB (4GB) is recommended for Debian 12 itself, with additional memory allocated based on the expected number and resource demands of your Docker containers.<sup>14</sup>
    
- **Network:** Select the appropriate Proxmox network bridge (e.g., `vmbr0`) to connect the VM to your local network.
    

### B. CPU Type and Core Allocation for Performance

The choice of CPU type in Proxmox VM settings significantly impacts guest performance.

- **CPU Type: `host`:** It is strongly recommended to select `host` as the CPU type.<sup>14</sup> The
    
    `host` type exposes the full capabilities and instruction sets of your physical CPU to the VM. This leads to significantly better performance compared to `kvm64`, which presents a limited set of instructions and can result in much worse CPU performance, particularly for demanding tasks.<sup>15</sup> While
    
    `kvm64` might offer better VM portability across different physical CPUs in a multi-node Proxmox cluster, for a single home server where maximum performance is prioritized, `host` is the superior choice. This decision, however, means that live migration of this VM to another Proxmox host with a different CPU architecture might be problematic without a full shutdown and restart.
    
- **Core Allocation:** Assign a reasonable number of CPU cores based on your host's physical cores and the resource requirements of other running VMs or LXCs. For Tika and Gotenberg, 2-4 cores are typically sufficient for moderate document processing loads, but this should be adjusted based on anticipated usage and the number of other Docker services planned for the VM.
    

### C. Disk Controller (VirtIO SCSI) and TRIM/Discard Implementation

Optimizing disk I/O is crucial for Docker performance, especially when dealing with frequent read/write operations.

- **Disk Controller: `VirtIO SCSI`:** Utilize `VirtIO SCSI` as the disk controller for the VM.<sup>16</sup> This paravirtualized driver offers superior I/O performance compared to older, emulated IDE or SATA controllers, as it provides a more direct and efficient path to the underlying storage.
    
- **Enable `Discard` (TRIM):** A critical configuration for SSDs and thin-provisioned storage is to enable the "Discard" checkbox for your virtual disk in Proxmox.<sup>16</sup> This setting allows the guest operating system (Debian 12) to send TRIM commands down to the physical storage. This is vital for reclaiming unused blocks on Solid State Drives (SSDs), preventing performance degradation over time, and ensuring that virtual disk images on thin-provisioned storage (such as ZFS, LVM-Thin, or Ceph) do not grow unnecessarily large with deleted data.
    
- **Guest OS Configuration for TRIM:** To complete the TRIM pipeline, ensure that TRIM is also enabled and actively used within the Debian 12 guest OS. For Linux filesystems like ext4, this can be managed by running `fstrim -va` periodically or enabling `fstrim.timer` for automatic weekly execution.<sup>17</sup> The combination of
    
    `VirtIO SCSI` with `Discard` enabled in Proxmox and `fstrim` in the guest OS creates a complete and effective TRIM mechanism. Without all components in place, the benefits of TRIM (SSD longevity, space reclamation) are lost, potentially leading to "disk bloat" in the virtual environment.
    

### D. Network Device (VirtIO) for High Throughput

Network performance is equally important for a Docker host, particularly for pulling images and communication between containers or with external services like Paperless-ngx.

- **Network Model: `VirtIO`:** Select `VirtIO` as the network device model for the VM.<sup>18</sup> Similar to the disk controller, VirtIO provides paravirtualized network drivers that result in significantly faster network transfers and lower latency compared to emulated devices like E1000E or Realtek. This enhanced network performance is critical for efficient communication between your Paperless-ngx LXC and the new Docker VM, as well as for the rapid pulling of Docker images from registries.

### E. Enabling the Qemu Guest Agent for Enhanced Management

As previously mentioned, ensuring the Qemu Guest Agent is enabled in the VM's System settings is a prerequisite for optimal Proxmox management.<sup>14</sup>

- **Installation:** Install the `qemu-guest-agent` package inside your Debian 12 VM: `sudo apt install qemu-guest-agent -y`.
    
- **Benefits:** The Guest Agent allows Proxmox to perform more consistent backups by leveraging `guest-fsfreeze-freeze` and `guest-fsfreeze-thaw` commands to quiesce the filesystem during live snapshots.<sup>20</sup> This significantly improves data consistency for running VMs. Additionally, it enables Proxmox to retrieve network information from the guest and execute graceful shutdowns, enhancing overall VM manageability. The importance of the Qemu Guest Agent extends directly to backup consistency, highlighting how different configuration choices are interconnected and contribute to the overall reliability of the system.
    

### F. Recommended Proxmox VM Settings for Docker Host

The following table summarizes the recommended Proxmox VM settings for a Debian 12 Docker host:

| Setting Category | Proxmox Setting | Recommended Value/Option | Rationale/Benefit |
| --- | --- | --- | --- |
| **System** | Qemu Agent | Enabled | Enables graceful shutdowns, live backups, guest IP retrieval.<sup>14</sup> |
| **CPU** | Type | `host` | Exposes full physical CPU capabilities for maximum performance.<sup>15</sup> |
|     | Cores | 2-4+ (adjust as needed) | Sufficient for Tika/Gotenberg, scalable for future Docker services. |
| **Disk** | Bus/Device | `VirtIO SCSI` | Superior I/O performance through paravirtualization.<sup>16</sup> |
|     | Discard | Enabled | Allows guest OS to send TRIM commands, reclaiming free space on SSDs/thin-provisioned storage.<sup>16</sup> |
| **Network** | Model | `VirtIO` | High-performance network throughput and lower latency.<sup>18</sup> |

## IV. Debian 12 Docker Host Hardening and Optimization

Securing and optimizing the Debian 12 host running Docker is a multi-layered process, involving system updates, SSH hardening, firewall configuration, Docker daemon security, container best practices, and kernel tuning. Each layer contributes to a more resilient and performant system.

### A. System Updates and Automated Security Patching

Maintaining an up-to-date system is foundational for security. Regular updates patch known vulnerabilities, including critical container escape vulnerabilities that could allow an attacker to gain root access to the host.<sup>11</sup>

To ensure the Debian 12 host remains secure, it is recommended to configure automated security updates using `unattended-upgrades` <sup>21</sup>:

1.  Install Packages: Install unattended-upgrades and apt-listchanges:
    
    sudo apt update && sudo apt install unattended-upgrades apt-listchanges -y.21
    
2.  **Configure Unattended Upgrades:** Edit the configuration file `/etc/apt/apt.conf.d/50unattended-upgrades`. Ensure the line `Unattended-Upgrade::Allowed-Origins { "${distro_id}:${distro_codename}-security"; };` is uncommented to allow automatic updates from the security repository. For systems that can tolerate reboots, consider uncommenting and setting `Unattended-Upgrade::Automatic-Reboot "true";` and `Unattended-Upgrade::Automatic-Reboot-Time "02:00";` to schedule reboots after updates.<sup>21</sup>
    
3.  Enable Daily Runs: Enable the automatic updates feature by editing /etc/apt/apt.conf.d/20auto-upgrades. Add the following lines to ensure the package list is updated and unattended upgrades run daily:
    
    APT::Periodic::Update-Package-Lists "1";
    
    APT::Periodic::Unattended-Upgrade "1";.21
    

### B. SSH Server Hardening

SSH is the primary remote access point to the server, making its security paramount.

1.  **Change the Default SSH Port:** Change the default SSH port from 22 to a non-standard, high-numbered port (e.g., 2222, 7869) in `/etc/ssh/sshd_config`.<sup>23</sup> This measure, often referred to as "security through obscurity," significantly reduces automated bot attacks that target the default port.
    
    - Example: `Port 7869`
2.  **Disable Root Login:** Set `PermitRootLogin no` in `/etc/ssh/sshd_config`.<sup>23</sup> This is a critical security practice, as it prevents direct brute-force attacks on the highly privileged root account. Administrators should always log in as a regular user and then use
    
    `sudo` for elevated privileges.
    
3.  **Implement Key-Based Authentication:** Disable password authentication and enforce SSH key authentication.<sup>23</sup> SSH keys provide a much stronger and more secure authentication method than passwords.
    
    - Generate a strong RSA 4096-bit key pair on your client machine: `ssh-keygen -b 4096 -t rsa`.
        
    - Copy the public key to the server: `ssh-copy-id user@your-server-ip`.
        
    - In `/etc/ssh/sshd_config`, set:
        
        - `PubkeyAuthentication yes`
            
        - `PasswordAuthentication no`
            
4.  **Configure Fail2ban for Brute-Force Protection:** Fail2ban is an intrusion prevention framework that monitors log files for suspicious activity, such as repeated failed SSH login attempts, and dynamically modifies firewall rules to ban malicious IP addresses.<sup>25</sup>
    
    - **Install:** `sudo apt install fail2ban -y`.<sup>25</sup>
        
    - **Copy Default Config:** `sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`.<sup>25</sup>
        
    - **Edit Configuration:** Open `/etc/fail2ban/jail.local` and in the `[sshd]` section, ensure `enabled = true`, `port = ssh` (or your custom SSH port), `filter = sshd`, `logpath = /var/log/auth.log`, and `maxretry = 3`.<sup>25</sup> Adjust
        
        `bantime` (how long an IP is banned) and `findtime` (time window for retries) as desired.<sup>26</sup>
        
    - **Restart Fail2ban:** `sudo systemctl restart fail2ban`.<sup>25</sup>
        

### C. UFW Firewall Management with Docker

Managing the Uncomplicated Firewall (UFW) on a Docker host requires careful configuration due to Docker's direct interaction with `iptables`. Docker manipulates `iptables` rules to manage network traffic for containers, and when container ports are exposed, Docker adds rules that bypass UFW's rules.<sup>27</sup> This means UFW alone cannot fully protect Docker-exposed ports unless specifically configured to integrate with Docker's

`iptables` chains. Docker is compatible with `iptables-nft` and `iptables-legacy`.<sup>27</sup>

The goal is to allow UFW to manage Docker-exposed ports without disabling Docker's `iptables` management, which can break container outbound connectivity.

1.  **Initial UFW Setup:**
    
    - **Install UFW:** `sudo apt install ufw -y`.<sup>28</sup>
        
    - Set Default Policies: It is generally a good security practice to block all incoming connections by default and allow all outgoing connections:
        
        sudo ufw default deny incoming
        
        sudo ufw default allow outgoing.28
        
    - Allow Custom SSH Port: Before enabling UFW, ensure you allow access to your custom SSH port to prevent locking yourself out:
        
        sudo ufw allow &lt;your_ssh_port&gt;/tcp.23
        
    - **Enable UFW:** `sudo ufw enable`.<sup>28</sup>
        
2.  **Integrating UFW with Docker:** This is the critical step to ensure UFW controls Docker-exposed ports.<sup>29</sup>
    
    - **Modify `/etc/ufw/after.rules`:** Edit this file and add the following rules at the end, *before* the `COMMIT` line. These rules allow Docker's internal networks to function while ensuring UFW can manage public access to container ports.
        
        ```
        # BEGIN UFW AND DOCKER [29]
        *filter
        :ufw-user-forward - [0:0]
        :DOCKER-USER - [0:0]
        -A DOCKER-USER -j RETURN -s 10.0.0.0/8
        -A DOCKER-USER -j RETURN -s 172.16.0.0/12
        -A DOCKER-USER -j RETURN -s 192.168.0.0/16
        -A DOCKER-USER -j ufw-user-forward
        -A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 192.168.0.0/16
        -A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 10.0.0.0/8
        -A DOCKER-USER -j DROP -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 172.16.0.0/12
        -A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 192.168.0.0/16
        -A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 10.0.0.0/8
        -A DOCKER-USER -j DROP -p udp -m udp --dport 0:32767 -d 172.16.0.0/12
        -A DOCKER-USER -j RETURN
        COMMIT
        # END UFW AND DOCKER
        ```
        
    - **Restart UFW:** After modifying the `after.rules` file, restart UFW to apply the changes: `sudo systemctl restart ufw`.<sup>29</sup>
        
    - **Allow Specific Docker Ports:** With the above rules in place, you can now use `ufw route allow` to expose specific Docker container ports to the public or local network. For example, to allow external access to Tika (container port 9998) or Gotenberg (container port 3000) if you configured them to bind to `0.0.0.0` or a specific LAN IP in your `docker-compose.yaml`:
        
        - `sudo ufw route allow proto tcp from any to any port 9998`
            
        - `sudo ufw route allow proto tcp from any to any port 3000`
            
        - It is important to note that if you bound these services to `127.0.0.1` in the `docker-compose.yaml` (as in the initial example in Section II), these `ufw route` rules will not make them publicly accessible; they will only be accessible from the Docker host itself. Adjust your compose file's port mapping (`"9998:9998"`) if external access is required.
            

The table below summarizes essential UFW rules for Docker hosts:

| Rule Description | UFW Command | Purpose/Benefit |  
| --- | --- | --- | --- |  
| **Default Policies** | `sudo ufw default deny incoming` | Blocks all incoming connections by default, enhancing security.<sup>28</sup> |  
| | `sudo ufw default allow outgoing` | Allows all outgoing connections, enabling containers to reach external networks.<sup>28</sup> |  
| **SSH Access** | `sudo ufw allow <your_ssh_port>/tcp` | Allows SSH access on your custom port, preventing lockout.<sup>23</sup> |  
| **Tika Access** | `sudo ufw route allow proto tcp from any to any port 9998` | Allows external access to Tika container on port 9998.<sup>29</sup> | *Only if Docker port mapping is `0.0.0.0:9998:9998` or `VM_LAN_IP:9998:9998`.* |  
| **Gotenberg Access** | `sudo ufw route allow proto tcp from any to any port 3000` | Allows external access to Gotenberg container on port 3000.<sup>29</sup> | *Only if Docker port mapping is `0.0.0.0:3000:3000` or `VM_LAN_IP:3000:3000`.* |  
| **Enable UFW** | `sudo ufw enable` | Activates the firewall rules.<sup>28</sup> |  
| **Restart UFW** | `sudo systemctl restart ufw` | Applies changes to UFW configuration, especially after modifying `after.rules`.<sup>29</sup> |

### D. Docker Daemon Security Best Practices

The Docker daemon runs with root privileges by default, making its security a critical component of host hardening.

1.  **Leveraging Rootless Mode and User Namespace Remapping:**
    
    - **Rootless Mode:** This is the strongest recommendation for Docker security.<sup>10</sup> It allows the Docker daemon and containers to run as a non-root user. This significantly reduces the risk that an exploited vulnerability in Docker or a container could grant an attacker root privileges on the host system. While it might introduce some setup complexities and limitations, the security benefits for any server, including a home server, are substantial.
        
    - **User Namespace Remapping (`--userns-remap`):** If full rootless mode is not implemented, enabling user namespace remapping is a vital security measure.<sup>10</sup> This feature maps container UIDs (including the root user inside the container) to an unprivileged range of UIDs on the host. This prevents privilege escalation from within a container to the host's actual root user. It can be enabled by adding
        
        `"userns-remap": "default"` to `/etc/docker/daemon.json`.<sup>13</sup>
        
2.  **Securing the Docker Socket:**
    
    - **Avoid Exposing TCP Daemon Socket:** The Docker daemon socket (`/var/run/docker.sock`) should never be exposed over TCP unless absolutely necessary. If remote access via TCP is unavoidable, it *must* be secured with TLS encryption and strict authentication.<sup>10</sup> Exposing it without proper authentication is equivalent to granting unrestricted root access to your host.
        
    - **Do Not Bind Mount `/var/run/docker.sock` into Containers:** Avoid using commands like `docker run -v /var/run/docker.sock:/var/run/docker.sock`.<sup>11</sup> Mounting the Docker socket into a container grants that container root access to the Docker daemon, effectively giving it control over the entire Docker host.
        
3.  **Keeping Docker Engine Updated:** Regularly update Docker Engine and its associated components (e.g., `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`).<sup>10</sup> Utilizing the official Docker APT repository simplifies this process and ensures access to the latest security patches and bug fixes.<sup>27</sup>
    

### E. Docker Container Security and Performance

Beyond the Docker daemon, individual container configurations play a significant role in overall system security and performance.

1.  **Setting Resource Limits (CPU, Memory):** Implementing resource limits is essential for preventing a single misbehaving or compromised container from consuming all available host resources, which could lead to denial-of-service for other applications or the entire host.<sup>11</sup>
    
    - **Memory:**
        
        - `--memory (-m)`: Sets a hard limit on the amount of RAM a container can use. If this limit is exceeded, the container will be terminated by Docker to protect the host system.<sup>30</sup>
            
        - `--memory-reservation`: Establishes a soft limit. A container may temporarily exceed this limit if the host has sufficient available memory, but it will be throttled back under heavy host memory load.<sup>30</sup>
            
        - `--memory-swap`: Controls the total amount of memory (RAM + swap) a container can use. Setting this value equal to `--memory` effectively disables swap for the container.<sup>30</sup>
            
    - **CPU:**
        
        - `--cpus`: Limits the container to a specific fraction or whole number of CPUs (e.g., `--cpus=1.5` restricts to 1.5 CPUs).<sup>30</sup>
            
        - `--cpu-shares`: Defines relative CPU weight for prioritizing containers when there is CPU contention. A higher value grants a container more CPU time relative to others (default is 1024).<sup>30</sup>
            
        - `--cpu-period` and `--cpu-quota`: Offer precise CPU throttling by setting a time-based quota. For example, `--cpu-period=100000` and `--cpu-quota=50000` limit the container to 50% of one CPU every 100 ms.<sup>30</sup>
            
        - `--cpuset-cpus`: Allows binding a container to specific CPU cores, which can optimize performance for workloads sensitive to CPU cache locality (e.g., `--cpuset-cpus="0,1"`).<sup>30</sup>
            
    - **Best Practices:** It is recommended to test applications to understand their memory and CPU requirements before setting limits in a production environment.<sup>31</sup> Monitoring tools like
        
        `docker stats` can provide real-time usage data. For Tika and Gotenberg, starting with reasonable defaults (e.g., 512MB-1GB memory, 0.5-1 CPU share) and adjusting based on actual document processing load is a practical approach.
        

The table below outlines common Docker container resource limit options:

| Resource Type | Docker Compose/Run Option | Description | Best Practice/Example |
| --- | --- | --- | --- |
| **Memory** | `memory` (`-m`) | Hard limit on RAM usage. Container is killed if exceeded. | `memory: 1g` (1 GB) |
|     | `memory_reservation` | Soft limit. Container can exceed if host has free memory. | `memory_reservation: 512m` (512 MB) |
|     | `mem_limit` (deprecated) / `memory_swap` | Total memory (RAM + swap). Set equal to `memory` to disable swap. | `memory_swap: 1g` (if `memory: 1g`) |
| **CPU** | `cpus` | Absolute CPU limit (e.g., 1.5 CPUs). | `cpus: "1.0"` |
|     | `cpu_shares` | Relative CPU weight (default 1024). Higher value gets more CPU time. | `cpu_shares: 2048` |
|     | `cpu_period` / `cpu_quota` | Precise throttling: CPU time within a period. | `cpu_period: 100000`, `cpu_quota: 50000` (50% of one CPU) |
|     | `cpuset` (`cpuset_cpus`) | Bind container to specific CPU cores. | `cpuset: "0,1"` (cores 0 and 1) |

2.  **Running Containers as Non-Root Users:** Always specify a non-root user for processes running inside containers.<sup>10</sup> This minimizes the potential damage if a container is compromised, as the attacker would not immediately gain elevated privileges.
    
3.  **Dropping Unnecessary Capabilities:** Containers are started with a default set of Linux capabilities. It is a best practice to drop all unnecessary capabilities (`--cap-drop all`) and explicitly add back only those capabilities that are strictly required for the container's function (e.g., `NET_BIND_SERVICE` for binding to low-numbered ports).<sup>10</sup> This adheres to the principle of least privilege.
    
4.  **Utilizing Read-Only Filesystems:** For containers that do not need to write to their filesystem during runtime (e.g., web servers serving static content), use the `--read-only` flag.<sup>10</sup> This prevents malicious writes to the container's filesystem and can enhance security.
    
5.  **Optimizing Docker Logging Drivers:** Docker's default `json-file` logging driver in blocking mode is generally recommended for most use cases due to its speed and reliability, as it writes logs to a local file on the host.<sup>32</sup> For high-volume logging or when forwarding logs to a remote destination (e.g., a centralized log management service), consider other drivers like
    
    `fluentd` or `syslog` in non-blocking mode. However, be aware that remote logging can introduce potential performance impacts or latency if not configured carefully.<sup>32</sup>
    
6.  **Image Security:**
    
    - **Trusted/Minimal Base Images:** Always use official Docker images or images from verified publishers.<sup>10</sup> Prefer minimal base images (e.g., Alpine-based variants) to reduce the overall image size and minimize the attack surface by including fewer unnecessary packages and dependencies.<sup>10</sup>
        
    - **Multi-Stage Builds:** Implement multi-stage Dockerfiles to separate build-time dependencies from the final runtime image.<sup>33</sup> This practice results in smaller, more secure images by excluding build tools and intermediate files from the final artifact.
        
    - **Regular Rebuilds:** Docker images are immutable snapshots. Regularly rebuild your images from your Dockerfiles to ensure they incorporate the latest OS packages and security patches from their base images and dependencies.<sup>10</sup> Using the
        
        `--no-cache` option during builds can force fresh downloads and prevent stale cached layers from being used.
        
    - **`.dockerignore`:** Utilize a `.dockerignore` file to exclude unnecessary files (e.g., source code, `.git` directories, temporary files) from the build context.<sup>34</sup> This reduces the image size and prevents sensitive data from accidentally being included in the image.
        

### F. Linux Kernel Tuning for Docker Performance

Specific Linux kernel parameters can be tuned to optimize performance and stability for applications running within Docker, particularly those with high memory or I/O demands.

1.  **`vm.overcommit_memory` for Database Stability (e.g., Redis):**
    
    - **Purpose:** This kernel parameter controls how the system handles memory allocation requests, especially when they exceed available physical memory. Redis, often used by Paperless-ngx for caching and fast data access <sup>1</sup>, issues a warning if
        
        `vm.overcommit_memory` is not set to `1`. This is because background save or replication operations might fail under low memory conditions if the kernel is too restrictive in allocating memory.<sup>37</sup> Setting it to
        
        `1` allows the kernel to always permit memory allocation, assuming that not all allocated memory will be used simultaneously, which is common for many applications.
        
    - **Configuration:** To ensure Redis stability and prevent memory allocation issues, add `vm.overcommit_memory = 1` to `/etc/sysctl.conf`.<sup>37</sup> Apply the changes by running
        
        `sudo sysctl -p`.
        
2.  **Disabling Transparent Huge Pages (THP):**
    
    - **Purpose:** Transparent Huge Pages (THP) is a Linux kernel feature designed to improve memory management performance by using larger memory pages. However, for certain applications, including Redis, THP can introduce significant latency and memory usage issues.<sup>37</sup>
        
    - **Configuration:** To mitigate these issues, it is recommended to disable THP. This can be achieved by adding `transparent_hugepage=never` to your GRUB kernel boot line in `/etc/default/grub`.<sup>39</sup> After editing, run
        
        `sudo update-grub` to regenerate the GRUB configuration file, and then reboot the system for the change to take effect. Alternatively, for a temporary change or if GRUB modification is not preferred, the command `echo never > /sys/kernel/mm/transparent_hugepage/enabled` can be run as root, and added to `/etc/rc.local` for persistence across reboots.<sup>37</sup>
        

The table below provides a summary of recommended Linux kernel tuning parameters:

| Parameter | Recommended Value | Configuration Method | Rationale/Impact |
| --- | --- | --- | --- |
| `vm.overcommit_memory` | `1` | Add to `/etc/sysctl.conf`, then `sudo sysctl -p`.<sup>38</sup> | Prevents memory allocation failures for applications like Redis, improving stability.<sup>37</sup> |
| `transparent_hugepage` | `never` | Add `transparent_hugepage=never` to `GRUB_CMDLINE_LINUX` in `/etc/default/grub`, then `sudo update-grub` and reboot.<sup>39</sup> | Mitigates latency and memory usage issues for certain applications, including Redis.<sup>37</sup> |

The comprehensive nature of these hardening steps, from SSH security to kernel tuning, underscores that securing a home server is not a single action but a layered defense. Each recommendation addresses different attack vectors and contributes to a more robust and resilient system.

## V. Docker Volume Backup Strategy within Proxmox

A robust backup strategy is essential for any server, including a home server running Docker. Leveraging Proxmox VE's integrated backup features is the most straightforward and recommended approach for backing up the entire Debian Docker VM.

### A. Leveraging Proxmox VE's Integrated Backup Features

Proxmox VE offers a powerful, integrated solution for backing up virtual machines and LXCs.<sup>20</sup> This is the primary and most recommended method for protecting your Debian Docker VM.

- **Full Backups:** Proxmox backups are always full backups, encompassing the entire VM configuration and all its data.<sup>20</sup> These backups can be initiated conveniently via the Proxmox graphical user interface (GUI) or through the
    
    `vzdump` command-line tool.
    
- **Backup Storage:** Before any backup operation can commence, a backup storage location must be defined within Proxmox. Common options include Network File System (NFS) shares or a dedicated Proxmox Backup Server (PBS).<sup>20</sup> PBS is highly recommended for its advanced features, particularly its data deduplication capabilities, which can significantly reduce storage requirements, although it does require more RAM on the PBS host.<sup>20</sup>
    

### B. Ensuring Data Consistency for Docker Volumes During Backups

Ensuring data consistency during backups is crucial, especially for applications that handle dynamic data.

- **For Stateless Containers (Tika/Gotenberg):** As Tika and Gotenberg are largely stateless services, their internal data is not critical for long-term persistence. A VM-level backup using Proxmox's snapshot mode, particularly with the Qemu Guest Agent enabled, is generally sufficient for these services. This mode provides low downtime while offering a good level of data consistency.<sup>20</sup>
    
- **For Stateful Applications:** While the current VM is intended for stateless Tika/Gotenberg, it is important to understand consistency considerations for any stateful Docker containers that might be deployed on this VM in the future.
    
    - **Stop Mode:** The highest level of data consistency is achieved by gracefully stopping the VM before the backup process begins. This ensures that all data is flushed from memory to disk, guaranteeing a consistent backup state.<sup>20</sup> However, this method incurs a short period of downtime for the VM.
        
    - **Snapshot Mode with Qemu Guest Agent:** This mode offers low downtime and is highly effective when the Qemu Guest Agent is properly enabled and installed within the VM. The Guest Agent allows Proxmox to issue `guest-fsfreeze-freeze` and `guest-fsfreeze-thaw` commands to quiesce the filesystem within the guest OS, significantly improving data consistency during live snapshots.<sup>20</sup> The importance of the Qemu Guest Agent, previously discussed in VM settings, extends directly to ensuring consistent backups, demonstrating how different system configurations are interconnected and contribute to overall reliability.
        
    - **Application-Aware Backups:** For highly transactional data, such as databases (which Paperless-ngx itself uses in its LXC, but might be hosted on this VM in other scenarios), it is ideal to perform application-specific backups (e.g., database dumps) *before* the VM-level backup. Alternatively, gracefully shutting down individual Docker containers using `docker compose stop` before the VM backup can ensure their data volumes are consistent.<sup>41</sup> This discussion of backup consistency for Docker volumes, differentiating between stateless and stateful applications, provides valuable context beyond the immediate query, preparing the user for future Docker deployments on their VM. It also reinforces the initial design choice to keep Tika and Gotenberg stateless, simplifying their backup profile.
        

### C. Recommended Backup Schedules and Retention Policies

Establishing a clear backup schedule and retention policy is crucial for effective disaster recovery and data management.

- **Scheduled Jobs:** Proxmox allows for the creation of periodic backup jobs through its GUI (`Datacenter` → `Backup`) or via its API.<sup>20</sup> These jobs can be configured to run automatically on specific days and times for selected nodes and guest systems.
    
- **Retention Policies (`prune-backups`):** Proxmox offers flexible retention options using the `prune-backups` setting, which can be configured for individual backup jobs or at the storage level.<sup>20</sup> A common retention strategy for a home server might include:
    
    - `keep-last=3`: Retains the last 3 backups (useful for quick rollbacks after recent changes).
        
    - `keep-daily=13`: Keeps the latest backup for the last 13 days (providing two weeks of daily backups).
        
    - `keep-weekly=8`: Retains the latest backup for the last 8 weeks (two months of weekly backups).
        
    - `keep-monthly=11`: Keeps the latest backup for the last 11 months (a year of monthly backups).
        
    - `keep-yearly=9`: Retains the latest backup for the last 9 years (for long-term archival).<sup>20</sup>
        
- **Backup Protection:** Critical backups can be marked as "protected" within Proxmox to prevent their accidental deletion via the Proxmox UI, CLI, or API.<sup>20</sup>
    
- **Single File Restore:** If using Proxmox Backup Server, a highly convenient feature is the ability to browse and restore individual files or directories directly from VM backups, which is invaluable for recovering specific configuration files or Docker volumes without needing to restore the entire VM.<sup>20</sup>
    

## VI. Conclusion

Establishing a robust and efficient home server environment, particularly one involving virtualization and containerization, necessitates a multi-faceted approach to configuration, security, and maintenance. This report has detailed the essential steps and best practices for deploying Apache Tika and Gotenberg on a Debian 12 Docker VM within a Proxmox host, ensuring optimal performance, security, and maintainability.

Key recommendations include:

- **Docker Compose Deployment:** Utilize the provided Docker Compose file for Tika and Gotenberg, ensuring that their ports are correctly exposed on the VM's network interface (e.g., `0.0.0.0`) to allow accessibility from the Paperless-ngx LXC. The stateless nature of these services simplifies their operational management.
    
- **Structured Docker Project Organization:** Adopt a systematic directory structure for Docker Compose projects, preferably within `/srv/docker/`, to enhance maintainability, simplify management, and streamline backup procedures.
    
- **Optimized Proxmox VM Settings:** Configure the Debian 12 VM in Proxmox with `VirtIO SCSI` disk controllers and `VirtIO` network devices for superior I/O and network performance. Select the `host` CPU type to leverage the full capabilities of your physical CPU. Crucially, enable "Discard" for virtual disks in Proxmox and ensure `fstrim` is active within the Debian guest to maintain SSD health and reclaim storage space. The Qemu Guest Agent must also be installed and enabled for enhanced VM management and consistent backups.
    
- **Layered Debian Host Hardening:** Implement a comprehensive security posture for the Debian 12 host. This involves configuring automated security updates via `unattended-upgrades`, hardening the SSH server by changing the default port, disabling root login, and enforcing key-based authentication. Furthermore, properly configure UFW to integrate with Docker's `iptables` rules, ensuring that exposed container ports are adequately protected by the host firewall.
    
- **Secure Docker Daemon and Containers:** Prioritize Docker daemon security by considering rootless mode or user namespace remapping to mitigate privilege escalation risks. Avoid exposing the Docker socket over TCP without TLS and refrain from bind mounting `/var/run/docker.sock` into containers. Apply resource limits (CPU, memory) to individual containers to prevent resource exhaustion, and adhere to container security best practices such as running processes as non-root users, dropping unnecessary capabilities, and using trusted, minimal base images with multi-stage builds.
    
- **Integrated Proxmox Backup Strategy:** Leverage Proxmox VE's integrated backup features for the entire Docker VM. Utilize snapshot mode with the Qemu Guest Agent for consistent backups with minimal downtime. Establish scheduled backup jobs and implement robust retention policies to ensure data recoverability. For any future stateful Docker applications on this VM, consider application-aware backups or graceful container shutdowns to ensure data integrity.
    

A secure and performant home server is the result of careful planning and diligent adherence to best practices across all layers of the stack, from the underlying virtualization platform to the containerized applications themselves. By implementing these recommendations, the home server environment will be well-prepared for reliable and secure operation.