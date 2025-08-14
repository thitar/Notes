---
title: >-
  Comprehensive Guide to Proxmox Backup Server: Protecting Your Proxmox VE and
  Phy
updated: 2025-08-03 20:54:18Z
created: 2025-08-03 20:54:14Z
latitude: 49.61162100
longitude: 6.13193460
altitude: 0.0000
---

# Comprehensive Guide to Proxmox Backup Server: Protecting Your Proxmox VE and Physical Servers

## I. Introduction to Proxmox Backup Server (PBS)

### What is Proxmox Backup Server?

Proxmox Backup Server (PBS) stands as an open-source, enterprise-class, client-server backup solution designed to provide robust data protection for modern IT environments.<sup>1</sup> Its primary function is the efficient backing up and restoring of virtual machines (VMs), containers (LXC), and physical hosts.<sup>2</sup> This broad compatibility positions PBS as a versatile tool capable of managing data across diverse infrastructure types.

The architectural foundation of PBS is a minimal Debian GNU/Linux distribution, which includes native ZFS support.<sup>1</sup> This choice of operating system and filesystem provides a stable, secure, and highly capable platform for storing and managing backup data. For ease of use, PBS offers an intuitive web-based graphical user interface (GUI) for simplified management tasks. Concurrently, it provides a powerful command-line interface (CLI) for advanced users who require greater control or wish to automate complex operations.<sup>1</sup> The availability of both interfaces caters to a wide range of administrative preferences and expertise levels.

### Key Features

Proxmox Backup Server incorporates a suite of features designed to optimize backup efficiency, security, and reliability:

- **Deduplication:** A cornerstone of PBS’s efficiency is its implementation of both client-side and server-side data deduplication.<sup>4</sup> This advanced technique ensures that identical data blocks are identified and stored only once, regardless of their origin within the backup set. This process significantly conserves disk space and reduces network load during backup operations, as only unique data needs to be transferred and stored. This is particularly beneficial when backing up multiple virtual machines or containers that share common operating system files or application binaries, as the shared data is stored just once, leading to substantial storage savings.<sup>5</sup>
    
- **Compression:** To further optimize storage utilization and accelerate data transfer, Proxmox Backup Server employs the Zstandard (ZSTD) compression algorithm.<sup>2</sup> ZSTD is renowned for its high compression ratios while maintaining remarkable speed, capable of compressing several gigabytes of data per second. This combination of efficiency and performance makes it an ideal choice for backup operations where both storage footprint and backup window are critical considerations.
    
- **Encryption:** Security is a paramount concern in any backup strategy. PBS addresses this by providing strong client-side encryption using AES-256 in Galois/Counter Mode (GCM).<sup>3</sup> This means that data is encrypted at the source machine
    
    *before* it is transmitted to the backup server. This pre-encryption ensures the confidentiality of data even if the backup storage itself is compromised or resides on an untrusted target. Furthermore, PBS supports the generation and use of master keys, which provide a secure mechanism for storing and recovering encryption keys, adding another layer of data protection and disaster recovery capability.<sup>5</sup>
    
- **Data Integrity Verification:** To guarantee the reliability and restorability of backups, PBS integrates a built-in SHA-256 checksum algorithm.<sup>4</sup> Each backup includes a manifest file, typically named
    
    `index.json`, which contains a comprehensive list of all backup files along with their respective sizes and checksums. This manifest file is crucial for verifying the integrity of each backup. Administrators can schedule regular integrity checks to detect any data corruption, often referred to as “bit rot,” and confirm that backups remain safe and fully restorable.<sup>5</sup> The checksumming process also plays a vital role in the deduplication layer, enabling the system to accurately detect identical blocks of data.
    
- **Incremental Backups:** Proxmox Backup Server performs incremental backups efficiently. After an initial full backup, only the changes—new or modified data blocks—between subsequent backup operations are transferred to the server.<sup>2</sup> This approach significantly reduces the time required for backup tasks and minimizes network bandwidth consumption, making daily or even more frequent backups practical without overburdening the network or storage infrastructure.
    
- **Ransomware Protection:** In the current threat landscape, ransomware protection is a critical aspect of data management. PBS includes several features that contribute to a robust defense strategy against ransomware attacks. These include fine-grained access control mechanisms, regular data integrity verification to detect unauthorized modifications, and the crucial ability to create off-site backup copies through remote synchronization and tape backups.<sup>2</sup> These features collectively help organizations and individuals plan and implement a comprehensive ransomware defense.
    

### Why use PBS for your Proxmox VE environment?

The tight and native integration of Proxmox Backup Server with Proxmox Virtual Environment (PVE) makes it an ideal choice for seamlessly backing up virtual machines and containers directly from the PVE interface.<sup>2</sup> This integration simplifies the backup process, allowing administrators to manage VM and container backups with familiar tools and workflows.

Beyond convenience, PBS offers advanced features like block-level deduplication and client-side encryption that are not typically available when performing simple file-level backups using traditional methods.<sup>1</sup> These capabilities translate into highly efficient space utilization on the backup storage and often result in faster restore times compared to generic backup solutions. The specialized design of PBS for virtualized environments ensures that backups are consistent, reliable, and optimized for recovery.

The consistent emphasis on PBS being an “enterprise-class” and “open-source” solution, coupled with its strong encryption, data integrity verification, and ransomware protection features, highlights its role as a strategic investment for data resilience.<sup>1</sup> Adopting PBS is not merely about creating copies of data; it is about establishing a robust system that ensures data safety, confidentiality, and rapid recovery in the face of various threats. The fact that it is free to download and use <sup>1</sup>, despite offering these enterprise-grade features, presents a compelling value proposition for both home lab enthusiasts and small businesses seeking reliable data protection.

