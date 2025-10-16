# 🧭 Semaphore (Self-hosted) Installation on Proxmox 🎯

This guide explains how to install and run a self-hosted Semaphore (CI/CD) instance inside Proxmox. It covers creating a VM or LXC, basic networking, Docker setup, and running Semaphore using Docker Compose. Use this as a concise reference for a small homelab deployment. ⚙️

> Assumptions:
> - You have Proxmox VE up and running with access to the web UI. 🖥️
> - You have basic familiarity with Proxmox VM/LXC creation, SSH, and Linux CLI. 🧑‍💻
> - This guide targets Ubuntu 22.04 LTS for VMs or unprivileged LXC containers. 🐧

---

## 🔹 Quick overview

- Create a VM (recommended) or unprivileged LXC for better isolation. 🧱
- Install Docker and Docker Compose. 🐳
- Deploy Semaphore (server + worker) with Docker Compose. 🚀
- Configure external access (reverse proxy / SSL) if required. 🔒

---

## 1️⃣ Choose VM or LXC

- VM (recommended): Full isolation, easier compatibility. Choose Ubuntu 22.04 cloud image or ISO. ✅
- LXC (possible): Lower overhead but requires proper privileges for Docker. Use unprivileged LXC and enable nesting if you plan to run Docker. ⚠️

Notes:
- For LXC, set "Features -> Nesting: Yes" in the Proxmox container options. Also add these to the container config if needed:

```
features: nesting=1
lxc.apparmor.profile: unconfined
lxc.cgroup.devices.allow: a
```

---

## 2️⃣ Create the VM (recommended) 🛠️

1. In Proxmox web UI, click `Create VM`.
2. Select an ISO or cloud image (Ubuntu 22.04 LTS recommended). 🗳️
3. Resources:
   - CPU: 2 vCPU (min) 🔁
   - RAM: 2 GB (4 GB recommended) 🧠
   - Disk: 20 GB (or larger) 💾
4. Network: bridge to your LAN bridge (e.g., vmbr0). 🌐
5. Start the VM and install Ubuntu, enable SSH. 🔑

---

## 3️⃣ Basic OS setup 🔧

SSH into the VM/container and run:

```bash
# update
sudo apt update && sudo apt upgrade -y

# install essentials
sudo apt install -y ca-certificates curl gnupg lsb-release
```

Set timezone, create a user, and ensure SSH key access. 🔑

---

## 4️⃣ Use the Proxmox VE helper script (recommended) 🧩

Instead of a bespoke manual setup, prefer the Proxmox VE community helper script for Semaphore. This repository contains tested automation scripts tailored for Proxmox to create and provision VMs or containers for many projects — including Semaphore. Using the helper script reduces manual steps and ensures your VM/LXC is provisioned consistently. 🔁

1. Open the helper script page in your browser and review the script and documentation:

- https://community-scripts.github.io/ProxmoxVE/scripts?id=semaphore&category=Automation+%26+Scheduling 🔗

2. From the Proxmox host (not inside a guest), fetch and inspect the script before running it. Replace `<RAW_SCRIPT_URL>` with the raw script URL you find on the page (always review the script first):

```bash
# example — replace with the actual raw script URL from the helper page
curl -fsSL -o /tmp/semaphore-proxmox.sh '<RAW_SCRIPT_URL>'

# inspect the script before running
less /tmp/semaphore-proxmox.sh

# when you're satisfied, run it with sudo
sudo bash /tmp/semaphore-proxmox.sh
```

3. What the helper script typically does:

- Creates a VM or LXC with recommended resources and cloud-init templates. 🖼️
- Installs required packages (or provisions a cloud-init user-data) and configures networking. 🌐
- Optionally installs Docker and deploys services or places files in the guest for you to finish the app setup. 🐳

4. After the script finishes:

- Check the Proxmox web UI for the new VM/LXC. 🖥️
- SSH into the new guest and verify services (for example, check Docker, containers, or that web UI is accessible). 🔎

5. Important safety notes:

- Always inspect community scripts before running them on your Proxmox host. ✅
- Prefer running scripts from the Proxmox host (shell access) rather than piping to sh from an unknown source. 🛡️

---

## 5️⃣ Manual Docker Compose (optional fallback) 📦

If you prefer to provision a VM/LXC manually, or want a fallback approach, you can still use Docker Compose inside the guest. The instructions below are an optional manual path and were the original approach in this guide.

Create a working directory, e.g., `/opt/semaphore`:

```bash
mkdir -p /opt/semaphore && cd /opt/semaphore
```

Create `docker-compose.yml` (example minimal):

```yaml
version: '3.8'
services:
  semaphore-server:
    image: semaphoreci/semaphore:latest
    container_name: semaphore-server
    restart: unless-stopped
    ports:
      - "3000:3000"    # web UI
      - "4000:4000"    # API (example)
    volumes:
      - ./semaphore-data:/var/lib/semaphore
    environment:
      - SEQUELIZE_DATABASE=semaphore
      - SEQUELIZE_USERNAME=semaphore
      - SEQUELIZE_PASSWORD=secret
      - SEQUELIZE_HOST=db

  db:
    image: postgres:15
    container_name: semaphore-db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=semaphore
      - POSTGRES_USER=semaphore
      - POSTGRES_PASSWORD=secret
    volumes:
      - ./pgdata:/var/lib/postgresql/data
```

Notes:
- The official Semaphore image name and env vars may differ by version—check Semaphore's official docs and change image/tags accordingly. 🔎
- Use strong passwords and consider external secrets storage for production. 🔐

Start the stack:

```bash
docker compose up -d
```

Check logs:

```bash
docker compose logs -f
```


## 6️⃣ Configure reverse proxy & SSL (optional but recommended) 🔁

Use Nginx or Caddy as a reverse proxy. Example with Caddy (automatic HTTPS):

```yaml
# Example additional service in docker-compose.yml
caddy:
  image: caddy:latest
  restart: unless-stopped
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - ./Caddyfile:/etc/caddy/Caddyfile
    - caddy_data:/data
    - caddy_config:/config
```

`Caddyfile` example:

```
semaphore.example.com {
  reverse_proxy semaphore-server:3000
}
```

Make sure DNS for `semaphore.example.com` points to your Proxmox host's public IP (or use LAN DNS for internal use). 🌍

---

## 7️⃣ Workers / Runners 🔁

Semaphore may run separate worker containers or use built-in runners. Follow Semaphore's docs to configure runners that connect to your server. Usually this involves running an agent container that registers with the server using a token. 🔐

---

## 8️⃣ Backups & Persistence 💾

- Persist PostgreSQL data (`./pgdata`) and Semaphore data (`./semaphore-data`).
- Snapshot the VM using Proxmox before major upgrades. ⏳
- Consider scheduled `pg_dump` backups and off-host copies. ☁️

---

## 9️⃣ Troubleshooting 🐞

- Docker container not starting: `docker compose ps` and `docker compose logs <service>`.
- DB connection errors: check env vars and `docker exec -it semaphore-db psql -U semaphore -d semaphore`.
- Port conflicts: confirm host ports (3000/80/443) not in use.

---

## 🔚 Verification Checklist ✅

- [ ] VM or LXC created and reachable via SSH. 🔌
- [ ] Docker + Docker Compose installed and tested. 🐳
- [ ] Semaphore containers running: `docker compose ps`. 🚦
- [ ] Web UI accessible on port 3000 or via a reverse-proxied domain. 🌐
- [ ] Backups configured. 💾

---

## 📚 References & Notes 📝

- Semaphore official docs: https://semaphoreci.com/docs (verify current image names and deployment recommendations) 🔗
- Proxmox docs: https://pve.proxmox.com/wiki/Main_Page 🔗

---

If you'd like, I can: 
- add a ready-to-run `docker-compose.yml` with the current official Semaphore image tag (I will check the latest image name). 🔎
- create a Proxmox VM template cloud-init example. ☁️

Happy to continue and tailor the doc to your environment! ✨
