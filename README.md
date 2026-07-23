# 🏠 Homelab Manual
## Podman Compose on Fedora Server

---

📌 Phase 1 - Install Fedora Server

⏱️ Estimated Time

30-45 minutes

🎯 What We're Doing

Installing Fedora Server as our base OS with Podman pre-installed for container orchestration.

🤔 Why

Fedora ships with Podman out-of-the-box. We'll use podman-compose to manage our entire stack with a single systemd service for auto-start and simplified management.

---

📝 Steps

1️⃣ Download Fedora Server ISO, create bootable USB, and boot from it

2️⃣ Follow graphical installer – choose minimal installation

3️⃣ Create a user with admin privileges during install

4️⃣ Set a static IP – either during install or via router DHCP reservation

---

🔄 After First Boot – Update Everything

```bash
sudo dnf update -y
```

🛠️ Install Essential Tools

```bash
sudo dnf install -y vim git curl wget htop policycoreutils-python-utils
```

🏷️ Set Hostname

```bash
sudo hostnamectl set-hostname podman-homelab
```

🔄 Enable Lingering (so containers start at boot without login)

```bash
sudo loginctl enable-linger $USER
```

✅ Verify

```bash
hostnamectl
podman --version
getenforce  # Should show "Enforcing"
```

---

💡 Notes

· SELinux stays in enforcing mode – Podman works great with it
· No need to enable SSH separately if configured during install
· Fedora comes with Podman out of the box – nothing extra to install 🎉

---

🖥️ Phase 2 - Cockpit Web Management

⏱️ Estimated Time

15 minutes

🎯 What We're Doing

Installing Cockpit web interface with Podman container management capabilities.

🤔 Why

Cockpit provides a GUI for system management, container monitoring, and terminal access – all without SSH.

---

📝 Steps

Install Cockpit and Podman Compose

```bash
sudo dnf install -y cockpit cockpit-podman cockpit-storaged cockpit-networkmanager podman-compose
```

Enable and Start Cockpit

```bash
sudo systemctl enable --now cockpit.socket
```

🔥 Open Firewall for Cockpit

```bash
sudo firewall-cmd --permanent --add-service=cockpit
sudo firewall-cmd --reload
```

✅ Verify

```bash
sudo systemctl status cockpit.socket
podman --version
podman-compose --version
```

🌐 Access at: https://your-server-ip:9090

🔑 Login with your regular user credentials

---

💡 Notes

· Cockpit's Podman module lets you start/stop containers and view logs
· podman-compose handles Docker Compose syntax for Podman
· All management can be done via Cockpit or SSH – you choose! 🎯

---

📂 Phase 3 - Directory Structure & Permissions

⏱️ Estimated Time

15 minutes

🎯 What We're Doing

Creating directories for persistent container storage and media, setting correct ownership and SELinux contexts.

🤔 Why

Rootless Podman needs user-owned directories. SELinux contexts allow containers to access mounted storage securely.

---

📝 Steps

📁 Create Main Directories

```bash
# Create main directories
sudo mkdir -p /mnt/docker/{config,backups}
sudo mkdir -p /mnt/{movies,tv,downloads}
```

👤 Change Ownership to Your User

```bash
sudo chown -R $USER:$USER /mnt/docker /mnt/downloads /mnt/movies /mnt/tv
```

🔒 Set Permissions

```bash
sudo chmod -R 755 /mnt/docker /mnt/downloads /mnt/movies /mnt/tv
```

🔐 Set SELinux Contexts for Container Access

```bash
sudo semanage fcontext -a -t container_file_t "/mnt/docker(/.*)?"
sudo semanage fcontext -a -t container_file_t "/mnt/downloads(/.*)?"
sudo semanage fcontext -a -t container_file_t "/mnt/movies(/.*)?"
sudo semanage fcontext -a -t container_file_t "/mnt/tv(/.*)?"
```

✅ Apply SELinux Contexts

```bash
sudo restorecon -Rv /mnt
```

🗂️ Create Config Subdirectories for Each Service

```bash
mkdir -p /mnt/docker/config/{homarr,uptime-kuma,duplicati,jellyfin,jellyseerr,prowlarr,sonarr,radarr,bazarr,sabnzbd,profilarr,memos,reclaimerr}
```

✅ Verify

```bash
ls -laZ /mnt/
# Should show: $USER:$USER ownership and container_file_t context
```

```bash
ls -la /mnt/docker/config/
# Should show all service directories
```

---

💡 Notes

· container_file_t context is crucial on Fedora – without it containers can't access storage
· All directories owned by your user for rootless Podman
· The :Z flag on volume mounts handles SELinux automatically at runtime ✨