The recurring description of PBS as a “client-server” solution <sup>1</sup> reveals a fundamental architectural design that extends its applicability beyond Proxmox VE. This model enables PBS to back up “physical hosts” in addition to virtual machines and containers. The client-server architecture implies that the initial backup processing, including data collection, compression, and encryption, occurs on the client machine. The processed data is then transmitted to the centralized PBS repository. This distributed approach supports flexible deployment and management of backups from diverse sources—such as a Proxmox VE server and other Linux machines—to a single, centralized PBS instance. This directly addresses the broader need for backing up other physical servers across a network.

## II. Planning Your PBS Deployment

Effective deployment of Proxmox Backup Server requires careful consideration of hardware, network, and security aspects to ensure optimal performance and data integrity.

### Hardware Considerations for PBS

A critical decision in deploying Proxmox Backup Server is whether to run it on dedicated hardware or as a virtualized instance. While it is technically possible to deploy PBS as a virtual machine on top of an existing Proxmox VE server, this approach is strongly recommended against for any production or critical backup environment.<sup>1</sup> Running PBS on the same host as the VMs it backs up creates a single point of failure. If the Proxmox VE host experiences a hardware failure, an operating system issue, or even goes offline due to system maintenance or “experiments” <sup>9</sup>, the backup server would become inaccessible. This would prevent access to or restoration of backups precisely when they are most needed. For true disaster recovery capabilities, PBS should reside on separate, dedicated hardware.

**System Requirements:**

For basic testing or evaluation purposes, PBS can run on minimal hardware: a 64-bit CPU, a minimum of 1GB RAM, a hard drive, and one network interface card (NIC).<sup>10</sup> However, for production environments, more robust specifications are recommended to ensure reliable and efficient operation.

- **CPU:** A 64-bit CPU with at least 2 cores is the minimum requirement <sup>11</sup>, but a 4-core CPU or more is recommended for better performance, especially when handling multiple backup streams or intensive tasks like garbage collection.<sup>11</sup>
    
- **RAM:** A minimum of 4 GB of RAM is required for PBS <sup>11</sup>, with 8 GB or more recommended for optimal performance. If ZFS is chosen for the datastore, additional memory is crucial, with a general guideline of approximately 1 GB of RAM for every TB of storage used by ZFS.<sup>10</sup>
    
- **Storage:**
    
    - **PBS Installation Disk:** Approximately 10 GB of free disk space is needed for the PBS operating system installation.<sup>11</sup> It is highly recommended to use a separate physical drive for the PBS OS installation, distinct from the storage volume dedicated to backups.<sup>9</sup> This separation isolates the operating system from the backup data, improving reliability and simplifying maintenance.
        
    - **Backup Storage Volume (Datastore):** A dedicated storage volume for backups is essential.<sup>9</sup> For superior performance, particularly for IOPS-intensive tasks such as Garbage Collection (GC) and data verification, Solid State Drives (SSDs) are strongly recommended.<sup>10</sup> These tasks involve numerous random read and write operations on small data chunks, which SSDs handle far more efficiently than traditional Hard Disk Drives (HDDs).
        
    - **External USB Drive:** If an external USB hard drive is intended for use with PBS, it should be connected directly to the *dedicated PBS machine*, not the Proxmox VE host. This is a critical distinction, as PBS requires its own local or network-mounted storage for its datastores.
        

The interaction between hardware, network, and storage significantly influences the performance and reliability of a PBS deployment. The recommendation for dedicated PBS hardware directly mitigates the risk of a single point of failure, thereby enhancing overall system reliability. The strong emphasis on SSDs and high IOPS for datastores, coupled with warnings about network latency and HDD performance, highlights that storage performance often represents a major bottleneck for PBS’s core functionalities, especially during demanding operations like Garbage Collection. Similarly, the advice to use dedicated NICs and implement Quality of Service (QoS) policies underscores the network’s crucial role in maintaining overall performance. This suggests that merely meeting minimum hardware specifications is insufficient for a truly robust and efficient backup system; administrators must consider these interconnected factors to prevent performance degradation and ensure timely backups and restores. This consideration is particularly relevant for environments utilizing a Synology NAS, which may primarily rely on HDDs.

**Table 1: Proxmox Backup Server System Requirements**

| Category | Minimum Requirement (for testing only) | Recommended for Production Environment |
| --- | --- | --- |
| **CPU** | 64-bit CPU | 64-bit CPU with 4+ cores <sup>11</sup> |
| **RAM** | 1 GB <sup>10</sup> | 8 GB + 1 GB/TB for ZFS <sup>10</sup> |
| **OS Disk Space** | Hard drive <sup>10</sup> | 10 GB (separate disk recommended) <sup>9</sup> |
| **Datastore Storage** | Hard drive <sup>10</sup> | SSD storage for faster access <sup>10</sup> |
| **Network Interface** | One NIC <sup>10</sup> | Dedicated NIC for backup traffic (Gbit/10Gbit+) <sup>10</sup> |
| **Operating System** | N/A | Debian 10 or 11 <sup>11</sup> |

### Network Requirements for Optimal Performance

A robust network foundation is essential for a Proxmox Backup Server to operate effectively. To prevent network congestion and ensure stable, high-speed backup transfers, it is advisable to use a dedicated network interface card (NIC) specifically for backup traffic.<sup>11</sup> This isolation helps prevent backup operations from impacting other critical network services. Sufficient network bandwidth is crucial for smooth backup operations; insufficient bandwidth can lead to slow transfers, prolonged backup windows, or even failed backups.<sup>11</sup> Proxmox Backup Server supports 10 Gbit and higher network speeds, which can be beneficial for large-scale deployments or environments with stringent backup windows.<sup>10</sup> If the network experiences contention or heavy traffic, implementing Quality of Service (QoS) policies can help prioritize backup traffic, ensuring it receives the necessary bandwidth to complete tasks efficiently.<sup>11</sup>

