# Homelab Organization Guide (Revised)  

**Goal:** Document your homelab and reorganize Docker stacks **while keeping containers on their original hosts**.  
**Tools:** Git, Markdown, Docker Compose, SSH.  

---

## **Part 1: Documentation Setup**  

**Solution:** Central Git repo for documentation (hosted on GitHub/GitLab) with Markdown files.  

### **Step 1: Create Documentation Repo**  

1. **On your main machine (e.g., Proxmox Docker VM):**  

   ```bash
   # Install Git
   sudo apt update && sudo apt install git -y

   # Create repo folder
   mkdir ~/homelab-docs && cd ~/homelab-docs
   git init
   ```

2. **Create basic structure:**  

   ```bash
   touch README.md                 # Homelab overview
   touch DEVICES.md                # Hardware specs
   touch NETWORK.md                # Network diagram
   touch SERVICES.md               # Service list
   mkdir -p hosts/{docker-vm,nuc,rpi2,pizero}  # Host-specific docs
   ```

3. **Example `hosts/docker-vm.md`:**  

   ```markdown
   ## Proxmox Docker VM
   - **IP:** 192.168.1.10
   - **Services:**
     - Portainer: `http://portainer.lab`
     - Nginx Proxy Manager
   - **Compose Path:** `/srv/docker`
   ```

### **Step 2: Commit & Push Docs**  

```bash
git add .
git commit -m "Initial documentation"
git remote add origin https://github.com/your-username/homelab-docs.git
git push -u origin main
```

---

## **Part 2: Docker Stack Reorganization**  

**Key Principle:** Keep containers on original hosts, but store all `docker-compose.yml` files in Git.  

### **Step 1: Create Docker Management Repo**  

**On your Proxmox Docker VM:**  

```bash
mkdir ~/homelab-docker && cd ~/homelab-docker
git init
```

### **Step 2: Folder Structure**  

Create host-specific folders:  

```bash
mkdir -p hosts/docker-vm/          # Proxmox Docker VM
mkdir -p hosts/nuc/                # Intel NUC
mkdir -p hosts/rpi2/               # Raspberry Pi 2
# Pi Zero doesn't use compose - document in homelab-docs instead
```

### **Step 3: Move Compose Files (Without Moving Containers)**  

1. **For Proxmox Docker VM** (containers stay here):  

   ```bash
   # Copy compose files to Git repo
   cp /srv/docker/*/docker-compose.yml ~/homelab-docker/hosts/docker-vm/
   ```

2. **For Intel NUC (WSL):**  
   - **On NUC's WSL terminal:**  

     ```bash
     # Copy compose files to Proxmox VM via SSH
     scp /opt/*/docker-compose.yml user@proxmox-ip:~/homelab-docker/hosts/nuc/
     ```

3. **For Raspberry Pi 2:**  
   - **On Pi 2:**  

     ```bash
     # Copy compose files to Proxmox VM
     scp /path/to/your/compose/*.yml user@proxmox-ip:~/homelab-docker/hosts/rpi2/
     ```

### **Step 4: Verify Volume Paths**  

**Critical:** Ensure compose files point to existing data paths.  
Example for NUC's service in `hosts/nuc/service1/docker-compose.yml`:  

```yaml
version: '3'
services:
  your_service:
    image: your/image
    volumes:
      # MUST match original path on NUC
      - /opt/service1/data:/data
```

### **Step 5: Start Containers from New Locations**  

1. **On each host**, clone the repo:  

   ```bash
   git clone https://github.com/your-username/homelab-docker.git
   ```

2. **Run containers from their host folder:**  
   - **On Proxmox Docker VM:**  

     ```bash
     cd ~/homelab-docker/hosts/docker-vm/service1
     docker-compose up -d  # Uses same local data
     ```

   - **On Intel NUC (WSL):**  

     ```bash
     cd ~/homelab-docker/hosts/nuc/service2
     docker-compose up -d
     ```

### **Step 6: Commit & Push Compose Files**  

**On Proxmox Docker VM:**  

```bash
cd ~/homelab-docker
git add .
git commit -m "Add all host compose files"
git push origin main
```

---

## **Part 3: Git Workflow for Beginners**  

### **Basic Commands Cheat Sheet**  

| Command | Description | Example |
|---------|-------------|---------|
| `git clone <url>` | Download repo | `git clone https://github.com/you/homelab.git` |
| `git pull` | Get latest changes | `git pull` before editing |
| `git add <file>` | Stage changes | `git add hosts/nuc/new-service.yml` |
| `git commit -m "msg"` | Save changes | `git commit -m "Added NUC service"` |
| `git push` | Upload changes | `git push` after commit |
| `git status` | Check changed files | `git status` |

### **Typical Workflow:**  

1. **Before editing:**  

   ```bash
   cd ~/homelab-docker
   git pull  # Get latest version
   ```

2. **Edit files:**  

   ```bash
   nano hosts/rpi2/new-service/docker-compose.yml
   ```

3. **Save & upload:**  

   ```bash
   git add hosts/rpi2/new-service/docker-compose.yml
   git commit -m "Added new RPi2 service"
   git push
   ```

---

## **Part 4: Structure Overview**  

```
homelab-docs/                   # Documentation
├── DEVICES.md
├── NETWORK.md
├── SERVICES.md
└── hosts/
    ├── docker-vm.md            # Proxmox Docker VM details
    ├── nuc.md                  # Intel NUC details
    └── rpi2.md                 # RPi2 details

homelab-docker/                 # Docker management
└── hosts/
    ├── docker-vm/              # Proxmox Docker VM services
    │   ├── portainer/
    │   │   └── docker-compose.yml
    │   └── nginx-proxy/
    │       └── docker-compose.yml
    ├── nuc/                    # Intel NUC services
    │   └── service1/
    │       └── docker-compose.yml
    └── rpi2/                   # RPi2 services
        └── service2/
            └── docker-compose.yml
```

---

## **Part 5: Maintenance & Best Practices**  

1. **Starting/Stopping Services:**  
   Always use compose files from the repo:  

   ```bash
   cd ~/homelab-docker/hosts/docker-vm/your-service
   docker-compose up -d    # Start
   docker-compose down     # Stop
   ```

2. **Updating Services:**  
   - Edit compose file in repo  
   - Run `docker-compose up -d` (pulls new images if needed)  

3. **Backup Strategy:**  
   - **Compose files:** Already in Git  
   - **Container data:** Back up host volumes separately  

     ```bash
     # Example: Backup Proxmox Docker VM data
     rsync -av /srv/docker/ /mnt/backup/docker-vm/
     ```

4. **Adding New Services:**  
   - Create folder in host directory  
   - Add `docker-compose.yml`  
   - Run `docker-compose up -d` on target host  
   - Commit/push to Git  

5. **Syncing Changes to Hosts:**  
   Periodically update local repos:  

   ```bash
   # On each host (except Pi Zero)
   cd ~/homelab-docker
   git pull
   ```

---

## FAQ  

**Q: What about my Pi Zero (no Docker Compose)?**  
A: Document its config in `homelab-docs/hosts/pizero.md` (e.g., Pi-hole settings, DHCP config).  

**Q: How to handle secrets (passwords, API keys)?**  
A: Use Docker secrets or `.env` files:  

1. Create `.env` in service directory:  

   ```ini
   DB_PASSWORD=secret123
   ```

2. Reference in compose:  

   ```yaml
   environment:
     - DB_PASSWORD=${DB_PASSWORD}
   ```

3. **Add `.env` to `.gitignore** so secrets don't go to GitHub!  

**Q: What if I make a mistake in Git?**  
A: Revert changes with:  

```bash
git checkout -- path/to/file.yml  # Discard local changes
git reset HEAD~1                 # Undo last commit (before push)
```

This setup gives you:  

- 📝 Centralized documentation  
- 🐳 Docker compose files under version control  
- ⚙️ Containers remain on original hosts  
- 🔄 Easy replication/recovery via Git  
- 🔒 No data loss (volumes remain untouched)  

Start small - document one device/service at a time!