---

📄 Phase 4 - Create Docker Compose File

⏱️ Estimated Time

10 minutes

🎯 What We're Doing

Creating a single podman-compose file with all 14 services defined.

🤔 Why

One file to rule them all – easy to edit, version control, and manage. Podman Compose handles networking between containers automatically.

---

📝 Steps

```bash
cat > ~/homelab-compose.yml << 'EOF'
version: '3'

services:
  # Dashboard
  homarr:
    image: ghcr.io/homarr-labs/homarr:latest
    container_name: homarr
    restart: unless-stopped
    ports:
      - "7575:7575"
    environment:
      TZ: America/Edmonton
      SECRET_ENCRYPTION_KEY:
    volumes:
      - /mnt/docker/config/homarr:/appdata:Z

  # Monitoring
  uptime-kuma:
    image: docker.io/louislam/uptime-kuma:2
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - /mnt/docker/config/uptime-kuma:/app/data:Z

  # Backup
  duplicati:
    image: docker.io/duplicati/duplicati:latest
    container_name: duplicati
    restart: unless-stopped
    ports:
      - "8200:8200"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /mnt/docker/config/duplicati:/data:Z
      - /mnt/docker/config:/source/config:Z
      - /mnt/docker/backups:/backups:Z

  # Media Server
  jellyfin:
    image: docker.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /mnt/docker/config/jellyfin:/config:Z
      - /mnt/movies:/movies:Z
      - /mnt/tv:/tv:Z
    ports:
      - "8096:8096"

  # Media Requests
  jellyseerr:
    image: docker.io/fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    restart: unless-stopped
    environment:
      - LOG_LEVEL=info
      - TZ=America/Toronto
    volumes:
      - /mnt/docker/config/jellyseerr:/app/config:Z
    ports:
      - "5055:5055"

  # Indexer Manager
  prowlarr:
    image: docker.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /mnt/docker/config/prowlarr:/config:Z
    ports:
      - "9696:9696"

  # TV Automation
  sonarr:
    image: docker.io/linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /mnt/docker/config/sonarr:/config:Z
      - /mnt/tv:/tv:Z
      - /mnt/downloads:/downloads:Z
    ports:
      - "8989:8989"

  # Movie Automation
  radarr:
    image: docker.io/linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /mnt/docker/config/radarr:/config:Z
      - /mnt/movies:/movies:Z
      - /mnt/downloads:/downloads:Z
    ports:
      - "7878:7878"

  # Subtitles
  bazarr:
    image: docker.io/linuxserver/bazarr:latest
    container_name: bazarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /mnt/docker/config/bazarr:/config:Z
      - /mnt/movies:/movies:Z
      - /mnt/tv:/tv:Z
    ports:
      - "6767:6767"

  # Downloader
  sabnzbd:
    image: docker.io/linuxserver/sabnzbd:latest
    container_name: sabnzbd
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /mnt/docker/config/sabnzbd:/config:Z
      - /mnt/downloads:/downloads:Z
    ports:
      - "8080:8080"

  # Profile Management
  profilarr:
    image: ghcr.io/dictionarry-hub/profilarr:latest
    container_name: profilarr
    restart: unless-stopped
    ports:
      - "6868:6868"
    volumes:
      - /mnt/docker/config/profilarr:/config:Z
    environment:
      - PUID=1000
      - PGID=1000
      - UMASK=022
      - TZ=America/Toronto
    depends_on:
      - profilarr-parser

  profilarr-parser:
    image: ghcr.io/dictionarry-hub/profilarr-parser:latest
    container_name: profilarr-parser
    restart: unless-stopped

  # Notes
  memos:
    image: docker.io/neosmemo/memos:stable
    container_name: memos
    restart: unless-stopped
    ports:
      - "5230:5230"
    environment:
      - TZ=America/Toronto
    volumes:
      - /mnt/docker/config/memos:/var/opt/memos:Z

  # Media Cleanup
  reclaimerr:
    image: docker.io/fvboegeld/reclaimerr:dev
    container_name: reclaimerr
    restart: unless-stopped
    ports:
      - "8282:8282"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /mnt/docker/config/reclaimerr:/config:Z
      - /mnt/downloads:/downloads:Z
EOF
```

✅ Verify

```bash
ls -la ~/homelab-compose.yml
cat ~/homelab-compose.yml | grep container_name | wc -l  # Should show 14
```

---

💡 Notes

