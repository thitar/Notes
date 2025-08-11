# Homelab migration guide (Docker, Intel NUC WSL, Pi Zero 2, Pi 2)

You’re ready to make this clean and future‑proof. Below is a complete, opinionated guide that:

- Creates the GitHub repos
- Standardizes folder layout
- Migrates stacks (including data) host‑by‑host
- Adds an inventory.yaml you can keep as the single source of truth

I’ll assume no secrets are committed. If you want encrypted secrets (e.g., SOPS), tell me and I’ll add that.

---

## Repo strategy

- Create two private repos:
  - homelab-compose: compose files, .env.example, inventory.yaml, docs
  - homelab-scripts (optional): backup and helper scripts

You can do this in the GitHub UI or via the gh CLI. Here’s the CLI way; skip if you prefer the UI.

```bash
# On your main Linux host (recommended: Docker host)
# 1) Install GitHub CLI if needed and login
#   sudo apt-get update && sudo apt-get install -y gh
#   gh auth login

# 2) Create repos (private)
gh repo create YOURUSER/homelab-compose --private --confirm
gh repo create YOURUSER/homelab-scripts --private --confirm
```

---

## Standard layout

- Config repo working copy on each host:
  - /opt/homelab/compose/<host>/<stack>/
  - Files: docker-compose.yml, .env (live), .env.example (sanitized)
- Data (bind mounts) live here:
  - /srv/docker/data/<stack>/...
- Backwards‑compat symlink (so old habits still work):
  - /srv/docker/<stack> → /opt/homelab/compose/<host>/<stack>

Example tree (after migrating one stack):

```
/opt/homelab/compose/
  Docker/
    portainer/
      docker-compose.yml
      .env
      .env.example
  intelnuc-wsl/
  pi-zero2/
  pi2/

# data (not in Git)
 /srv/docker/data/portainer
```

---

## Git setup on each host

```bash
# Install git
sudo apt-get update && sudo apt-get install -y git

# Prepare base dirs
sudo mkdir -p /opt/homelab/compose
sudo mkdir -p /srv/docker/data
sudo chown -R $USER:$USER /opt/homelab /srv/docker

# Clone the config repo
cd /opt/homelab
git clone git@github.com:YOURUSER/homelab-compose.git
cd homelab-compose
git config user.name "Thierry"
git config user.email "YOUR_EMAIL@example.com"
```

Tip: Use SSH deploy keys per host if you want read‑only pull on the Pis:

- Generate a key: ssh-keygen -t ed25519 -C "pi2 deploy"
- Add the public key as a Deploy key in GitHub (Read‑only) for homelab-compose.

---

## Before you start: pick a platform per host

Set the target platform for images on each host to avoid pulling the wrong architecture:

- Docker (Linux VM): linux/amd64
- Intel NUC WSL: linux/amd64
- Raspberry Pi Zero 2: likely linux/arm/v7 (32‑bit) unless you’re on a 64‑bit OS
- Raspberry Pi 2: linux/arm/v7

You can put platform at the service level in docker-compose.yml:

```yaml
services:
  app:
    image: ghcr.io/some/image:latest
    platform: linux/arm/v7    # adjust per host
```

If you’re not sure about the Pi architecture, run: uname -m

---

## WSL notes for the Intel NUC

- Keep all files inside the WSL filesystem (e.g., /opt, /home). Avoid /mnt/c paths for Compose and data.
- Systemd is supported in modern WSL. Enable it if you run dockerd inside WSL:
  - /etc/wsl.conf:

    ```
    [boot]
    systemd=true
    ```

  - Then wsl --shutdown from Windows PowerShell, and reopen the distro.
- If you use Docker Desktop with WSL integration, it provides the daemon—no need to install dockerd in WSL. Just ensure your distro is enabled under Docker Desktop > Settings > Resources > WSL Integration.

I’ll refer to this host as intelnuc-wsl.

---

## Inventory file

Add this at /opt/homelab/homelab-compose/inventory.yaml. It’s a living map of “what runs where.”