### Security Best Practices

Implementing comprehensive security measures is fundamental for any backup solution, as it directly impacts the integrity and confidentiality of the stored data.

- **Network Segmentation (VLANs):** To enhance security and reduce potential attack surfaces, it is advisable to isolate backup traffic by setting up Virtual Local Area Networks (VLANs).<sup>11</sup> This segregates backup data from other network traffic, preventing unauthorized access or interference.
    
- **Firewall Rules:** Configure strict firewall rules on the PBS machine to limit access only to necessary ports and trusted IP addresses.<sup>11</sup> This minimizes the exposure of the backup server to potential threats.
    
- **Regular Updates:** Keeping the PBS operating system, firmware, and Proxmox Backup Server software updated regularly is crucial for patching known vulnerabilities and ensuring optimal security.<sup>11</sup> This proactive approach helps protect against newly discovered exploits.
    
- **Strong Password Policies:** Enforce complex and unique passwords for all user accounts on PBS to prevent unauthorized access through brute-force attacks or credential stuffing.<sup>11</sup>
    
- **Two-Factor Authentication (2FA):** Proxmox Backup Server supports two-factor authentication.<sup>4</sup> Implementing 2FA adds an extra layer of security, making it significantly harder for unauthorized users to gain access even if they compromise a password.<sup>11</sup> It is important to note that enabling 2FA for the
    
    `root` user on the *Proxmox VE host* might complicate automated backup jobs if the backup solution or other integrated tools (such as Veeam, as noted in <sup>13</sup>) rely on
    
    `root` credentials. A better practice is to create and use dedicated PBS users with specific, limited roles for integration, rather than relying solely on the PVE `root` account.
    
- **Least Privilege Principle:** Adhere to the principle of least privilege by assigning users only the minimum access rights required for them to perform their tasks.<sup>11</sup> PBS facilitates this by offering predefined user roles such as Administrator (full access), Backup Operator (can perform backups but not change settings), and Read-Only User (can only view backups).<sup>5</sup> This granular control helps prevent accidental or malicious modifications to the backup system or data.
    

The consistent emphasis on comprehensive security measures, including network segmentation, firewalls, strong passwords, two-factor authentication, the principle of least privilege, and client-side encryption, underscores their importance. This is not solely about protecting the PBS server from unauthorized access; it is fundamentally about ensuring the integrity and confidentiality of the backup data itself. The observation about two-factor authentication on the PVE root account potentially impacting backup jobs serves as a practical illustration of how security implementations must be carefully planned to avoid operational conflicts. The capability to encrypt data *before* it leaves the client machine is a powerful feature that guarantees data remains unusable to unauthorized parties, even if the storage medium is compromised. This holistic approach to security is crucial for building and maintaining trust in the backup system, which is paramount for effective disaster recovery.

## III. Setting Up Proxmox Backup Server

The initial setup of Proxmox Backup Server involves several key steps, from preparing the installation media to configuring initial network and user access.

### Downloading and Creating a Bootable PBS Installation Medium

The first step in deploying Proxmox Backup Server is to obtain the necessary installation files. Begin by downloading the latest Proxmox Backup Server ISO image directly from the official Proxmox website.<sup>1</sup> Always ensure that downloads originate from the official source to guarantee authenticity and security, preventing the introduction of compromised software.

Once the ISO image is acquired, the next step is to create a bootable USB flash drive. A reliable utility such as Balena Etcher is recommended for this purpose.<sup>9</sup> This tool simplifies the process of writing the ISO image to the USB drive, rendering it bootable for the installation process. For advanced users with existing network infrastructure, a PXE server can serve as an alternative to a physical USB boot drive, enabling network-based installation.<sup>9</sup>

### Step-by-Step Installation Guide (Graphical Installer)

With the bootable USB drive prepared, the physical installation can commence. Plug the newly-flashed USB drive into the dedicated Proxmox Backup Server machine and power it on. To initiate the installation process, access the machine’s BIOS or UEFI settings, typically by repeatedly pressing the Delete key or F2 during the boot sequence. Within these settings, modify the boot order to prioritize booting from the USB drive.<sup>9</sup>

Upon successful booting from the USB, the installer menu will appear. Select “Install Proxmox Backup Server (Graphical)” to proceed with the guided installation.<sup>9</sup> The wizard will then display the End User License Agreement (EULA); review it and tap “I agree” to accept and continue.<sup>9</sup>

Next, choose the target drive where the Proxmox Backup Server operating system will be installed.<sup>9</sup> It is crucial to understand that PBS is a bare-metal installer, meaning the complete selected disk will be utilized, and all existing data on it will be removed.<sup>1</sup> Ideally, this installation disk should be a separate physical drive from the one intended for your backup datastore to prevent data loss and improve system architecture.

Proceed to configure the geographical settings by choosing the correct Country, Timezone, and Keyboard Layout.<sup>9</sup> Following this, set a strong Password for the

`root` user account on your PBS system and provide an Email address. This email address can be used for system notifications and alerts.<sup>9</sup>

The network configuration is a critical step. Ensure that the correct `Network Device` is chosen as the `Management Interface`. Then, accurately enter the `Hostname (FQDN)`, `IP Address (CIDR)`, `Gateway`, and `DNS Server` details for your network environment.<sup>9</sup>

Finally, a summary screen will present all configured settings for review. Double-check these details for accuracy, then tap `Install` to begin the operating system installation. The PBS wizard will proceed to install the OS on the chosen PC. Once the installation is complete, click `Reboot` and remember to disconnect the installation media to ensure the system boots from the newly installed OS.<sup>9</sup>

### Initial Web Interface Access and Configuration