· Use full image paths like docker.io/linuxserver/sonarr instead of just linuxserver/sonarr
· Reclaimerr uses the dev tag from docker.io/fvboegeld/reclaimerr:dev
· The :Z flag on volumes handles SELinux labeling automatically
· All containers share the same network automatically via compose 🌐

---

🚀 Phase 5 - Deploy the Stack

⏱️ Estimated Time

15-20 minutes (depends on internet speed)

🎯 What We're Doing

Pulling all images and starting the complete stack for the first time.

🤔 Why

Podman Compose handles pulling, networking, and starting all containers in the correct order.

---

📝 Steps

```bash
# Deploy the stack
podman-compose -f ~/homelab-compose.yml up -d
```

This will download all images – grab a coffee ☕

You'll see output as each image is pulled and each container starts

✅ Verify

```bash
# Count running containers
podman ps -q | wc -l  # Should show 14
```

```bash
# See all containers with status
podman ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

🔥 Open Firewall Ports for All Services

```bash
sudo firewall-cmd --permanent --add-port=7575/tcp
sudo firewall-cmd --permanent --add-port=3001/tcp
sudo firewall-cmd --permanent --add-port=8200/tcp
sudo firewall-cmd --permanent --add-port=8096/tcp
sudo firewall-cmd --permanent --add-port=5055/tcp
sudo firewall-cmd --permanent --add-port=9696/tcp
sudo firewall-cmd --permanent --add-port=8989/tcp
sudo firewall-cmd --permanent --add-port=7878/tcp
sudo firewall-cmd --permanent --add-port=6767/tcp
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=6868/tcp
sudo firewall-cmd --permanent --add-port=5230/tcp
sudo firewall-cmd --permanent --add-port=8282/tcp
sudo firewall-cmd --reload
```

✅ Verify Firewall

```bash
sudo firewall-cmd --list-ports
```

🔬 Test a Service Locally

```bash
curl -I http://localhost:7575
```

---

💡 Notes

· First pull takes longest – subsequent starts are instant ⚡
· If a container fails, check logs: podman logs <container-name>
· All containers auto-restart unless stopped manually

---

🔄 Phase 6 - Create Systemd Service for Auto-Start

⏱️ Estimated Time

5 minutes

🎯 What We're Doing

Creating a single systemd service that manages the entire compose stack.

🤔 Why

One command to start/stop/restart everything. Auto-starts on boot. Much simpler than managing 14 individual services.

---

📝 Steps

```bash
cat > ~/.config/systemd/user/homelab.service << 'EOF'
[Unit]
Description=Homelab Podman Compose Stack
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=%h
ExecStart=/usr/bin/podman-compose -f %h/homelab-compose.yml up -d
ExecStop=/usr/bin/podman-compose -f %h/homelab-compose.yml down

[Install]
WantedBy=default.target
EOF
```

🔄 Reload Systemd and Enable

```bash
systemctl --user daemon-reload
systemctl --user enable homelab
```

🧪 Test the Service

```bash
# Stop everything
systemctl --user stop homelab

# Verify all stopped
podman ps -q | wc -l  # Should show 0

# Start via systemd
systemctl --user start homelab

# Verify all 14 are back
sleep 10
podman ps -q | wc -l  # Should show 14
```

✅ Verify

```bash
systemctl --user status homelab
systemctl --user is-enabled homelab
```

---

💡 Notes

· RemainAfterExit=yes keeps the service showing as "active" after starting
· Restart with: systemctl --user restart homelab
· Stops cleanly with: systemctl --user stop homelab
· All containers auto-start on system boot now 🚀

---

🔄 Phase 7 - Auto-Updates

⏱️ Estimated Time

5 minutes

🎯 What We're Doing

Setting up automatic updates for both system packages and container images.

🤔 Why

Keeps everything patched and updated without manual intervention – set and forget 🧠

---

📝 Steps

📦 System Package Auto-Updates

```bash
sudo dnf install -y dnf-automatic
sudo sed -i 's/upgrade_type = default/upgrade_type = security/' /etc/dnf/automatic.conf
sudo sed -i 's/apply_updates = no/apply_updates = yes/' /etc/dnf/automatic.conf
sudo systemctl enable --now dnf-automatic.timer
```

🐳 Container Image Auto-Update

```bash
cat > ~/.config/systemd/user/podman-auto-update.service << 'EOF'
[Unit]
Description=Podman Auto-Update
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/podman auto-update
ExecStartPost=/usr/bin/podman image prune -f
EOF
```

```bash
cat > ~/.config/systemd/user/podman-auto-update.timer << 'EOF'
[Unit]
Description=Daily Podman Auto-Update

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
EOF
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now podman-auto-update.timer
```

✅ Verify

```bash
systemctl --user status podman-auto-update.timer
sudo systemctl status dnf-automatic.timer
podman auto-update --dry-run
```

---

💡 Notes

· System updates run daily for security patches 🛡️
· Container updates check for new images daily 🔍
· Run podman-compose -f ~/homelab-compose.yml pull manually to force image updates
· Then systemctl --user restart homelab to apply them

---

💾 Phase 8 - Config Backup

⏱️ Estimated Time

5 minutes

🎯 What We're Doing

Setting up daily backups of container configs and compose file.

🤔 Why

If something breaks, you can restore all your configurations quickly – disaster recovery made simple 🛟

---

📝 Steps

```bash
mkdir -p /mnt/docker/backups/config
```

```bash
cat > ~/.config/systemd/user/config-backup.service << 'EOF'
[Unit]
Description=Backup Container Configs

