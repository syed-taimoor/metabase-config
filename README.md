# Metabase Local Setup Guide (Docker)

This document explains how to run Metabase locally using Docker with persistent storage.
All dashboards, reports, and users will remain stored even if the container is restarted or removed.

***

# 1. Prerequisites

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

# 2. Create a Directory for Persistent Data

Metabase stores its internal database (dashboards, reports, users, etc.) in a file.
We mount a host directory so the data survives container deletion.

Create the directory:

```bash  theme={null}
mkdir -p /home/dev-admin/metabase-data
```

Verify it exists:

```bash  theme={null}
ls /home/dev-admin
```

Expected output:

```
metabase-data
```

***

# 3. Pull the Metabase Docker Image

Download the official Metabase Docker image.

```bash  theme={null}
docker pull metabase/metabase
```

Verify image:

```bash  theme={null}
docker images
```

***

# 4. Run the Metabase Container

Run the following command to start Metabase:

```bash  theme={null}
docker run -d \
-p 3000:3000 \
-v /home/dev-admin/metabase-data:/metabase-data \
-e MB_SITE_URL='https://your_domain.com/metabase' \
-e MB_DB_FILE=/metabase-data/metabase.db \
--name metabase \
--restart=always \
metabase/metabase
```

Explanation of parameters:

| Parameter                                         | Purpose                                     |
| ------------------------------------------------- | ------------------------------------------- |
| `-p 3000:3000`                                    | Exposes Metabase web UI on port 3000        |
| `-v /home/dev-admin/metabase-data:/metabase-data` | Mounts persistent storage                   |
| `MB_SITE_URL`                                     | Public URL used for links and embeds        |
| `MB_DB_FILE`                                      | Location of Metabase internal database file |
| `--name metabase`                                 | Container name                              |
| `--restart=always`                                | Automatically restart after server reboot   |

***

# 5. Verify Container is Running

Check running containers:

```bash  theme={null}
docker ps
```

Expected output:

```
CONTAINER ID   IMAGE              PORTS
xxxxxxx        metabase/metabase  0.0.0.0:3000->3000
```

***

# 6. Access Metabase

Open the following URL in your browser:

```
http://localhost:3000
```

Or if running on a remote server:

```
https://your_domain/metabase
```

Complete the initial setup:

1. Create admin account
2. Connect your database (optional)
3. Finish onboarding

***

# 7. Data Persistence

All Metabase metadata is stored in:

```
/home/dev-admin/metabase-data/metabase.db
```

This includes:

* dashboards
* saved questions
* reports
* users
* permissions
* settings

Even if the container is deleted, the data remains in this directory.

***

# 8. Useful Docker Commands

### View container logs

```bash  theme={null}
docker logs metabase
```

### Follow logs in real time

```bash  theme={null}
docker logs -f metabase
```

### Stop container

```bash  theme={null}
docker stop metabase
```

### Start container

```bash  theme={null}
docker start metabase
```

### Restart container

```bash  theme={null}
docker restart metabase
```

### Remove container

```bash  theme={null}
docker rm -f metabase
```

Note: Removing the container **will not delete the stored data** because it is stored in `/home/dev-admin/metabase-data`.

***

# 9. Recreate Container (If Needed)

If the container is removed or recreated, run the same command again:

```bash  theme={null}
docker run -d \
-p 3000:3000 \
-v /home/dev-admin/metabase-data:/metabase-data \
-e MB_SITE_URL='https://api-admin-release.byotautoparts.com/metabase' \
-e MB_DB_FILE=/metabase-data/metabase.db \
--name metabase \
--restart=always \
metabase/metabase
```

Metabase will automatically load the existing database file.

***

# 10. Backup Recommendation

Backup the persistent directory regularly:

```bash  theme={null}
tar -czvf metabase-backup.tar.gz /home/dev-admin/metabase-data
```

This ensures dashboards and reports can be restored if needed.

***

# 11. Troubleshooting

### Container not starting

Check logs:

```bash  theme={null}
docker logs metabase
```

### Port already in use

Check running services:

```bash  theme={null}
sudo lsof -i :3000
```

Then stop the conflicting service or change the port mapping.

***

# Summary

Architecture overview:

```
Server
│
├── Docker Container (Metabase)
│
└── Host Storage
    /home/dev-admin/metabase-data
        └── metabase.db
```

The container can be restarted or recreated without losing dashboards or reports because the data is stored on the host machine.
