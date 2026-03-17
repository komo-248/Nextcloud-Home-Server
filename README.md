# Nextcloud Server – Inovato Quadra

A self-hosted personal cloud storage server running on an Inovato Quadra single-board computer. Built using Docker and Portainer for container management, Cloudflare Tunnel for secure external access without port forwarding, and Uptime Kuma for uptime monitoring.

---

## Table of Contents

- [Overview](#overview)
- [Hardware](#hardware)
- [Software Stack](#software-stack)
- [Setup](#setup)
  - [1. OS Installation (Armbian to eMMC)](#1-os-installation-armbian-to-emmc)
  - [2. External Storage Setup](#2-external-storage-setup)
  - [3. Docker & Portainer](#3-docker--portainer)
  - [4. Nextcloud & MariaDB (Docker Compose)](#4-nextcloud--mariadb-docker-compose)
  - [5. Cloudflare Tunnel (Zero Trust)](#5-cloudflare-tunnel-zero-trust)
  - [6. Nextcloud Configuration](#6-nextcloud-configuration)
  - [7. Cron Jobs & Uptime Kuma](#7-cron-jobs--uptime-kuma)
  - [8. Finalize & External Storage App](#8-finalize--external-storage-app)
- [Troubleshooting](#troubleshooting)
  - [File Locking Issues](#file-locking-issues)
  - [Deadlocked Files (SQL Fix)](#deadlocked-files-sql-fix)

---

## Overview

This project sets up a fully functional private cloud accessible from anywhere over HTTPS using a Cloudflare Tunnel — no open ports or port forwarding required on the home router. All services run as Docker containers managed through a Portainer web UI, making updates and monitoring straightforward.

**Key features:**
- Remote file access via browser or Nextcloud desktop/mobile client
- Secure HTTPS via Cloudflare Tunnel with a custom domain (no exposed ports)
- Containerized stack — Nextcloud, MariaDB, and Uptime Kuma each in isolated containers
- External SSD for data storage (keeps OS drive separate from data)
- APCu local memory caching for improved performance
- Uptime Kuma monitoring dashboard with cron job health checks

---

## Hardware

| Component | Details |
|-----------|---------|
| **Board** | Inovato Quadra (ARM SBC) |
| **OS Storage** | eMMC (internal) — OS migrated here after initial microSD boot |
| **Data Storage** | External USB SSD or flash drive (formatted ext4) |
| **Network** | Ethernet (recommended for stability) |

> The Quadra boots from microSD for the initial install, then the OS is migrated to the faster internal eMMC. The microSD is removed after that.

---

## Software Stack

| Layer | Technology |
|-------|-----------|
| **OS** | Armbian (Debian-based, headless) |
| **Container Runtime** | Docker |
| **Container UI** | Portainer CE |
| **Database** | MariaDB (Docker container) |
| **Application** | Nextcloud (Docker container) |
| **Tunneling** | Cloudflare Zero Trust Tunnel |
| **Caching** | PHP APCu |
| **Monitoring** | Uptime Kuma (Docker container) |

---

## Setup

### 1. OS Installation (Armbian to eMMC)

The Quadra ships with a default image, but flashing a fresh Armbian image to a microSD and then migrating it to eMMC gives a clean, up-to-date base.

**References:**
- https://www.inovato.net/Armbian/
- https://forum.inovato.com/post/doing-a-factory-restore-12449421

**Steps:**

1. Download [Balena Etcher](https://www.balena.io/etcher/) and the latest Armbian image for the Quadra.
2. Flash the Armbian image to a microSD card using Etcher.
3. Unplug the Quadra completely, insert the microSD, then plug power back in.
4. Wait for a solid blue LED — this indicates a successful boot.
5. Connect a monitor, keyboard, and mouse (leave the microSD inserted).
6. Log in as `root` and launch the Armbian config tool:

```bash
armbian-config
```

7. Navigate to **Install → Install on eMMC** to migrate the OS to internal storage.
8. Once complete, power off, **remove the microSD card**, and reboot.
9. Update the system:

```bash
apt update
apt full-upgrade -y
reboot
```

> After this point, the Quadra boots entirely from eMMC. The microSD card is no longer needed.

---

### 2. External Storage Setup

An external USB SSD or flash drive is used as the Nextcloud data volume. This keeps data off the eMMC, extends its lifespan, and allows for larger storage capacity.

**Reference:** https://www.youtube.com/watch?v=D8eDnlGDUk4

```bash
# Identify the drive
lsblk
```

Find the device name (typically `/dev/sda`). Then partition and format it:

```bash
# Install fdisk if not present
sudo apt install fdisk -y

# Partition the drive
sudo fdisk /dev/sda
# Inside fdisk:
#   d  — delete existing partitions
#   n  — new partition (accept all defaults)
#   w  — write and exit
```

```bash
# Install filesystem tools and format as ext4
sudo apt-get install -y util-linux
sudo mkfs -t ext4 /dev/sda1
```

Mount the drive and set permissions:

```bash
sudo mkdir /mnt/NextcloudStorage
sudo mount /dev/sda1 /mnt/NextcloudStorage
sudo chown -R www-data:www-data /mnt/NextcloudStorage
sudo chmod -R 0750 /mnt/NextcloudStorage
```

Make the mount persistent across reboots by adding it to `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Add this line at the bottom:

```
/dev/sda1    /mnt/NextcloudStorage    ext4    defaults    0    0
```

Reboot and verify the drive is still mounted:

```bash
reboot
lsblk  # Confirm /dev/sda1 is mounted at /mnt/NextcloudStorage
```

---

### 3. Docker & Portainer

Docker handles all containerized services. Portainer provides a browser-based GUI to manage containers, stacks, and volumes without needing to use the command line for routine tasks.

**Reference:** https://www.youtube.com/watch?v=O7G3oatg5DA

```bash
# Install Docker
curl -sSL https://get.docker.com | sh

# Add your user to the docker group (replace 'homeserver' with your username)
sudo usermod -aG docker homeserver

# Pull and run Portainer CE
sudo docker pull portainer/portainer-ce

sudo docker run -d \
  -p 9000:9000 \
  --name=portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce
```

Access the Portainer UI by navigating to `http://192.168.1.xxx:9000` on your local network and completing the initial setup wizard.

---

### 4. Nextcloud & MariaDB (Docker Compose)

Nextcloud and its MariaDB database are deployed together as a Portainer stack using Docker Compose. The Nextcloud data directory is mapped directly to the external drive mounted in Step 2.

**References:**
- https://www.youtube.com/watch?v=Qcsi9yvSHgU
- https://www.youtube.com/watch?v=aIBTbsk7rnA

In Portainer, go to **Stacks → Add Stack**, name it `mariadb_nextcloud`, and paste the following compose definition:

```yaml
version: '2'

volumes:
  nextcloud:
  db:

services:
  db:
    image: mariadb
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    restart: always
    volumes:
      - db:/var/lib/mysql
    container_name: mariadb
    environment:
      - MYSQL_ROOT_PASSWORD=<your_root_password>
      - MYSQL_PASSWORD=<your_db_password>
      - MYSQL_DATABASE=<your_db_name>
      - MYSQL_USER=<your_db_user>

  app:
    image: nextcloud
    environment:
      - MYSQL_PASSWORD=<your_db_password>
      - MYSQL_DATABASE=<your_db_name>
      - MYSQL_USER=<your_db_user>
      - MYSQL_HOST=mariadb
    ports:
      - 9321:80
    links:
      - db
    container_name: nextcloud
    volumes:
      - /mnt/NextcloudStorage/nextcloud:/var/www/html
      - /mnt/NextcloudStorage/app/data:/var/www/html/data
    restart: always
```

> Replace all `<placeholder>` values with your own credentials before deploying. The `MYSQL_HOST` value of `mariadb` matches the `container_name` of the db service — Docker's internal DNS resolves container names automatically.

Deploy the stack, then navigate to `http://192.168.1.xxx:9321` to complete the Nextcloud setup wizard using the database credentials defined above.

---

### 5. Cloudflare Tunnel (Zero Trust)

Rather than opening ports on the router, a Cloudflare Tunnel creates an outbound-only encrypted connection between the server and Cloudflare's edge network. This means no port forwarding, no exposed public IP, and built-in DDoS protection.

**Reference:** https://www.youtube.com/watch?v=p0I8pikm2P4

**Steps:**

1. Log into [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) and navigate to **Access → Tunnels → Add a Tunnel**.
2. Name the tunnel. The setup page will generate a `docker run` command with a unique tunnel token.
3. SSH into the Quadra and run the provided command:

```bash
docker run cloudflare/cloudflared:latest tunnel --no-autoupdate run --token "<your_tunnel_token>"
```

4. In Portainer, find the newly created `cloudflared` container and set its **restart policy to "Always"** so it survives reboots.
5. Back in Cloudflare Zero Trust, continue the setup and configure the public hostname:

| Setting | Value |
|---------|-------|
| Subdomain | `yoursubdomain` (e.g. `thecloud`) |
| Domain | `yourdomain.com` |
| Service Type | `http` |
| URL | `192.168.1.xxx:9321` |
| Happy Eyeballs | Disabled |
| Chunk Encoding | Disabled |

> Disabling Happy Eyeballs and Chunk Encoding prevents proxying issues that can cause Nextcloud uploads to stall or fail through the tunnel.

At this point, navigating to `https://yoursubdomain.yourdomain.com` will show a Nextcloud **"Untrusted Domain"** error — this is expected and resolved in the next step.

---

### 6. Nextcloud Configuration

With the tunnel active, Nextcloud needs to be told to trust the custom domain and properly handle HTTPS when sitting behind a reverse proxy.

**References:**
- https://www.youtube.com/watch?v=p0I8pikm2P4
- https://dbt3ch.com/books/nextcloud-with-cloudflare-tunnels/page/nextcloud-with-cloudflare-tunnels

SSH into the server, then open a shell inside the Nextcloud container:

```bash
docker exec -it nextcloud bash
apt update && apt install nano -y
```

**Increase upload limits** — edit `.htaccess` inside the container:

```bash
nano .htaccess
```

Add the following to the top of the file:

```apache
php_value upload_max_filesize 16G
php_value post_max_size 16G
php_value max_input_time 3600
php_value max_execution_time 3600
php_value memory_limit 2048M
```

Save: `CTRL+O` → `CTRL+X`

**Update `config/config.php`** to add the trusted domain and HTTPS settings:

```bash
nano config/config.php
```

Find the `trusted_domains` array and add your public domain:

```php
'trusted_domains' =>
  array (
    0 => '192.168.1.xxx:9321',
    1 => 'yoursubdomain.yourdomain.com',
  ),
```

Scroll to the bottom and add the following before the closing `);`:

```php
'overwriteprotocol' => 'https',
'default_phone_region' => 'US',
'enable_previews' => true,
'check_for_working_wellknown_setup' => false,
```

> `overwriteprotocol` is critical when running behind a reverse proxy or tunnel — it forces Nextcloud to generate `https://` links internally rather than `http://`. Without it, many features (sharing links, app sync) will break. `check_for_working_wellknown_setup` suppresses the `.well-known` warning that appears when using Cloudflare Tunnel.

**Configure WebDAV/CardDAV/CalDAV redirects** in Apache:

```bash
nano /etc/apache2/sites-enabled/000-default.conf
```

Add to the end of the file:

```apache
Redirect 301 /.well-known/carddav https://yoursubdomain.yourdomain.com/remote.php/dav
Redirect 301 /.well-known/caldav https://yoursubdomain.yourdomain.com/remote.php/dav
Redirect 301 /.well-known/webdav https://yoursubdomain.yourdomain.com/remote.php/dav
Redirect 301 /.well-known/webfinger /nextcloud/index.php/.well-known/webfinger
Redirect 301 /.well-known/nodeinfo /nextcloud/index.php/.well-known/nodeinfo
Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains; preload"
```

Restart the Nextcloud container in Portainer to apply all changes.

---

### 7. Cron Jobs & Uptime Kuma

Nextcloud's background tasks (cleanup, notifications, file indexing) run more reliably via system cron than the default browser-triggered AJAX method. Uptime Kuma provides a self-hosted monitoring dashboard to alert if the server or cron job goes down.

**Switch background jobs to Cron:**

In the Nextcloud admin panel: **Settings → Basic Settings → Background Jobs → select Cron**.

**Install APCu and configure memory caching:**

Inside the Nextcloud container:

```bash
docker exec -it nextcloud bash
apt-get install php-apcu -y
```

Add to `config/config.php`:

```php
'memcache.local' => '\OC\Memcache\APCu',
```

**Set up the cron job** on the host system:

```bash
sudo crontab -u www-data -e
# Select nano (option 1) if prompted
```

Add this line at the end:

```
*/5  *  *  *  * /usr/bin/php7.4 -f /var/www/html/cron.php --define apc.enable_cli=1
```

**Deploy Uptime Kuma:**

In Portainer, add a new stack named `uptimekuma` with the following:

```yaml
version: '3.3'

volumes:
  uptimekuma:

services:
  uptime-kuma:
    image: louislam/uptime-kuma
    container_name: uptime-kuma
    volumes:
      - uptimekuma:/app/data
    ports:
      - 3001:3001
```

Navigate to `http://192.168.1.xxx:3001`, create an admin account, then add a monitor:

| Setting | Value |
|---------|-------|
| Type | HTTP(s) |
| Name | Nextcloud Cron |
| URL | `https://yoursubdomain.yourdomain.com/cron.php` |
| Heartbeat Interval | 300 seconds |
| Retry Interval | 120 seconds |

---

### 8. Finalize & External Storage App

1. Log into Nextcloud as admin.
2. Go to **Apps → Your Apps** and enable **External Storage Support**.
3. Navigate to **Settings → External Storages** and configure:
   - Add a folder pointing to `/data`
   - Add a second folder named `Nextcloud Storage` pointing to `/data-NextcloudStorage`

**Optional — Email notifications via Gmail SMTP:**

Go to **Settings → Basic Settings → Email Server** and configure:

| Setting | Value |
|---------|-------|
| Send Mode | SMTP |
| Encryption | SSL/TLS |
| From Address | your Gmail address |
| Authentication | Login (required) |
| Server | `smtp.gmail.com` |
| Port | `465` |
| Password | Google App Password |

> Use a [Google App Password](https://myaccount.google.com/apppasswords) rather than your main account password. Generate one under **Google Account → Security → App Passwords**.

---

## Troubleshooting

### File Locking Issues

If files become randomly locked and inaccessible, temporarily disable file locking in `config/config.php` inside the container:

```php
'filelocking.enabled' => false,
```

Restart the container, attempt to delete or move the affected files, then re-enable locking by removing that line.

---

### Deadlocked Files (SQL Fix)

If files are completely deadlocked — nothing can be moved, deleted, or uploaded — use the following procedure to clear the lock table directly in MariaDB.

**Step 1 — Enable maintenance mode** in `config/config.php` to block new access during the fix:

```php
'maintenance' => true,
```

**Step 2 — Open a terminal in the MariaDB container** via Portainer (Container → Console → Connect):

```bash
mysql -u root -p
```

**Step 3 — Clear the file locks table:**

```sql
-- Check your config.php 'dbname' field for the exact database name
connect <your_nextcloud_db_name>;

-- Delete all lock entries
DELETE FROM oc_file_locks WHERE 1;

-- Exit
\q
```

**Step 4 — Disable maintenance mode** in `config/config.php`:

```php
'maintenance' => false,
```

Restart the Nextcloud container. Files should now be fully accessible.