[Service]
Type=oneshot
ExecStart=/usr/bin/tar -czf /mnt/docker/backups/config/configs-%Y%m%d.tar.gz %h/.config/systemd/user/ %h/homelab-compose.yml /mnt/docker/config/
ExecStart=/usr/bin/find /mnt/docker/backups/config/ -name "*.tar.gz" -mtime +7 -delete
EOF
```

```bash
cat > ~/.config/systemd/user/config-backup.timer << 'EOF'
[Unit]
Description=Daily Config Backup

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
EOF
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now config-backup.timer
```

✅ Verify

```bash
systemctl --user status config-backup.timer
```

Configure Duplicati separately at http://your-server-ip:8200

For cloud backups, set up Backblaze B2 (free 10GB) in Duplicati web UI ☁️

---

💡 Notes

· Local backups keep 7 days of configs
· Duplicati handles encrypted cloud backups 🔐
· Backblaze B2 is free for up to 10GB

---

🌐 Phase 9 - Netbird Remote Access

⏱️ Estimated Time

15 minutes

🎯 What We're Doing

Installing Netbird for secure remote access via WireGuard mesh VPN with SSO.

🤔 Why

Access your services securely from anywhere without exposing ports to the internet. SSO means no managing VPN credentials – just use your Google/GitHub account! 🔑

---

📝 Steps

```bash
# Install Netbird
curl -sfL https://pkgs.netbird.io/install.sh | sudo bash
```

```bash
# Connect to Netbird Cloud (free up to 100 devices)
sudo netbird up
```

```bash
# Enable auto-start
sudo systemctl enable netbird
```

Visit https://app.netbird.io to:

1. Create an account
2. Set up SSO (Google, GitHub, Microsoft)
3. Add this server as a peer
4. Install Netbird client on your phone 📱

✅ Verify

```bash
sudo netbird status
ip addr show wt0  # WireGuard interface
```

Access services via Netbird IP instead of local IP

Example: http://netbird-ip:7575 instead of http://192.168.0.666:7575

---

💡 Notes

· Free tier supports up to 100 devices
· SSO works with Google, GitHub, Microsoft accounts 🎯
· More secure than port forwarding or Cloudflare Tunnels
· Services are only accessible to devices on your Netbird network 🛡️

---

📋 Quick Reference

🛠️ Service Management

```bash
systemctl --user status homelab          # Check stack status
systemctl --user restart homelab         # Restart everything
systemctl --user stop homelab            # Stop everything
systemctl --user start homelab           # Start everything
```

🐳 Container Management

```bash
podman ps                                # List running containers
podman ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
podman logs <container>                  # View container logs
podman exec -it <container> bash         # Enter a container
podman stats                             # Resource usage
```

🔄 Updates

```bash
podman-compose -f ~/homelab-compose.yml pull  # Pull new images
systemctl --user restart homelab              # Apply updates
```

🏥 Health Check

```bash
podman ps -q | wc -l                     # Should show 14
systemctl --user status --failed         # Check for failed services
```

---

🔗 Service URLs

- Homarr http://192.168.0.666:7575
- Uptime Kuma http://192.168.0.666:3001
- Duplicati http://192.168.0.666:8200
- Jellyfin http://192.168.0.666:8096
- Jellyseerr http://192.168.0.666:5055
- Prowlarr http://192.168.0.666:9696
- Sonarr http://192.168.0.666:8989
- Radarr http://192.168.0.666:7878
- Bazarr http://192.168.0.666:6767
- SABnzbd http://192.168.0.666:8080
- Profilarr http://192.168.0.666:6868
- Memos http://192.168.0.666:5230
- Reclaimerr http://192.168.0.666:8282
- Cockpit https://192.168.0.666:9090

---