After the Proxmox Backup Server machine successfully reboots, note its assigned IP address, which is typically displayed on the command-line interface. From another computer connected to the same network, open a web browser and navigate to `https://<PBS_IP_Address>:8007`.<sup>9</sup>

It is common for the web browser to display a security warning due to the self-signed SSL certificate used by default. This is expected and can be safely bypassed (e.g., by selecting “Proceed to IP Address (unsafe)” or a similar option, depending on the browser) to access the PBS web UI.<sup>9</sup> On the login screen, enter

`root` as the `User name` and provide the `Password` that was set during the installation process.<sup>9</sup>

The detailed installation steps, particularly those related to network settings, hostname, root password, and timezone, highlight the importance of foundational configuration for system stability and security.<sup>9</sup> Any inaccuracies or omissions at this initial stage can lead to significant connectivity issues, security vulnerabilities, or operational problems later on. The explicit instructions to set a strong root password and provide an email address, along with correctly configuring the network interface, underscore that these are not merely procedural steps but essential elements for building a stable and secure backup server. Proactively addressing the common initial user experience hurdle of bypassing the self-signed certificate warning also demonstrates attention to practical implementation details.

## IV. Configuring Storage for Proxmox Backup Server

Once Proxmox Backup Server is installed and accessible, the next crucial step is to configure storage locations, known as datastores, where backups will be saved.

### A. Local Storage (External USB Drive on PBS Host)

For optimal performance, particularly for demanding tasks such as Garbage Collection (GC) and Verify jobs, local SSD storage is highly recommended for Proxmox Backup Server datastores.<sup>11</sup> If the external USB drive is a traditional Hard Disk Drive (HDD), it will likely introduce performance bottlenecks due to its lower Input/Output Operations Per Second (IOPS) capabilities.

Preparing the USB Drive:

Physically connect the external USB drive to your dedicated Proxmox Backup Server machine. Access the PBS shell (either via the web UI or SSH) and identify the USB drive’s device identifier (e.g., /dev/sdX) using Linux commands like lsblk or fdisk -l.

Before formatting, initialize the disk with a new GPT partition table using the command: `# proxmox-backup-manager disk initialize sdX`.<sup>15</sup> This prepares the disk for use by creating a modern partition scheme.

Next, format the drive with a Linux filesystem. `ext4` or `xfs` are common and well-supported choices. ZFS is also an option if there is an intention to create a ZFS pool for the datastore. To create an `ext4` filesystem and automatically add it as a datastore, the following command can be used: `# proxmox-backup-manager disk fs create <datastore_id> --disk sdX --filesystem ext4 --add-datastore true`.<sup>15</sup> This command will create the filesystem, mount it, and then configure the datastore within PBS. A critical requirement for PBS’s internal file layout is that the chosen filesystem

*must* support at least 65538 subdirectories per directory, as PBS pre-creates a large number of chunk namespace directories.<sup>15</sup>

**Adding the USB Drive as a Datastore in PBS (if not done automatically with `add-datastore`):**

- Via Web Interface:
    
    In the PBS web UI, navigate to the Storage/Disks tab, then select Directory.9 Click
    
    `Create: Directory`.<sup>9</sup> Provide a
    
    `Name` for the directory, and select the `Disk` and `Filesystem`.<sup>9</sup> Alternatively, one can navigate to the
    
    `Datastore` section in the side menu and click `Add Datastore`.<sup>15</sup> Enter a descriptive
    
    `Name` for the datastore (e.g., `usb-backups`) and specify the `Backing Path` (e.g., `/mnt/datastore/usb-backups` if the `disk fs create --add-datastore true` command was used).<sup>15</sup> Configure
    
    `Prune Options`, which define backup retention policies, and set the `Garbage Collector Schedule` to automate maintenance tasks.<sup>4</sup>
    
- Via Command Line:
    
    A new datastore can also be created from the command line using: # proxmox-backup-manager datastore create &lt;datastore_id&gt; &lt;backing_path&gt;.15 For example, to create a datastore named
    
    `usb-backups` at a specific mount point: `# proxmox-backup-manager datastore create usb-backups /mnt/usb_drive_mount_point`.
    

### B. Network Storage (Synology NAS)

Utilizing network-attached storage (NAS) such as NFS or SMB/CIFS shares from a Synology NAS for Proxmox Backup Server datastores requires careful consideration of performance implications.

Performance Considerations for Network Storage (IOPS, Latency):

A critical warning pertains to using network-attached storage, especially if the NAS primarily uses traditional Hard Disk Drives (HDDs). This setup will likely result in significantly slower performance compared to local SSDs.12 PBS tasks, particularly Garbage Collection (GC) and data verification, are highly IOPS-intensive, involving millions of random read/write operations on small data chunks.12 Network latency, combined with the inherent seek times and lower IOPS of HDDs, will severely impact these operations. This can lead to prolonged GC times, potentially lasting hours or even days, during which the NAS might become unusable for other tasks.12 Furthermore, a slow backup destination has been observed to potentially lead to VM corruption during backup operations.12 Therefore, while technically feasible, it is generally advisable to use a Synology NAS for

*secondary* backup copies (e.g., via PBS Sync Jobs to offload data from a fast local datastore) or for less critical backups, rather than as the primary, high-performance datastore, unless the Synology NAS is equipped with SSDs.

Configuring NFS Share on Synology NAS:

To prepare the Synology NAS for use as an NFS datastore, log in to the Synology DiskStation Manager (DSM) web interface. Navigate to Control Panel > File Services > NFS. Ensure that Enable NFS service is checked and specify the Maximum NFS protocol (NFSv3 is commonly compatible and recommended, as Synology does not fully support NFS v4.2 for Proxmox needs).14