```yaml
version: 1
owner: "Thierry"
default_repo: "git@github.com:YOURUSER/homelab-compose.git"

hosts:
  - name: "Docker"
    role: "primary-docker"
    os: "linux"
    arch: "amd64"
    platform: "linux/amd64"
    base_paths:
      compose: "/opt/homelab/compose/Docker"
      data: "/srv/docker/data"
    stacks:
      - name: "portainer"
        compose_file: "/opt/homelab/compose/Docker/portainer/docker-compose.yml"
        env_file: "/opt/homelab/compose/Docker/portainer/.env"
        data_dirs:
          - "/srv/docker/data/portainer"
        repo: "git@github.com:YOURUSER/homelab-compose.git"
        health: "http://docker:9000"
        notes: "Admin UI"

  - name: "intelnuc-wsl"
    role: "wsl-docker"
    os: "linux-wsl2"
    arch: "amd64"
    platform: "linux/amd64"
    base_paths:
      compose: "/opt/homelab/compose/intelnuc-wsl"
      data: "/srv/docker/data"
    stacks:
      # Add your WSL stacks here
      # Example:
      - name: "uptime-kuma"
        compose_file: "/opt/homelab/compose/intelnuc-wsl/uptime-kuma/docker-compose.yml"
        env_file: "/opt/homelab/compose/intelnuc-wsl/uptime-kuma/.env"
        data_dirs:
          - "/srv/docker/data/uptime-kuma"
        repo: "git@github.com:YOURUSER/homelab-compose.git"
        health: "http://localhost:3001"
        notes: "Monitoring"

  - name: "pi-zero2"
    role: "edge-arm"
    os: "raspbian"
    arch: "armv7"   # change to arm64 if you run 64-bit OS
    platform: "linux/arm/v7"
    base_paths:
      compose: "/opt/homelab/compose/pi-zero2"
      data: "/srv/docker/data"
    stacks: []

  - name: "pi2"
    role: "edge-arm"
    os: "raspbian"
    arch: "armv7"
    platform: "linux/arm/v7"
    base_paths:
      compose: "/opt/homelab/compose/pi2"
      data: "/srv/docker/data"
    stacks: []
```

Commit it:

```bash
cd /opt/homelab/homelab-compose
git add inventory.yaml
git commit -m "Add homelab inventory"
git push
```

---

## Data migration options (pick one per stack)

- Option A: Keep existing named volumes. Easiest, but less transparent.
- Option B (recommended): Move to bind mounts under /srv/docker/data/<stack>, so backups are trivial.

Here’s how to do both.

### A) Back up a named volume (stay on named volumes)

```bash
# Find volume name from compose or:
docker inspect <container> --format '{{json .Mounts}}' | jq

# Backup a volume to a tar.gz
VOL=my_stack_data
mkdir -p ~/vol-backups
docker run --rm -v ${VOL}:/from -v ~/vol-backups:/to alpine sh -c "cd /from && tar czf /to/${VOL}-$(date +%F).tar.gz ."
```

### B) Convert a named volume → bind mount

1) Stop the stack

```bash
cd /srv/docker/<stack>         # or the new symlink path after you create it
docker compose down
```

2) Create bind mount target

```bash
sudo mkdir -p /srv/docker/data/<stack>/<service>   # one dir per service mount
sudo chown -R $USER:$USER /srv/docker/data/<stack>
```

3) Copy the data out of the named volume

```bash
VOL=my_stack_data
DST=/srv/docker/data/<stack>/<service>
docker run --rm -v ${VOL}:/from -v ${DST}:/to alpine sh -c "cd /from && cp -a . /to"
```

4) Update docker-compose.yml: replace

```yaml
# before
volumes:
  - my_stack_data:/data
# after
volumes:
  - /srv/docker/data/<stack>/<service>:/data
```

5) Start and verify

```bash
docker compose up -d
```

---

## Host-by-host migration

We’ll repeat a safe loop: backup → move compose/env → add .env.example → symlink → test → commit.

### 1) Docker (Linux VM)

