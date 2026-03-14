# Metabase Local Setup Guide (Docker)

This document explains how to run Metabase locally using Docker with persistent storage.
All dashboards, reports, and users will remain stored even if the container is restarted or removed.

***

# Prerequisites

Before starting, ensure the following tools are installed:

* Docker
* Docker CLI access (permission to run docker commands)
* Internet access to pull Docker images

Verify Docker installation:

```bash  theme={null}
docker --version
```

Example output:

```
Docker version 24.x.x
```

***
# Metabase Setup — Machine Name

**Server:** Machine_Name (Azure/ or any)
**Domain:** http://metabase.your_domain.com
**Date:** 2026-03-14
**OS:** Ubuntu 24.04.4 LTS

---

## Table of Contents

1. [Infrastructure Overview](#1-infrastructure-overview)
2. [What is Running & Why](#2-what-is-running--why)
3. [Directory Structure](#3-directory-structure)
4. [Docker — Metabase Container](#4-docker--metabase-container)
5. [Nginx — Reverse Proxy](#5-nginx--reverse-proxy)
6. [Data Persistence](#6-data-persistence)
7. [Memory & Performance Config](#7-memory--performance-config)
8. [Common Operations](#8-common-operations)
9. [SSL Setup (Pending)](#9-ssl-setup-pending)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Infrastructure Overview

```
Internet
    |
    v
xx.xx.xx.xxx (Azure Public IP)
    |
    v
name-Runner  (Private IP: xx.x.x.x)
    |
    v
Nginx :80  (Reverse Proxy)
    |
    v
Docker Container: metabase  (localhost:3000)
    |
    v
/opt/metabase-data/metabase.db  (SQLite — persistent storage)
```

**Why this architecture?**
- Metabase runs on port 3000 but is bound to `127.0.0.1` only — NOT directly exposed to internet
- Nginx acts as the public-facing gateway — handles routing, headers, SSL (future)
- All data lives outside the container so it survives container restarts/upgrades

---

## 2. What is Running & Why

| Component | Purpose | Why Used |
|-----------|---------|----------|
| **Docker** | Container runtime | Isolates Metabase from OS, easy upgrades |
| **metabase/metabase** | BI & reporting tool | Dashboards, reports, data queries |
| **Nginx** | Reverse proxy | Exposes Metabase on port 80/443, handles SSL |
| **SQLite (MB_DB_FILE)** | Metabase internal DB | Stores users, dashboards, saved questions |
| **Swap (2GB)** | Virtual memory | Handles memory spikes during heavy reports |

---

## 3. Directory Structure

```
/opt/metabase-data/
└── metabase.db          # SQLite database — ALL Metabase config, users, dashboards

/etc/nginx/
├── sites-available/
│   └── metabase.your_domain.com    # Nginx config for this domain
└── sites-enabled/
    └── metabase.your_domain.com -> ../sites-available/...  (symlink)

/var/log/letsencrypt/
└── letsencrypt.log      # SSL cert logs (for future SSL setup)
```

---

## 4. Docker — Metabase Container

### Full Run Command

```bash
sudo docker run -d \
  -p 127.0.0.1:3000:3000 \
  -v /opt/metabase-data:/metabase-data \
  -e MB_SITE_URL='https://metabase.your_domain.com' \
  -e MB_DB_FILE=/metabase-data/metabase.db \
  -e JAVA_OPTS="-Xmx1g -Xms256m" \
  --name metabase \
  --restart=always \
  metabase/metabase
```

### Flag-by-Flag Explanation

| Flag | Value | What it Does |
|------|-------|--------------|
| `-d` | — | Run in background (detached mode) |
| `-p 127.0.0.1:3000:3000` | localhost only | Binds container port 3000 to localhost:3000 — NOT public |
| `-v /opt/metabase-data:/metabase-data` | volume mount | Maps host folder into container for persistence |
| `MB_SITE_URL` | domain URL | Tells Metabase its own public URL (used in emails, links) |
| `MB_DB_FILE` | path inside container | Location of SQLite DB inside container |
| `JAVA_OPTS -Xmx1g` | 1GB max heap | Maximum RAM Metabase JVM can use |
| `JAVA_OPTS -Xms256m` | 256MB start heap | JVM starts with 256MB, grows up to 1GB as needed |
| `--name metabase` | — | Container name for easy reference in commands |
| `--restart=always` | — | Auto-starts on VM reboot or container crash |

---

## 5. Nginx — Reverse Proxy

### Config Location
```
/etc/nginx/sites-available/metabase.your_domain.com
```

### Current Config (HTTP only)

```nginx
server {
    listen 80;
    server_name metabase.your_domain.com;

    location / {
        proxy_pass         http://127.0.0.1:3000;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_connect_timeout 300;
        proxy_send_timeout    300;
        proxy_read_timeout    300;
        client_max_body_size  10m;
    }
}
```

### Header Explanation

| Header | Purpose |
|--------|---------|
| `Host` | Tells backend the original domain name |
| `X-Real-IP` | Passes real client IP (not nginx IP) to Metabase |
| `X-Forwarded-For` | Full chain of IPs if behind multiple proxies |
| `X-Forwarded-Proto` | Tells Metabase if request came via HTTP or HTTPS |
| `proxy_*_timeout 300` | 5 min timeout — needed for slow/heavy report queries |
| `client_max_body_size 10m` | Allows uploading files up to 10MB (CSV imports etc.) |

---

## 6. Data Persistence

Metabase data is stored at:
```
/opt/metabase-data/metabase.db
```

This includes:
- Admin users & all user accounts
- Database connections configured in Metabase
- All saved questions, dashboards, collections
- Subscriptions & alerts

**Backup command:**
```bash
sudo cp /opt/metabase-data/metabase.db /opt/metabase-data/metabase.db.backup-$(date +%Y%m%d)
```

> If container is deleted or upgraded, data is SAFE because it lives on the host, not inside the container.

---

## 7. Memory & Performance Config

### VM Resources

| Resource | Total | Allocated to Metabase |
|----------|-------|-----------------------|
| RAM | 1.86 GB | 1 GB (Xmx1g) |
| Swap | 2 GB | Fallback for spikes |
| Disk | 29 GB | ~800MB for image + DB |

### Why `-Xmx1g` and not `-Xmx512m`?

`512m` is too low for report generation — complex SQL queries or large result sets will cause `OutOfMemoryError`. With 1GB:
- OS + system processes use ~400MB
- Metabase gets 1GB heap
- ~460MB buffer remains
- Swap handles any occasional spikes

---

## 8. Common Operations

### Check if Metabase is running
```bash
sudo docker ps | grep metabase
```

### View live logs
```bash
sudo docker logs -f metabase
```

### View last 50 lines of logs
```bash
sudo docker logs --tail 50 metabase
```

### Restart Metabase
```bash
sudo docker restart metabase
```

### Stop Metabase
```bash
sudo docker stop metabase
```

### Upgrade Metabase to latest version
```bash
# 1. Backup data first
sudo cp /opt/metabase-data/metabase.db /opt/metabase-data/metabase.db.backup-$(date +%Y%m%d)

# 2. Pull new image
sudo docker pull metabase/metabase

# 3. Stop and remove old container
sudo docker stop metabase && sudo docker rm metabase

# 4. Run new container (same command)
sudo docker run -d \
  -p 127.0.0.1:3000:3000 \
  -v /opt/metabase-data:/metabase-data \
  -e MB_SITE_URL='https://metabase.your_domain.com' \
  -e MB_DB_FILE=/metabase-data/metabase.db \
  -e JAVA_OPTS="-Xmx1g -Xms256m" \
  --name metabase \
  --restart=always \
  metabase/metabase
```

### Reload Nginx (after config change)
```bash
sudo nginx -t && sudo systemctl reload nginx
```

### Check Nginx status
```bash
sudo systemctl status nginx
```

---

## 9. SSL Setup (Pending)

SSL was not configured because the ACME challenge failed.

**Root cause:** The `/.well-known/acme-challenge/` path was being proxied to Metabase (port 3000) which returned 404 instead of serving the challenge file.

**Fix when ready:**

```bash
# Step 1: Stop nginx
sudo systemctl stop nginx

# Step 2: Get certificate via standalone mode
sudo certbot certonly --standalone \
  -d metabase.your_domain.com \
  --non-interactive --agree-tos \
  -m your@email.com

# Step 3: Start nginx
sudo systemctl start nginx

# Step 4: Update nginx config to use SSL (replace existing config)
sudo tee /etc/nginx/sites-available/metabase.your_domain.com << 'EOF'
server {
    listen 80;
    server_name metabase.your_domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name metabase.your_domain.com;

    ssl_certificate     /etc/letsencrypt/live/metabase.your_domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/metabase.your_domain.com/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass         http://127.0.0.1:3000;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_connect_timeout 300;
        proxy_send_timeout    300;
        proxy_read_timeout    300;
        client_max_body_size  10m;
    }
}
EOF

# Step 5: Reload nginx
sudo nginx -t && sudo systemctl reload nginx

# Step 6: Update MB_SITE_URL — restart container
sudo docker stop metabase && sudo docker rm metabase
sudo docker run -d \
  -p 127.0.0.1:3000:3000 \
  -v /opt/metabase-data:/metabase-data \
  -e MB_SITE_URL='https://metabase.your_domain.com' \
  -e MB_DB_FILE=/metabase-data/metabase.db \
  -e JAVA_OPTS="-Xmx1g -Xms256m" \
  --name metabase \
  --restart=always \
  metabase/metabase
```

---

## 10. Troubleshooting

| Problem | Command | Fix |
|---------|---------|-----|
| Metabase not loading | `sudo docker ps` | Check if container is running |
| Slow reports / OOM | `sudo docker logs metabase \| grep -i error` | Increase `-Xmx` value |
| Nginx 502 Bad Gateway | `sudo docker ps \| grep metabase` | Container crashed — `docker start metabase` |
| Nginx not serving | `sudo systemctl status nginx` | `sudo systemctl restart nginx` |
| Disk full | `df -h` | `sudo docker system prune -f` |
| Container won't start (name conflict) | `sudo docker ps -a` | `sudo docker rm metabase` then re-run |

### Metabase startup time
Metabase takes **2-3 minutes** to fully start. Check logs:
```bash
sudo docker logs -f metabase | grep "COMPLETE"
# Wait for: Metabase Initialization COMPLETE
```