Next, go to `Control Panel > Shared Folder`, select the shared folder designated for backups, and click `Edit`. Navigate to the `NFS Permissions` tab and click `Create`. Enter the IP address of your *Proxmox Backup Server* in the `Hostname or IP` field. Configure other settings as appropriate, ensuring `Read/Write` access for the PBS machine. Crucially, note the shared folder’s mount path displayed in the lower-left corner (e.g., `/[volume name]/[shared folder name]`).<sup>18</sup>

Mounting NFS Share on PBS:

Log in to your Proxmox Backup Server web UI and open the >_ Shell.14 Create a local mount point directory on your PBS server:

`# mkdir /mnt/synology-nfs`.<sup>14</sup> Assign appropriate ownership and permissions to this new directory. Proxmox Backup Server typically operates using the

`backup` user (UID 34) and group, so ensure this user has write access: `# chown backup:backup /mnt/synology-nfs` and `# chmod 775 /mnt/synology-nfs`.<sup>14</sup>

Add an entry to the `/etc/fstab` file for automatic mounting of the NFS share after reboots. Replace `<Synology_IP>` and `<Synology_Share_Path>` with your actual Synology NAS IP and the noted shared folder path: `# echo "<Synology_IP>:/<Synology_Share_Path> /mnt/synology-nfs nfs vers=3,nouser,atime,auto,retrans=2,rw,dev,exec 0 0" >> /etc/fstab`.<sup>14</sup> For example:

`192.168.1.100:/volume1/ProxmoxBackups /mnt/synology-nfs nfs vers=3,nouser,atime,auto,retrans=2,rw,dev,exec 0 0`. Mount the newly added volume using `# mount -a`.<sup>14</sup> Reload the systemd daemon to ensure it recognizes the changes made to

`fstab`: `# systemctl daemon-reload`.<sup>14</sup> It is good practice to verify permissions again after mounting:

`# chmod 775 /mnt/synology-nfs`.<sup>14</sup>

Adding NFS Share as a Datastore in PBS:

In the Proxmox Backup Server web UI, navigate to the Datastore section in the left panel and click Add Datastore.14 Enter a descriptive

`Name` for the datastore (e.g., `synology-nfs-datastore`). Specify the `Backing Path` corresponding to your local mount point (`/mnt/synology-nfs`).<sup>14</sup> Configure

`Prune Options` and `Garbage Collector Schedule` according to your backup retention strategy.

Configuring SMB/CIFS Share on Synology NAS (Alternative to NFS):

Similar to NFS, SMB/CIFS can be enabled on the Synology NAS, a shared folder created, and appropriate permissions set for a dedicated backup user. The SMB/CIFS share can then be mounted on the PBS server. Specific mount options for the backup user (UID 34) may be required: uid=34,noforceuid,gid=34,noforcegid.17 After mounting, it can be added as a datastore in PBS. However, it is important to note that CIFS has been reported to exhibit slower backup speeds and potential issues with pruning or deleting backups.17 For these reasons, NFS is generally preferred for Proxmox Backup Server datastores on network-attached storage.

## V. Integrating Proxmox VE with Proxmox Backup Server

The seamless integration between Proxmox VE and Proxmox Backup Server is a core advantage, simplifying backup management for virtual environments.

### Adding PBS as Storage in Proxmox VE

To enable Proxmox VE to utilize the Proxmox Backup Server as a backup destination, the PBS instance must be added as a storage target within the PVE environment. This process is straightforward and can be accomplished via the Proxmox VE web interface or command line.

Via Web Interface:

Log in to your Proxmox VE web interface. Navigate to Datacenter > Storage in the left-hand menu.8 Click the

`Add` button and select `Proxmox Backup Server` from the dropdown list.<sup>8</sup>

In the configuration dialog, provide the following details:

- **ID:** Enter a unique identifier for this storage (e.g., `pbs-main-storage`).<sup>9</sup>
    
- **Server:** Input the IP address or hostname (FQDN) of your Proxmox Backup Server workstation.<sup>9</sup>
    
- **Username:** Specify the username for authentication on the Proxmox Backup Server. It is recommended to use a dedicated backup user (e.g., `backupuser@pbs`) rather than `root@pam` for security purposes, although `root@pam` is a valid option.<sup>6</sup> Remember to include the realm (e.g.,
    
    `@pam` or `@pbs`).<sup>6</sup>
    
- **Password:** Enter the password associated with the specified username.<sup>6</sup>
    
- **Datastore:** Provide the exact ID of the datastore configured on the Proxmox Backup Server where backups should be saved (e.g., `usb-backups` or `synology-nfs-datastore`).<sup>6</sup> This field will auto-populate with available datastores if the server IP and credentials are correct.<sup>18</sup>
    
- **Fingerprint:** For enhanced security and to establish a trust relationship, copy the SHA-256 fingerprint from your Proxmox Backup Server’s dashboard (under `Dashboard > Show Fingerprint`) and paste it into this field.<sup>6</sup> This verifies the authenticity of the PBS instance.
    

Click `Add` to complete the storage configuration. The new PBS storage will now appear in the `Storage` list within Proxmox VE.<sup>18</sup>

Via Command Line:

For command-line enthusiasts or automated deployments, PBS storage can also be added using the pvesm command. First, you can scan for available datastores on the PBS: # pvesm scan pbs &lt;server&gt; &lt;username&gt; \[–password &lt;string&gt;\]\[–fingerprint &lt;string&gt;\].6

Then, add the datastore to the Proxmox VE cluster: `# pvesm add pbs <id> --server <server> --datastore <datastore> --username <username> --fingerprint <fingerprint> --password`.<sup>6</sup> If the password is not provided directly, the command will prompt for it, avoiding plain-text exposure in history.<sup>20</sup>

### Creating Backup Jobs in Proxmox VE

Once the Proxmox Backup Server is added as storage, creating backup jobs for VMs and containers is a straightforward process within the Proxmox VE web interface.