```bash
# 0) Snapshot current state
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}' > ~/running-Docker.txt
sudo tar czf ~/Docker-config-backup-$(date +%F).tar.gz /srv/docker || true

# 1) Pick ONE stack (example: portainer)
mkdir -p /opt/homelab/compose/Docker/portainer

# Move compose + env into repo path
mv /srv/docker/portainer/docker-compose.yml /opt/homelab/compose/Docker/portainer/
[ -f /srv/docker/portainer/.env ] && mv /srv/docker/portainer/.env /opt/homelab/compose/Docker/portainer/

# 2) Create sanitized .env.example
cd /opt/homelab/compose/Docker/portainer
[ -f .env ] && cp .env .env.example && sed -i 's/=.*/=/g' .env.example

# 3) Replace old folder with symlink to repo path
sudo rm -rf /srv/docker/portainer
sudo ln -s /opt/homelab/compose/Docker/portainer /srv/docker/portainer

# 4) Optional: convert volumes to bind mounts (see section above)
#    Otherwise, keep existing volumes.

# 5) Test
cd /srv/docker/portainer
docker compose pull
docker compose up -d

# 6) Commit
cd /opt/homelab/homelab-compose
mkdir -p Docker/portainer
# record the compose path relative to repo root if you want everything inside the repo
# Option 1: store files IN the repo; Option 2: keep them in /opt/... and just track them.
# Recommended: store IN the repo for portability:
#   mv /opt/homelab/compose/Docker/portainer/* /opt/homelab/homelab-compose/Docker/portainer/
#   and update the symlink to point to the repo path.
# If you prefer keeping working tree = repo path, do this instead:
sudo rm -f /srv/docker/portainer
sudo ln -s /opt/homelab/homelab-compose/Docker/portainer /srv/docker/portainer

git add Docker/portainer/docker-compose.yml Docker/portainer/.env.example inventory.yaml
git commit -m "Migrate portainer on Docker host to Git"
git push
```

Repeat for each stack on Docker until all are migrated.

Note on repo vs working path

- Cleanest is to make the repo path itself your working tree:
  - Put compose files inside /opt/homelab/homelab-compose/Docker/<stack>
  - Symlink /srv/docker/<stack> → /opt/homelab/homelab-compose/Docker/<stack>
- This way, git add works from one root and inventory paths match the repo.

Adjust the steps above accordingly (I showed both options).

---

### 2) Intel NUC (WSL: intelnuc-wsl)

Pre-checks:

- Ensure Docker Desktop WSL integration is on, or install Docker Engine in WSL with systemd.
- Keep everything off /mnt/c.

```bash
# 0) Backup
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}' > ~/running-intelnuc-wsl.txt
sudo tar czf ~/intelnuc-wsl-backup-$(date +%F).tar.gz /srv/docker || true

# 1) Clone repo if not done
cd /opt/homelab
[ -d homelab-compose ] || git clone git@github.com:YOURUSER/homelab-compose.git

# 2) Pick one stack (example: uptime-kuma)
mkdir -p /opt/homelab/homelab-compose/intelnuc-wsl/uptime-kuma

# Move compose + env into repo tree
mv /srv/docker/uptime-kuma/docker-compose.yml /opt/homelab/homelab-compose/intelnuc-wsl/uptime-kuma/
[ -f /srv/docker/uptime-kuma/.env ] && mv /srv/docker/uptime-kuma/.env /opt/homelab/homelab-compose/intelnuc-wsl/uptime-kuma/

# 3) .env.example
cd /opt/homelab/homelab-compose/intelnuc-wsl/uptime-kuma
[ -f .env ] && cp .env .env.example && sed -i 's/=.*/=/g' .env.example

# 4) Symlink
sudo rm -rf /srv/docker/uptime-kuma
sudo ln -s /opt/homelab/homelab-compose/intelnuc-wsl/uptime-kuma /srv/docker/uptime-kuma

# 5) Optional: convert volumes to bind mounts (see earlier)
# Ensure bind mounts target /srv/docker/data/uptime-kuma/...

# 6) Test
cd /srv/docker/uptime-kuma
docker compose pull
docker compose up -d

# 7) Commit
cd /opt/homelab/homelab-compose
git add intelnuc-wsl/uptime-kuma/docker-compose.yml intelnuc-wsl/uptime-kuma/.env.example inventory.yaml
git commit -m "Migrate uptime-kuma on intelnuc-wsl"
git push
```

Repeat for each WSL stack.

---

### 3) Raspberry Pi Zero 2 (pi-zero2)

