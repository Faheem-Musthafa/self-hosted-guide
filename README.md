# 🖥️ Self-Hosted Website & Database Setup Guide

> **Stack:** Arch Linux / Ubuntu · Nginx Reverse Proxy · PostgreSQL / MySQL / PocketBase · Cloudflare Tunnel  
> No port forwarding. No exposed IP. Full HTTPS. Zero Trust.

---

## Table of Contents

- [0. Prerequisites](#0-prerequisites)
- [1. System Setup & Dependencies](#1-system-setup--dependencies)
- [2. Nginx Reverse Proxy](#2-nginx-reverse-proxy)
- [3. Database Setup](#3-database-setup)
  - [PostgreSQL](#postgresql)
  - [MySQL / MariaDB](#mysql--mariadb)
  - [PocketBase](#pocketbase-single-binary)
- [4. Cloudflare Tunnel](#4-cloudflare-tunnel)
- [5. Firewall & SSL](#5-firewall--ssl)
- [6. GitHub Webhook Auto-Deploy](#6-github-webhook-auto-deploy)
- [7. Troubleshooting](#7-troubleshooting)

---

Check this out too https://claude.ai/public/artifacts/8a0e2ef5-a562-4bde-8ea8-ac8f1429417c

---

## 0. Prerequisites

You need:
- A machine (physical, VPS, or homelab) running Arch Linux or Ubuntu/Debian
- A domain name with DNS managed by Cloudflare (free account works)
- SSH access to the machine

```bash
# Verify internet access
ping -c 2 archlinux.org

# Check architecture (should be x86_64)
uname -m

# Disk space (need 10GB+ free)
df -h /

# RAM check
free -h
```

---

## 1. System Setup & Dependencies

### Arch Linux

```bash
# Full system update
sudo pacman -Syu

# Install essentials
sudo pacman -S --needed \
  nginx git curl wget unzip \
  ufw certbot python-certbot-nginx

# Install Node.js via nvm (recommended)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc && nvm install --lts

# Enable Nginx on boot
sudo systemctl enable --now nginx
```

### Ubuntu / Debian

```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Install essentials
sudo apt install -y \
  nginx git curl wget unzip \
  ufw certbot python3-certbot-nginx \
  build-essential

# Install Node.js LTS
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs

# Enable Nginx
sudo systemctl enable --now nginx
```

> ✅ Verify: `nginx -v` and `node -v` should both return version numbers.

---

## 2. Nginx Reverse Proxy

Nginx sits in front of your app (e.g. Next.js on port 3000) and handles routing, headers, and connections.

```bash
# Create site config
sudo nano /etc/nginx/sites-available/myapp
```

Paste the following (replace `yourdomain.com` and port `3000`):

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location / {
        proxy_pass         http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection 'upgrade';
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/

# Test config (always do this before reloading)
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

> ⚠️ **Arch Linux note:** The `sites-available/sites-enabled` pattern needs manual setup. Alternatively, use `/etc/nginx/conf.d/myapp.conf` directly.

---

## 3. Database Setup

### PostgreSQL

#### Arch Linux

```bash
sudo pacman -S postgresql
sudo -u postgres initdb -D /var/lib/postgres/data
sudo systemctl enable --now postgresql
```

#### Ubuntu

```bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl enable --now postgresql
```

#### Create DB & User (both distros)

```bash
sudo -u postgres psql
```

```sql
CREATE USER myuser WITH PASSWORD 'strongpassword';
CREATE DATABASE mydb OWNER myuser;
\q
```

---

### MySQL / MariaDB

#### Arch Linux

```bash
sudo pacman -S mariadb
sudo mariadb-install-db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
sudo systemctl enable --now mariadb
sudo mysql_secure_installation
```

#### Ubuntu

```bash
sudo apt install -y mysql-server
sudo systemctl enable --now mysql
sudo mysql_secure_installation
```

#### Create DB & User

```bash
sudo mysql
```

```sql
CREATE DATABASE mydb;
CREATE USER 'myuser'@'localhost' IDENTIFIED BY 'strongpassword';
GRANT ALL PRIVILEGES ON mydb.* TO 'myuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

### PocketBase (Single Binary)

No config needed. Embedded SQLite. Perfect for solo/small projects.

```bash
# Download latest PocketBase
wget https://github.com/pocketbase/pocketbase/releases/latest/download/pocketbase_linux_amd64.zip
unzip pocketbase_linux_amd64.zip -d /opt/pocketbase
chmod +x /opt/pocketbase/pocketbase

# Create systemd service
sudo nano /etc/systemd/system/pocketbase.service
```

```ini
[Unit]
Description=PocketBase

[Service]
ExecStart=/opt/pocketbase/pocketbase serve --http=0.0.0.0:8090
Restart=always
User=www-data

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now pocketbase
# Admin UI available at: https://yourdomain.com/_/
```

---

## 4. Cloudflare Tunnel

Cloudflare Tunnel creates an **outbound-only** encrypted connection. No port forwarding, no exposed IP address. Zero Trust networking for free.

### Install `cloudflared`

#### Arch Linux

```bash
# Via AUR
yay -S cloudflared

# Or via binary
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o cloudflared
chmod +x cloudflared && sudo mv cloudflared /usr/local/bin/
```

#### Ubuntu

```bash
curl -L https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared jammy main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared
```

### Setup Tunnel (both distros)

```bash
# Authenticate (opens browser)
cloudflared tunnel login

# Create a named tunnel
cloudflared tunnel create myserver

# Create config file
nano ~/.cloudflared/config.yml
```

```yaml
tunnel: <YOUR_TUNNEL_ID>
credentials-file: /home/youruser/.cloudflared/<YOUR_TUNNEL_ID>.json

ingress:
  - hostname: yourdomain.com
    service: http://localhost:80
  - hostname: db.yourdomain.com
    service: http://localhost:8090      # PocketBase admin
  - service: http_status:404
```

```bash
# Route DNS through Cloudflare (auto-creates CNAME)
cloudflared tunnel route dns myserver yourdomain.com

# Install as systemd service
sudo cloudflared service install
sudo systemctl enable --now cloudflared

# Check status
cloudflared tunnel info myserver
```

> ✅ Your site now has HTTPS automatically — Cloudflare handles SSL termination. No certbot needed for public access.

---

## 5. Firewall & SSL

### With Cloudflare Tunnel (recommended)

Lock down inbound traffic completely. Only `cloudflared` needs outbound access.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
sudo ufw status
```

> ⚠️ Do **not** open ports 80 or 443 when using Cloudflare Tunnel. The tunnel uses outbound connections only.

### Without Cloudflare Tunnel (direct SSL)

If you're exposing your server directly:

```bash
sudo ufw allow 80
sudo ufw allow 443

# Get SSL cert via Let's Encrypt
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
# Auto-renewal is configured automatically
```

---

## 6. GitHub Webhook Auto-Deploy

Push to GitHub → webhook fires → server pulls & rebuilds. No manual SSH.

### Webhook Server (`/opt/webhook/server.js`)

```js
const http = require('http');
const { exec } = require('child_process');
const crypto = require('crypto');

const SECRET = process.env.WEBHOOK_SECRET;
const PORT = 9000;

http.createServer((req, res) => {
  if (req.method !== 'POST') { res.end(); return; }
  let body = '';
  req.on('data', d => body += d);
  req.on('end', () => {
    const sig = `sha256=` + crypto
      .createHmac('sha256', SECRET).update(body).digest('hex');
    if (req.headers['x-hub-signature-256'] !== sig) {
      res.writeHead(403); res.end(); return;
    }
    exec('cd /var/www/myapp && git pull && npm install && pm2 restart all',
      (err, out) => console.log(out || err));
    res.writeHead(200); res.end('ok');
  });
}).listen(PORT, () => console.log(`Webhook server on :${PORT}`));
```

### Run with PM2

```bash
npm install -g pm2
WEBHOOK_SECRET=yoursecret pm2 start /opt/webhook/server.js --name webhook
pm2 startup && pm2 save
```

### Add Webhook in GitHub

Go to **Repo → Settings → Webhooks → Add webhook**:
- **Payload URL:** `https://yourdomain.com/webhook`
- **Content type:** `application/json`
- **Secret:** `yoursecret` (match `WEBHOOK_SECRET` above)
- **Events:** Just the push event

Add a matching Nginx location block to proxy port 9000:

```nginx
location /webhook {
    proxy_pass http://localhost:9000;
}
```

---

## 7. Troubleshooting

### Quick Diagnostics

```bash
# Check all key services at once
sudo systemctl status nginx cloudflared postgresql

# Live Nginx error log
sudo journalctl -u nginx -f

# Live Cloudflare tunnel log
sudo journalctl -u cloudflared -f

# Test Nginx config syntax
sudo nginx -t

# Check which ports are in use
sudo ss -tlnp | grep -E '80|443|3000|8090|5432'

# Test app locally (bypassing Nginx)
curl -I http://localhost:3000

# Cloudflare tunnel info
cloudflared tunnel info myserver
```

---

### Common Issues & Fixes

| Error | Likely Cause | Fix |
|---|---|---|
| `502 Bad Gateway` | App not running on expected port | Check `pm2 list`, verify port in `proxy_pass` |
| `Tunnel not connecting` | Auth expired or wrong tunnel ID | Re-run `cloudflared tunnel login` |
| `DB connection refused` | Service not started | `sudo systemctl start postgresql` |
| `nginx: [emerg] ...` | Syntax error in config | Run `sudo nginx -t` and fix the highlighted line |
| `403 from webhook` | Secret mismatch | Check `WEBHOOK_SECRET` env var matches GitHub secret |
| `Permission denied` on files | Wrong file ownership | `sudo chown -R www-data:www-data /var/www/myapp` |
| Tunnel connects but site 404s | Ingress rule mismatch | Check `hostname` in `config.yml` matches your domain exactly |

---

### Logs Location Reference

| Service | Log Command |
|---|---|
| Nginx errors | `sudo journalctl -u nginx` or `/var/log/nginx/error.log` |
| Cloudflared | `sudo journalctl -u cloudflared -f` |
| PostgreSQL | `sudo journalctl -u postgresql` |
| PocketBase | `sudo journalctl -u pocketbase` |
| PM2 apps | `pm2 logs` |

---

## Architecture Overview

```
Internet → Cloudflare Edge (HTTPS/TLS)
                ↓ (encrypted outbound tunnel)
         cloudflared daemon (on your server)
                ↓
         Nginx (localhost:80)
                ↓
     ┌──────────┴──────────┐
     │                     │
Your App             PocketBase / DB
(localhost:3000)     (localhost:8090 / 5432)
```

---

> Built for homelab builders & indie developers. Self-host everything. Own your data.