Log in to the Proxmox VE web interface. Navigate to `Datacenter > Backup`.<sup>8</sup> Click

`Add` to create a new backup job.<sup>8</sup>

In the backup job configuration dialog, specify the following:

- **Storage:** Select the Proxmox Backup Server storage (the ID you assigned earlier, e.g., `pbs-main-storage`) from the dropdown list.<sup>8</sup>
    
- **Schedule:** Define the frequency and time for the backup job (e.g., daily at 2:00 a.m., weekly, monthly).<sup>8</sup> This automation ensures timely and consistent backups.
    
- **Guest/VM Selection:** Choose the specific virtual machines or containers you wish to back up. You can select multiple guests in a single backup plan.<sup>8</sup>
    
- **Backup Mode:** Select the appropriate backup mode:
    
    - **Snapshot Mode:** This is generally recommended for production environments as it creates a snapshot of the VM or container, allowing it to run continuously with minimal downtime while the backup is performed.<sup>8</sup> The backup is based on this snapshot, capturing data changes up to the snapshot point.
        
    - **Suspend Mode:** Maintains the running state but suspends activity during backup, suitable for fast backups where stopping the VM is not an option.<sup>8</sup>
        
    - **Stop Mode:** Suspends the VM or container entirely during the backup process, ensuring data consistency. The VM resumes operation after the backup completes. This is ideal when absolute data consistency is paramount and brief downtime is acceptable.<sup>8</sup>
        
- **Compression:** Select your preferred compression method (e.g., ZSTD, gzip, LZO).<sup>21</sup> PBS uses Zstandard by default for its efficiency.
    
- **Retention:** Configure backup retention policies (Prune Options) directly within the PVE interface or manage them on the PBS side.<sup>8</sup> This dictates how many hourly, daily, weekly, monthly, or yearly backups to keep.<sup>15</sup>
    
- **Encryption:** If client-side encryption is desired, this can be configured here. Note that deduplication between backups encrypted with different keys is not possible, so separate datastores might be preferred for different encryption keys.<sup>6</sup>
    

Click `Create` to finalize the backup task.<sup>8</sup> The Proxmox VE Task manager will display the progress, and a “Task OK” message will indicate successful completion.<sup>9</sup>

### Restoring Virtual Machines and Containers

Restoring a virtual machine or container from Proxmox Backup Server to Proxmox VE is a straightforward process, whether using the graphical user interface or the command line.

Via Web Interface:

To restore a VM or container, navigate to the specific VM or container in the Proxmox VE web interface. Go to its Backup tab.9 A list of available backup files will be displayed. Select the desired backup file and click

`Restore`.<sup>23</sup>

A new window will appear, allowing configuration of restoration parameters:

- **Storage:** Select the storage destination where the VM should be restored.<sup>23</sup>
    
- **Bandwidth Limit:** Optionally specify a bandwidth limit for the restoration process (`0` for no limit). This can be useful to prevent the restoration from consuming excessive network bandwidth, which could affect other running VMs, especially when restoring from external NFS/NAS storage.<sup>23</sup>
    
- **Unique:** Check this option to generate new unique attributes (e.g., MAC addresses) after restoration, which is important if the original VM is still running to avoid conflicts.<sup>23</sup>
    
- **Start after restore:** Select this to automatically start the VM once the restoration process is complete.<sup>23</sup>
    
- **Override Settings:** Customize VM settings such as name, CPU, and memory if needed.<sup>23</sup>
    

Click `Restore`. A confirmation prompt will appear, as this operation might erase existing data in the target VM. Confirm with `Yes` to start the process.<sup>23</sup> The progress can be monitored in the

`Task Log` window, which will display “TASK OK” upon completion.<sup>23</sup> After restoration, the VM will appear in the left panel under the selected node.<sup>23</sup>

Via Command Line:

For command-line restoration, the qmrestore command is used for QemuServer VMs and pct restore for LXC containers.

First, locate the backup file. Proxmox backups typically have .vma.zst, .vma.gz, or .vma.lzo extensions.23 The default location for backups is often

`/var/lib/vz/dump/`, but they will be on the PBS datastore.

To restore a QemuServer VM: `# qmrestore <backup_file_path> <new_vm_id>`.<sup>23</sup> For example,

`qmrestore /mnt/pve/NFS-test/dump/vzdump-qemu-107-2023_11_27-11_50_25.vma.zst 115` would restore the specified backup as VM ID 115.<sup>23</sup>

For LXC containers: # pct restore &lt;new_ct_id&gt; &lt;backup_file_path&gt;.21

Monitor the progress in the terminal. After completion, verify the VM’s status using `qm status <vm_id>`.<sup>23</sup>

## VI. Backing Up Other Physical Servers (Linux & Windows)

Proxmox Backup Server is designed to back up not only Proxmox VE virtual machines and containers but also physical hosts.<sup>2</sup> This capability is facilitated by the client-server architecture of PBS.

### A. Linux Physical Servers (Using Proxmox Backup Client)

For physical Linux servers, the `proxmox-backup-client` command-line tool is the primary method for creating and managing backups.<sup>7</sup>

Installation:

The proxmox-backup-client needs to be installed on the Linux server to be backed up. It can be installed on Debian-based systems via standard package management. A statically linked version is also available for Linux systems where the regular client might not be available, though the regular client is recommended when possible.7

Configuring the Backup Repository:

The client specifies the datastore repository on the backup server using the format \[\[username@\]server\[:port\]:\]datastore.7 The default username is

`root@pam`. If no server is specified, it defaults to `localhost`. IPv6 addresses must be enclosed in square brackets. The repository can be passed using the `--repository` command-line option, or by setting the `PBS_REPOSITORY` environment variable for convenience (e.g., `export PBS_REPOSITORY=backup-server:store1` in `.bashrc` for persistence).<sup>7</sup>