```bash
# 0) Backup
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}' > ~/running-pi-zero2.txt
sudo tar czf ~/pi-zero2-backup-$(date +%F).tar.gz /srv/docker || true

# 1) Repo
cd /opt/homelab
[ -d homelab-compose ] || git clone git@github.com:YOURUSER/homelab-compose.git

# 2) One stack at a time (example)
mkdir -p /opt/homelab/homelab-compose/pi-zero2/<stack>

mv /srv/docker/<stack>/docker-compose.yml /opt/homelab/homelab-compose/pi-zero2/<stack>/
[ -f /srv/docker/<stack>/.env ] && mv /srv/docker/<stack>/.env /opt/homelab/homelab-compose/pi-zero2/<stack>/

cd /opt/homelab/homelab-compose/pi-zero2/<stack>
[ -f .env ] && cp .env .env.example && sed -i 's/=.*/=/g' .env.example

sudo rm -rf /srv/docker/<stack>
sudo ln -s /opt/homelab/homelab-compose/pi-zero2/<stack> /srv/docker/<stack>

# Ensure services specify platform: linux/arm/v7 (or arm64 if applicable)
sed -i 's/^    image:/    platform: linux\/arm\/v7\n    image:/' docker-compose.yml

cd /srv/docker/<stack>
docker compose pull
docker compose up -d

cd /opt/homelab/homelab-compose
git add pi-zero2/<stack>/docker-compose.yml pi-zero2/<stack>/.env.example inventory.yaml
git commit -m "Migrate <stack> on pi-zero2"
git push
```

---

### 4) Raspberry Pi 2 (pi2)

```bash
# 0) Backup
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}' > ~/running-pi2.txt
sudo tar czf ~/pi2-backup-$(date +%F).tar.gz /srv/docker || true

# 1) Repo
cd /opt/homelab
[ -d homelab-compose ] || git clone git@github.com:YOURUSER/homelab-compose.git

# 2) One stack at a time
mkdir -p /opt/homelab/homelab-compose/pi2/<stack>

mv /srv/docker/<stack>/docker-compose.yml /opt/homelab/homelab-compose/pi2/<stack>/
[ -f /srv/docker/<stack>/.env ] && mv /srv/docker/<stack>/.env /opt/homelab/homelab-compose/pi2/<stack>/

cd /opt/homelab/homelab-compose/pi2/<stack>
[ -f .env ] && cp .env .env.example && sed -i 's/=.*/=/g' .env.example

sudo rm -rf /srv/docker/<stack>
sudo ln -s /opt/homelab/homelab-compose/pi2/<stack> /srv/docker/<stack>

# Ensure platform lines exist for armv7
sed -i 's/^    image:/    platform: linux\/arm\/v7\n    image:/' docker-compose.yml

cd /srv/docker/<stack>
docker compose pull
docker compose up -d

cd /opt/homelab/homelab-compose
git add pi2/<stack>/docker-compose.yml pi2/<stack>/.env.example inventory.yaml
git commit -m "Migrate <stack> on pi2"
git push
```

---

## After migration: daily operations

- Update a single stack:

  ```bash
  cd /srv/docker/<stack>
  docker compose pull && docker compose up -d
  ```

- Update inventory.yaml when stacks move or change ports; commit and push.
- Backups:
  - Bind mounts: tar czf backups from /srv/docker/data/<stack>
  - Named volumes: use the alpine tar trick shown earlier
- Quick status (all hosts):
  - Maintain a tiny script in homelab-scripts to ssh and run docker ps, or use a dashboard like Portainer across endpoints.

---

## Common pitfalls to avoid

- Accidentally committing .env with real secrets. Always use .env.example for the repo.
- Running Compose from /mnt/c in WSL. Keep it inside the WSL filesystem.
- Wrong image architecture on Pis. Always set platform explicitly if images default to amd64.

---

## Want me to prefill inventory.yaml with your actual stacks?

Tell me:

- The list of stacks per host (names, ports)
- Any bind mount paths already in use
- Whether Pi Zero 2 runs 32‑bit or 64‑bit

I’ll generate a filled inventory.yaml so your “what runs where” is instantly accurate.