Creating Backups:

The proxmox-backup-client backup command is used to initiate backups. A basic command to back up the root directory (/) to a datastore named store1 on backup-server would be: # proxmox-backup-client backup root.pxar:/ --repository backup-server:store1.7 This command will prompt for a password and then upload a file archive named

`root.pxar`.

Handling Mount Points and Multiple Archives:

The proxmox-backup-client does not automatically include mount points by default; it will show “skip mount point” messages.7 The recommended approach is to create a separate file archive for each mounted disk or explicitly include them with

`--include-dev`. For example, to back up `/` and `/boot/efi`: `# proxmox-backup-client backup root.pxar:/ --include-dev /boot/efi --repository backup-server:store1`.<sup>7</sup> A single backup can contain multiple archives, allowing the backup of different directories or block devices in one job:

`# proxmox-backup-client backup disk1.pxar:/mnt/disk1 disk2.pxar:/mnt/disk2`.<sup>7</sup> Block devices can be backed up as

`.img` files: `# proxmox-backup-client backup mydata.img:/dev/mylvm/mydata`.<sup>7</sup>

Excluding Files/Directories:

Specific files or directories can be excluded from a backup by placing a text file named .pxarexclude in the filesystem hierarchy. Each line in this file is a glob match pattern for exclusion. Alternatively, the --exclude parameter can be used directly in the CLI: # proxmox-backup-client backup archive-name.pxar:./linux --exclude /usr.7

Change Detection Mode:

For large file-based backups, the change-detection-mode can be set to metadata to avoid re-reading unchanged files, improving efficiency. This mode encodes changed files and reuses unchanged ones from the previous snapshot, creating a split archive.7

Encryption:

Client-side encryption with AES-256 in GCM mode is supported. An encryption key can be created using # proxmox-backup-client key create my-backup.key.7 This key is then used with the

`--keyfile` parameter during backup: `# proxmox-backup-client backup etc.pxar:/etc --keyfile /path/to/my-backup.key`.<sup>7</sup> Environment variables (

`PBS_PASSWORD`, `PBS_ENCRYPTION_PASSWORD`) can be set to avoid password prompts.<sup>7</sup> For advanced key management, an RSA public/private key pair (master key) can be used to store an encrypted version of the symmetric backup encryption key alongside each backup, aiding in disaster recovery.<sup>6</sup> It is critical to keep encryption keys safe and separate from the backed-up content, as without them, backed-up files will be inaccessible.<sup>6</sup>

### B. Windows Physical Servers (Alternative Methods)

Proxmox Backup Server is primarily designed for backing up and restoring virtual machines and containers within a virtualized environment.<sup>25</sup> Currently, there is no dedicated Windows client specifically designed for direct backups to PBS.<sup>25</sup> However, alternative methods exist to back up physical Windows machines to PBS:

1.  Using a Shared Drive on a Linux Machine:
    
    This method involves sharing a drive on the Windows machine and then mounting this shared drive on a Linux machine.25 Subsequently, the Proxmox backup client, running on the Linux machine, can be used to back up data from the mounted shared drive to the PBS. This approach requires additional setup and configuration but can serve as a viable solution for file-level backups.
    
2.  Utilizing Windows Subsystem for Linux (WSL):
    
    Windows Subsystem for Linux (WSL) can be leveraged to run a Proxmox backup client directly on a Windows machine.25 This allows for backing up specific paths and folders rather than the entire operating system. This approach can be effective for targeted data protection in certain scenarios.
    
3.  Virtualizing the Windows Machine on Proxmox VE (P2V Conversion):
    
    This is a robust and effective method. The physical Windows machine is converted into a virtual machine (VM) that runs on Proxmox VE, and then PBS can seamlessly back up this VM.25
    
    - **Preparation:** Before starting the conversion, it is essential to back up all important data on the physical machine to prevent data loss in case of unexpected issues. A suitable physical-to-virtual (P2V) conversion tool should be downloaded and installed.<sup>25</sup>
        
    - **Steps:**
        
        1.  **Convert Physical Machine Disk:** Run the P2V conversion tool on the physical machine to convert its disk into a virtual machine format compatible with Proxmox (e.g., VMDK, QCOW2).<sup>25</sup>
            
        2.  **Upload Converted Disk File:** Upload the converted disk file to an appropriate storage location on the Proxmox VE server.<sup>25</sup>
            
        3.  **Create a New Virtual Machine:** Create a new virtual machine in Proxmox VE, selecting the correct operating system type and version, and then associate the uploaded disk file with this newly created VM.<sup>25</sup>
            
        4.  **Configure and Test the VM:** Configure the VM’s hardware settings (CPU cores, memory, network adapter, IP address) and start it. Install VirtIO drivers for improved performance within the VM.<sup>25</sup>
            
        5.  **Use PBS to Back Up the VM:** Once the Windows machine is running as a VM on Proxmox VE, it can be backed up using the standard PBS integration as described in Section V. This involves creating a backup task within the Proxmox VE interface, selecting the Windows VM, and configuring the backup schedule and options.<sup>25</sup>
            

For comprehensive Windows client backup solutions, specialized backup software like Vinchin Backup & Recovery is often recommended.<sup>25</sup> Such professional solutions support a wider range of features for Windows/Linux servers, virtual machines, file servers, NAS, and databases, offering enterprise-grade backup and disaster recovery capabilities, including application-aware snapshots using VSS for Windows Server.<sup>25</sup>

## VII. Advanced PBS Management and Data Lifecycle

Proxmox Backup Server provides advanced tools for managing the data lifecycle within its datastores, ensuring efficiency and reliability.

### Data Management: Prune, Sync, Verify, GC

PBS offers granular control over backup data through several dedicated management functions accessible via the web UI or CLI:

- **Prune:** The Prune function manages backup retention policies.<sup>9</sup> It allows administrators to define how many backup snapshots to keep for different intervals (e.g., hourly, daily, weekly, monthly, yearly) and a time-independent number of backups.<sup>15</sup> This ensures that old, unneeded backups are automatically removed, freeing up storage space while adhering to recovery point objectives.
    
- **Sync Jobs:** For enhanced redundancy and disaster recovery, PBS supports synchronization jobs. Datastores can be synchronized to other locations, transferring only the changes since the previous sync.<sup>2</sup> This is managed through “Remotes” and “Sync Jobs,” allowing for efficient off-site or secondary backup copies without re-transferring full datasets. This is particularly useful for implementing a 3-2-1 backup strategy.
    
- **Verify Jobs:** To ensure the integrity and restorability of backups, PBS includes Verify Jobs.<sup>4</sup> These jobs perform regular integrity checks of the data using the built-in SHA-256 checksum algorithm.<sup>5</sup> This process helps detect potential data corruption (bit rot) and confirms that backups are safe and can be reliably restored when needed.
    
- **Garbage Collection (GC):** Garbage Collection is a critical maintenance task in PBS that reclaims space occupied by pruned or deleted backups.<sup>4</sup> It involves reading and writing millions of chunk files in random order, making it highly IOPS-intensive.<sup>12</sup> GC can be configured to run periodically based on a defined schedule per datastore.<sup>15</sup> Due to its demanding nature, local SSD storage is strongly recommended for datastores to ensure efficient GC operations and prevent performance degradation of the PBS system or underlying storage.<sup>12</sup>
    

### Encryption Key Management

Proxmox Backup Server’s robust encryption capabilities rely heavily on secure key management. Client-side encryption with AES-256 in GCM mode ensures data is encrypted before it leaves the client.<sup>5</sup>

- **Key Creation and Usage:** Encryption keys can be generated using the `proxmox-backup-client key create` command.<sup>7</sup> These keys are then passed to backup commands via the
    
    `--keyfile` parameter.<sup>7</sup>
    
- **Master Key Management:** For advanced security and recovery, PBS supports the use of an RSA public/private key pair as a “master key”.<sup>5</sup> This master key can encrypt and store the symmetric backup encryption key alongside the backup itself. This means that even if the primary encryption key is lost, it can be recovered from the backup using the master key.<sup>7</sup>
    
- **Key Security Best Practices:** It is paramount to keep encryption keys safe and easily accessible, yet detached from any single system. Recommendations include storing keys in a password manager, on a secure USB flash drive, or even as a paper copy (which can be generated as a QR-encoded version using the `paperkey` subcommand).<sup>6</sup> Without the correct encryption key, backed-up files will be inaccessible and unrecoverable.<sup>6</sup>
    

## VIII. Conclusions and Recommendations

Proxmox Backup Server offers a powerful, open-source solution for comprehensive data protection within virtualized and physical environments. Its core strengths lie in efficient data storage through deduplication and compression, robust security via client-side encryption and integrity verification, and seamless integration with Proxmox VE. The client-server architecture extends its utility to backing up other Linux physical servers.

For the user’s specific setup, the following recommendations are provided:

1.  **Dedicated PBS Hardware:** To ensure high availability and prevent a single point of failure, it is strongly recommended to deploy Proxmox Backup Server on dedicated hardware, separate from the Proxmox VE host.<sup>1</sup> This separation is crucial for disaster recovery scenarios.
    
2.  **Storage Strategy for PBS Datastores:**
    
    - **Prioritize Local SSDs:** For the primary PBS datastore, particularly for demanding operations like Garbage Collection and data verification, local Solid State Drives (SSDs) on the dedicated PBS machine are highly recommended.<sup>10</sup> This will ensure optimal performance and prevent bottlenecks.
        
    - **External USB Drive on PBS:** The external USB hard drive should be connected directly to the dedicated PBS machine. While it can serve as a datastore, its performance (especially if it’s an HDD) will be limited for IOPS-intensive tasks. It may be more suitable for less critical backups or as a target for secondary sync jobs.
        
    - **Synology NAS for Secondary Storage:** While technically feasible to use the Synology NAS (via NFS or SMB/CIFS) as a primary PBS datastore, it is important to be aware of the significant performance limitations, especially if the NAS uses HDDs.<sup>12</sup> Network latency and HDD IOPS will severely impact PBS’s efficiency. The Synology NAS is better suited as a target for
        
        *remote synchronization jobs* from a local, high-performance PBS datastore, providing an off-site or secondary copy for redundancy rather than serving as the primary, active datastore.
        
3.  **Network Configuration:** Implement a dedicated network interface card (NIC) for backup traffic on the PBS machine to ensure sufficient bandwidth and prevent network congestion.<sup>11</sup>
    
4.  **Security Implementation:** Adopt a comprehensive security posture for PBS, including strong password policies, two-factor authentication, and adherence to the principle of least privilege for user accounts.<sup>4</sup> Client-side encryption should be utilized for all backups to ensure data confidentiality.<sup>5</sup>
    
5.  **Backup Other Physical Servers:** For Linux physical servers, the `proxmox-backup-client` tool offers a robust solution for incremental, deduplicated, and encrypted backups.<sup>7</sup> For Windows physical servers, consider P2V conversion to virtualize them on Proxmox VE, allowing PBS to back them up as VMs. Alternatively, use shared drives mounted on a Linux machine with the PBS client or specialized third-party backup software for direct Windows physical server backups.<sup>25</sup>
    

By carefully planning the hardware, network, and security aspects, and leveraging PBS’s native capabilities, a robust and efficient backup strategy can be established for both the Proxmox VE environment and other physical servers on the network.