This is a fantastic and comprehensive homelab documentation\! It's well-structured, clear, and provides all the essential steps for setting up a robust homelab, especially for those without a static IP.

To address your request, I'll add a section highlighting crucial details that a first-timer might miss, emphasizing the importance of checking official documentation and specific file configurations.

-----

# 🏡 My Homelab Documentation

Welcome to the documentation for my personal homelab setup\! This guide details the configuration of my Ubuntu Server, Dockerized applications, and secure remote access via Cloudflare Tunnel. This documentation serves as a reference for myself in case of OS corruption or data loss, and as a guide for others looking to build a similar homelab, especially those without a static IP address.

## Table of Contents

1.  [Overview](#1-overview)
2.  [Prerequisites](#2-prerequisites)
3.  [Ubuntu Server Setup](#3-ubuntu-server-setup)
    * [Initial Setup](#initial-setup)
    * [Static IP Configuration](#static-ip-configuration)
4.  [Docker and Docker Compose Installation](#4-docker-and-docker-compose-installation)
    * [Install Docker Engine](#install-docker-engine)
    * [Post-installation Steps](#post-installation-steps)
    * [Install Docker Compose](#install-docker-compose)
5.  [Homelab Directory Structure](#5-homelab-directory-structure)
6.  [Cloudflare Tunnel Setup (Secure Remote Access)](#6-cloudflare-tunnel-setup-secure-remote-access)
    * [Why Cloudflare Tunnel?](#why-cloudflare-tunnel)
    * [Install Cloudflared](#install-cloudflared)
    * [Authenticate Cloudflared](#authenticate-cloudflared)
    * [Create a Tunnel](#create-a-tunnel)
    * [Configure Tunnel Routes (`homelab.yml`)](#configure-tunnel-routes-homelabyml)
    * [Add DNS Records via CLI](#add-dns-records-via-cli)
    * [Run the Tunnel as a Service](#run-the-tunnel-as-a-service)
    * [JSON and YML File Formats](#json-and-yml-file-formats)
7.  [Application Management with Docker](#7-application-management-with-docker)
    * [Portainer](#portainer)
    * [ERPNext Specifics](#erpnext-specifics)
        * [Installing Custom Apps for ERPNext](#installing-custom-apps-for-erpnext)
8.  [Important Notes for First-Timers](#-8-important-notes-for-first-timers)
9.  [Backup and Recovery Strategy (Future Enhancements)](#9-backup-and-recovery-strategy-future-enhancements)
10.  [Troubleshooting](#10-troubleshooting)
11. [Docker Container List and Ports](#11-docker-container-list-and-ports)

---

## 1\. Overview

My homelab is built on a robust Ubuntu Server installation, leveraging Docker containers for nearly all applications. This approach provides excellent isolation, portability, and ease of management. Since my ISP does not provide a static IP address, I utilize Cloudflare Tunnel to securely expose my homelab services to the internet without opening any ports on my router. This eliminates the need for dynamic DNS or exposing my public IP address.

**Key Components:**

  * **Operating System:** Ubuntu Server
  * **Containerization:** Docker & Docker Compose
  * **Remote Access:** Cloudflare Tunnel
  * **Application Management:** Portainer (for Docker GUI)
  * **Data Storage:** `/srv/homelab`

**Note for users with a static IP:** If your ISP provides you with a static IP address, you can directly point your domain records (A/AAAA) to your public IP. In such cases, a reverse proxy (like Nginx Proxy Manager or Traefik) would be a more common choice than Cloudflare Tunnel for internal routing and SSL termination. However, for those without a static IP, Cloudflare Tunnel is an excellent, secure, and performant alternative.

## 2\. Prerequisites

Before you begin, ensure you have:

  * A dedicated machine for your homelab (e.g., an old PC, a mini PC, a Raspberry Pi 4).
  * A USB drive or method to install Ubuntu Server.
  * Basic understanding of Linux command line.
  * A Cloudflare account with a registered domain name (for Cloudflare Tunnel).

## 3\. Ubuntu Server Setup

This section outlines the initial setup of your Ubuntu Server.

### Initial Setup

1.  **Download Ubuntu Server:** Get the latest LTS (Long Term Support) version from the official Ubuntu website.

2.  **Create Bootable USB:** Use tools like Rufus (Windows) or Etcher (cross-platform) to create a bootable USB drive.

3.  **Install Ubuntu Server:**

      * Boot your homelab machine from the USB drive.
      * Follow the on-screen prompts for installation.
      * **Crucial Step:** When prompted, select the option to install **OpenSSH server**. This will allow you to connect to your server remotely via SSH.
      * Set up a strong password for your user account.
      * Consider setting up LVM (Logical Volume Management) for easier disk management and resizing in the future, especially if you plan to expand storage.

4.  **Update Your System:** Once the installation is complete and you've logged in (either directly or via SSH), update all packages to their latest versions:

    ```bash
    sudo apt update
    sudo apt upgrade -y
    sudo apt autoremove -y
    ```

### Static IP Configuration

It's highly recommended to assign a static IP address to your Ubuntu Server. This ensures that its IP address on your local network never changes, making it easier to manage and connect to. We'll use Netplan for this.

**Your Netplan configuration is located in `/etc/netplan/01-netcfg.yaml` and should look like this:**

```yaml
network:
  version: 2
  ethernets:
    enp1s0: # Your network interface
      dhcp4: no
      addresses:
        - 192.168.1.7/24 # Your desired static IP and subnet mask
      routes:
        - to: 0.0.0.0/0
          via: 192.168.1.1 # Your router's gateway IP
      nameservers:
        addresses:
          - 1.1.1.1 # Primary DNS server (Cloudflare)
          - 8.8.8.8 # Secondary DNS server (Google)
```

1.  **Edit Netplan Configuration:**

    ```bash
    sudo nano /etc/netplan/01-netcfg.yaml
    ```

2.  **Apply Netplan Configuration:**

    ```bash
    sudo netplan try
    ```

    If there are no errors, press Enter to accept the changes. If you lose connectivity, Netplan will revert the changes after a timeout.

    ```bash
    sudo netplan apply
    ```

3.  **Verify IP Address:**

    ```bash
    ip a | grep enp1s0
    ```

    Confirm that your server now has the static IP address `192.168.1.7`.

## 4\. Docker and Docker Compose Installation

Docker is the cornerstone of this homelab, allowing us to run applications in isolated containers. Docker Compose simplifies the management of multi-container Docker applications.

### Install Docker Engine

Follow the official Docker documentation for the most up-to-date installation instructions. Here's a common method:

1.  **Install necessary packages:**
    ```bash
    sudo apt update
    sudo apt install ca-certificates curl gnupg lsb-release -y
    ```
2.  **Add Docker's official GPG key:**
    ```bash
    sudo mkdir -m 0755 -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    ```
3.  **Set up the repository:**
    ```bash
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```
4.  **Install Docker Engine, containerd, and Docker Compose (CLI plugin):**
    ```bash
    sudo apt update
    sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
    ```

### Post-installation Steps

These steps allow your non-root user to manage Docker and ensure Docker starts on boot.

1.  **Add your user to the `docker` group:**
    ```bash
    sudo usermod -aG docker $USER
    ```
    **Important:** You need to **log out and log back in** (or restart your SSH session) for the group changes to take effect.
2.  **Verify Docker installation:**
    ```bash
    docker run hello-world
    ```
    You should see a message indicating Docker is working correctly.
3.  **Enable Docker to start on boot:**
    ```bash
    sudo systemctl enable docker.service
    sudo systemctl enable containerd.service
    ```

### Install Docker Compose

While the `docker-compose-plugin` was installed above, you might sometimes encounter older guides that refer to a separate `docker-compose` binary. If you prefer the standalone `docker-compose` binary (though the `docker compose` CLI plugin is generally preferred now), you can install it as follows:

1.  **Download the latest stable release of Docker Compose:**
    ```bash
    sudo curl -L "https://github.com/docker/compose/releases/download/v2.24.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    # Replace v2.24.5 with the latest stable version from Docker Compose GitHub releases
    ```
2.  **Apply executable permissions:**
    ```bash
    sudo chmod +x /usr/local/bin/docker-compose
    ```
3.  **Verify installation:**
    ```bash
    docker-compose --version
    ```
    You should see the Docker Compose version.

## 5\. Homelab Directory Structure

All my homelab applications and their persistent data are stored under `/srv/homelab`. This provides a centralized and organized location for all services.

```
/srv/homelab/
├── apps.json           # ERPNext custom app configuration
├── apps-test-output.json # ERPNext testing output
├── frappe_docker/      # ERPNext Docker setup
├── filebrowser/        # Filebrowser configuration
├── homepage/           # Homepage dashboard configuration
├── Media/              # Centralized media storage (e.g., for Plex, Jellyfin)
├── nextcloud/          # Nextcloud data and configuration
├── n8n/                # n8n workflow automation data
├── nocodb/             # NocoDB data
└── portainer/          # Portainer persistent data
```

**Note on `apps.json` and `apps-test-output.json`:** These files are specifically for my ERPNext installation, used to manage custom applications directly with the ERPNext Docker image.

## 6\. Cloudflare Tunnel Setup (Secure Remote Access)

Cloudflare Tunnel is a fantastic solution for exposing services running on your homelab to the internet securely, without opening ports on your router or needing a static IP.

### Why Cloudflare Tunnel?

  * **No Static IP Required:** Connects to Cloudflare's edge network, bypassing dynamic IP issues.
  * **No Port Forwarding:** Eliminates the security risks associated with opening ports on your router.
  * **Enhanced Security:** Benefits from Cloudflare's DDoS protection, WAF, and other security features.
  * **Global Network:** Routes traffic through Cloudflare's expansive network for low latency.
  * **Easy SSL/TLS:** Cloudflare handles SSL/TLS termination automatically.

### Install Cloudflared

1.  **Download and Install `cloudflared`:**
    Cloudflare provides a convenient way to install `cloudflared` on Debian-based systems.

    ```bash
    sudo apt update
    sudo apt install -y lsb-release apt-transport-https ca-certificates curl
    curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-archive-keyring.gpg >/dev/null
    echo "deb [signed-by=/usr/share/keyrings/cloudflare-archive-keyring.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee \
    /etc/apt/sources.list.d/cloudflared.list
    sudo apt update
    sudo apt install cloudflared -y
    ```

2.  **Verify Installation:**

    ```bash
    cloudflared --version
    ```

### Authenticate Cloudflared

This step links your `cloudflared` instance to your Cloudflare account.

1.  **Run the login command:**
    ```bash
    cloudflared tunnel login
    ```
    This will output a URL. Copy this URL and paste it into your web browser.
2.  **Authorize Cloudflared:**
    In your browser, select your Cloudflare account and the domain you want to use for the tunnel. Cloudflare will then generate a certificate file (`cert.pem`) in `~/.cloudflared/`. This file authenticates your `cloudflared` instance.

### Create a Tunnel

1.  **Create a new tunnel:**
    Choose a meaningful name for your tunnel, e.g., `homelab-tunnel`.

    ```bash
    cloudflared tunnel create homelab-tunnel
    ```

    This command will output a **Tunnel ID** and a credentials file path (e.g., `/root/.cloudflared/<TUNNEL-ID>.json`).
    **Important:** You need to move this JSON file to a more accessible and secure location for your homelab setup. I store mine in `/etc/cloudflared/`.

    ```bash
    sudo mv ~/.cloudflared/<TUNNEL-ID>.json /etc/cloudflared/homelab.json
    ```

    Make sure to replace `<TUNNEL-ID>` with your actual Tunnel ID.

2.  **Record Tunnel ID:**
    Make a note of the **Tunnel ID** from the output of the `tunnel create` command. You'll need it for the `homelab.yml` configuration.

### Configure Tunnel Routes (`homelab.yml`)

This is where you define which internal services are exposed through the tunnel and at what public hostnames. I store my configuration in `/etc/cloudflared/homelab.yml`.

1.  **Create the configuration file:**

    ```bash
    sudo nano /etc/cloudflared/homelab.yml
    ```

2.  **Add your tunnel configuration:**
    Replace `YOUR_TUNNEL_ID` with the ID you obtained earlier and adjust the service mappings to match your Docker container names or internal IP addresses/ports.

    ```yaml
    tunnel: YOUR_TUNNEL_ID # e.g., a1b2c3d4-e5f6-7890-1234-567890abcdef
    credentials-file: /etc/cloudflared/homelab.json

    ingress:
      - hostname: homepage.yourdomain.com
        service: http://homepage:3000 # Docker container name and port or internal IP:Port
      - hostname: nextcloud.yourdomain.com
        service: http://nextcloud:80 # Docker container name and port
      - hostname: filebrowser.yourdomain.com
        service: http://filebrowser:80
      - hostname: portainer.yourdomain.com
        service: http://portainer:9000
      - hostname: erpnext.yourdomain.com
        service: http://frappe_docker-frontend-1:8080 # Using the specific container name and port
      - hostname: n8n.yourdomain.com
        service: http://n8n-n8n-1:5678 # Using the specific container name and port
      - hostname: nocodb.yourdomain.com
        service: http://2_pg-nocodb-1:8080 # Using the specific container name and port
      # If you wish to expose other Coolify related services:
      # - hostname: coolify.yourdomain.com
      #   service: http://coolify:8080
      # - hostname: traefik-dashboard.yourdomain.com
      #   service: http://coolify-proxy:8080 # Traefik dashboard typically on 8080 or 8088
      - service: http_status:404 # Default catch-all for unmatched requests
    ```

      * **`tunnel`**: Your unique Tunnel ID.
      * **`credentials-file`**: Path to the JSON file generated during tunnel creation.
      * **`ingress`**: A list of rules defining how requests are routed.
          * `hostname`: The public domain/subdomain you want to use.
          * `service`: The internal address and port of your Docker container or service. If you use Docker's default bridge network, you can often use the container's service name as the hostname (e.g., `http://homepage:3000`). Otherwise, use `http://<INTERNAL_IP>:<PORT>`.
          * `http_status:404`: A fallback rule to return a 404 for any unmatched hostnames, preventing unintended exposure.

### Add DNS Records via CLI

Instead of going to the Cloudflare dashboard, you can create the necessary CNAME DNS records for your tunnel directly from the command line.

For each `hostname` defined in your `homelab.yml` (e.g., `homepage.yourdomain.com`, `erpnext.yourdomain.com`), execute the following command, replacing `<TUNNEL_NAME>` with your tunnel's name (e.g., `homelab-tunnel`) and `<HOSTNAME>` with the desired subdomain:

```bash
cloudflared tunnel route dns <TUNNEL_NAME> <HOSTNAME>
```

**Example:**
If your tunnel name is `homelab-tunnel` and you want to expose `homepage.yourdomain.com`:

```bash
cloudflared tunnel route dns homelab-tunnel homepage.yourdomain.com
cloudflared tunnel route dns homelab-tunnel nextcloud.yourdomain.com
cloudflared tunnel route dns homelab-tunnel filebrowser.yourdomain.com
cloudflared tunnel route dns homelab-tunnel portainer.yourdomain.com
cloudflared tunnel route dns homelab-tunnel erpnext.yourdomain.com
cloudflared tunnel route dns homelab-tunnel n8n.yourdomain.com
cloudflared tunnel route dns homelab-tunnel nocodb.yourdomain.com
```

These commands will automatically create a CNAME record in your Cloudflare DNS, pointing `yourdomain.com` (e.g., `homepage`) to your tunnel's unique `*.cfargotunnel.com` address.

### Run the Tunnel as a Service

To ensure your tunnel starts automatically and runs reliably, configure it as a systemd service.

1.  **Create the systemd service file:**

    ```bash
    sudo nano /etc/systemd/system/cloudflared-homelab.service
    ```

2.  **Add the service configuration:**

    ```ini
    [Unit]
    Description=Cloudflare Tunnel for Homelab
    After=network.target

    [Service]
    Type=simple
    User=root
    ExecStart=/usr/bin/cloudflared --config /etc/cloudflared/homelab.yml tunnel run homelab-tunnel
    Restart=on-failure
    RestartSec=10
    StandardOutput=journal
    StandardError=journal

    [Install]
    WantedBy=multi-user.target
    ```

      * `ExecStart`: Points to the `cloudflared` executable, your configuration file, and the tunnel you created.
      * `User=root`: It's generally safe to run `cloudflared` as root since it only initiates outbound connections.

3.  **Reload systemd, enable, and start the service:**

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable cloudflared-homelab.service
    sudo systemctl start cloudflared-homelab.service
    ```

4.  **Check the service status:**

    ```bash
    sudo systemctl status cloudflared-homelab.service
    journalctl -u cloudflared-homelab.service -f
    ```

    Look for messages indicating the tunnel is healthy and connected.

### JSON and YML File Formats

Here's how your `homelab.json` and `homelab.yml` files should look in `/etc/cloudflared/`.

**`/etc/cloudflared/homelab.json` (Credentials file)**

This file is automatically generated by `cloudflared tunnel create` and contains sensitive credentials. **Do not share this file publicly.**

```json
{
  "TunnelID": "YOUR_TUNNEL_ID",
  "TunnelSecret": "YOUR_TUNNEL_SECRET",
  "ClientID": "YOUR_CLIENT_ID",
  "AccountID": "YOUR_ACCOUNT_ID",
  "TeamName": "YOUR_TEAM_NAME"
}
```

*Actual values will be generated by Cloudflare.*

**`/etc/cloudflared/homelab.yml` (Configuration file)**

This file defines the routing rules for your tunnel.

```yaml
tunnel: YOUR_TUNNEL_ID # Example: "a1b2c3d4-e5f6-7890-1234-567890abcdef"
credentials-file: /etc/cloudflared/homelab.json

ingress:
  - hostname: homepage.yourdomain.com
    service: http://homepage:3000
    # You can also add other properties like noTLSVerify if needed (use with caution)
    # noTLSVerify: true
  - hostname: nextcloud.yourdomain.com
    service: http://nextcloud:80
  - hostname: filebrowser.yourdomain.com
    service: http://filebrowser:80
  - hostname: portainer.yourdomain.com
    service: http://portainer:9000
  - hostname: erpnext.yourdomain.com
    service: http://frappe_docker-frontend-1:8080
  - hostname: n8n.yourdomain.com
    service: http://n8n-n8n-1:5678
  - hostname: nocodb.yourdomain.com
    service: http://2_pg-nocodb-1:8080
  # - hostname: coolify.yourdomain.com
  #   service: http://coolify:8080
  # - hostname: traefik-dashboard.yourdomain.com
  #   service: http://coolify-proxy:8080
  - service: http_status:404 # Fallback for unmatched hostnames
```

## 7\. Application Management with Docker

All homelab applications are deployed as Docker containers, managed via Docker Compose. Each application typically has its own directory under `/srv/homelab/` containing its `docker-compose.yml` and any persistent data.

### Portainer

Portainer provides a user-friendly web interface for managing Docker containers, images, volumes, and networks. It's a great tool for visualizing and interacting with your Docker environment.

**Installation (example `docker-compose.yml` in `/srv/homelab/portainer`):**

```yaml
version: '3.8'
services:
  portainer:
    image: portainer/portainer-ce:lts
    container_name: portainer
    restart: always
    ports:
      - "9000:9000" # Expose for internal access, Cloudflare Tunnel maps to this
      - "9443:9443" # For HTTPS access to Portainer directly (optional, if you bypass tunnel)
      - "8001:8000" # For Agent communication (if you use Portainer Agent)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /srv/homelab/portainer:/data # Persistent data for Portainer
```

To run Portainer:

```bash
cd /srv/homelab/portainer
docker compose up -d
```

Access via `http://portainer.yourdomain.com` (via Cloudflare Tunnel).

### ERPNext Specifics

My ERPNext installation involves custom apps, which are handled by `apps.json` and `apps-test-output.json` within the `frappe_docker` directory. The `frappe_docker` directory contains the specific Docker Compose setup for ERPNext. The main frontend container for ERPNext, as per your `docker ps` output, is `frappe_docker-frontend-1` and it runs on port `8080`.

**`/srv/homelab/frappe_docker/apps.json` (Example Structure):**

```json
[
  {
    "name": "custom_app_one",
    "url": "https://github.com/your-username/custom_app_one.git",
    "branch": "main"
  },
  {
    "name": "custom_app_two",
    "url": "https://github.com/your-username/custom_app_two.git",
    "branch": "develop"
  }
]
```

This file is crucial for telling the ERPNext Docker build process which custom applications to install and their respective Git repositories and branches.

#### Installing Custom Apps for ERPNext

To install custom applications for your ERPNext instance, you should follow the dedicated instructions provided within the `frappe_docker` repository. This guide will walk you through the process of adding new apps to your ERPNext Docker setup.

Please refer to the following documentation:

[**frappe\_docker/docs/custom-apps.md**](https://github.com/xEMPERORx/homelab/blob/master/frappe_docker/docs/custom-apps.md)

-----

## 8\. Important Notes for First-Timers

When setting up your homelab, it's easy to overlook small but critical details. Here are some key points that a first-timer should pay close attention to:

  * **Understanding `enp1s0` in Netplan:** The network interface name (`enp1s0` in this guide) can vary between machines. Always verify your actual interface name using `ip a` or `ifconfig` (if installed) before modifying your Netplan configuration. Using the wrong interface name will prevent your static IP from being applied.
  * **Logging out and back in after Docker Group Add:** After adding your user to the `docker` group (`sudo usermod -aG docker $USER`), you **must** log out of your SSH session and log back in (or reboot the server). Docker commands will not work as your user until this is done. This is a common pitfall\!
  * **Cloudflare Tunnel Credentials File (`homelab.json`):** The `cloudflared tunnel create` command generates this file in your user's home directory (`~/.cloudflared/`). It's crucial to move this file to a more secure and system-managed location like `/etc/cloudflared/` and update the `credentials-file` path in `homelab.yml` accordingly. Failing to move it could lead to issues if your user's home directory isn't persistent or has different permissions.
  * **Docker Container Names vs. Service Names in Cloudflare Ingress:** When defining `service` in your `homelab.yml`, you're typically referencing the **Docker service name** as defined in your `docker-compose.yml` file, especially if they are on the same Docker network (usually the default bridge network created by Docker Compose). For example, if your `homepage` service in `docker-compose.yml` is named `homepage`, then `http://homepage:3000` will work. If you have complex Docker networks or isolated containers, you might need to use the container's internal IP address instead.
  * **Verifying `cloudflared` DNS Routes:** After running `cloudflared tunnel route dns <TUNNEL_NAME> <HOSTNAME>`, always double-check your Cloudflare DNS dashboard to ensure the CNAME records have been created successfully. They should point to `YOUR_TUNNEL_ID.cfargotunnel.com`. Misconfigured DNS is a frequent cause of tunnel connectivity issues.
  * **Systemd Service File (`cloudflared-homelab.service`):** Pay close attention to the `ExecStart` line. Ensure the path to `cloudflared`, your configuration file (`/etc/cloudflared/homelab.yml`), and your tunnel name (`homelab-tunnel`) are all correct and match your setup precisely. A typo here will prevent the tunnel from starting as a service.
  * **Application-Specific Configuration Files:** Many applications (like Homepage, Nextcloud, n8n, etc.) require their own specific configuration files in their respective `/srv/homelab/<app_name>/` directories. This documentation provides the top-level structure, but remember to refer to each application's official documentation for detailed setup and configuration (e.g., creating user accounts, database connections, environment variables). These are critical for the application to function correctly *within* its Docker container.

-----

## 9\. Backup and Recovery Strategy (Future Enhancements)

While this documentation provides the setup steps, a robust backup and recovery strategy is paramount for any homelab.

**Future Considerations:**

  * **Configuration Backups:** Regularly back up all `docker-compose.yml` files and application configuration directories (`/srv/homelab/*`).
  * **Data Volume Backups:** Implement automated backups for Docker volumes (e.g., using `borgbackup`, `restic`, or simple `rsync` to another storage).
  * **Offsite Backups:** Consider pushing critical backups to cloud storage (e.g., Backblaze B2, S3 compatible storage) or an external drive.
  * **OS Image/Snapshots:** For critical OS configurations, consider disk imaging tools or hypervisor snapshots if running in a VM.

## 10\. Troubleshooting

  * **Cannot connect to SSH:**
      * Ensure OpenSSH server is installed and running (`sudo systemctl status ssh`).
      * Check your server's IP address.
      * Verify firewall rules (UFW).
  * **Docker containers not starting:**
      * Check Docker logs: `docker logs <container_name>`
      * Check Docker Compose logs: `docker compose logs <service_name>`
      * Verify port conflicts.
      * Ensure sufficient disk space and RAM.
  * **Cloudflare Tunnel issues:**
      * Check `cloudflared` service status: `sudo systemctl status cloudflared-homelab.service`
      * View `cloudflared` logs: `journalctl -u cloudflared-homelab.service -f`
      * Verify your `homelab.yml` syntax using a YAML linter.
      * Ensure your DNS CNAME records in Cloudflare are correctly pointing to your Tunnel ID (e.g., `xxxx.cfargotunnel.com`).
      * Temporarily try running the tunnel directly (not as a service) to see immediate output: `cloudflared --config /etc/cloudflared/homelab.yml tunnel run homelab-tunnel`

## 11\. Docker Container List and Ports

Below is a table summarizing the active Docker containers in your homelab, their assigned names, and the internal ports they are listening on. These are the ports that Cloudflare Tunnel routes traffic to.

| Application Name          | Image                                | Internal Port(s) | Description                                       |
| :------------------------ | :----------------------------------- | :--------------- | :------------------------------------------------ |
| `portainer`               | `portainer/portainer-ce:lts`         | `9000`           | Portainer web UI for Docker management            |
| `homepage`                | `ghcr.io/gethomepage/homepage:latest`| `3000`           | Your personal homelab dashboard                   |
| `frappe_docker-frontend-1`| `frappe-custom`                      | `8080`           | ERPNext / Frappe web frontend                     |
| `frappe_docker-websocket-1`| `frappe-custom`                     | *(No direct port)* | ERPNext websocket service (internal)              |
| `frappe_docker-scheduler-1`| `frappe-custom`                     | *(No direct port)* | ERPNext scheduler service (internal)              |
| `2_pg-nocodb-1`           | `nocodb/nocodb:latest`               | `8080`           | NocoDB database interface                         |
| `2_pg-root_db-1`          | `postgres:16.6`                      | `5432`           | PostgreSQL database for NocoDB (internal)         |
| `n8n-n8n-worker-1`        | `docker.n8n.io/n8nio/n8n`            | *(No direct port)* | n8n worker (internal)                             |
| `n8n-n8n-1`               | `docker.n8n.io/n8nio/n8n`            | `5678`           | n8n workflow automation platform                  |
| `n8n-postgres-1`          | `postgres:16`                        | `5432`           | PostgreSQL database for n8n (internal)            |
| `n8n-redis-1`             | `redis:6-alpine`                     | `6379`           | Redis cache for n8n (internal)                    |
| `coolify-proxy`           | `traefik:v3.1`                       | `80`, `443`, `8080` | Traefik proxy for Coolify (might not be directly exposed via Cloudflare Tunnel if Coolify manages its own subdomains) |
| `coolify`                 | `ghcr.io/coollabsio/coolify:4.0.0-beta.406` | `8080` (`8000/tcp` mapped to `8080`) | Coolify server for self-hosting apps            |
| `coolify-realtime`        | `ghcr.io/coollabsio/coolify-realtime:1.0.6` | `6001-6002`      | Coolify realtime service (internal)               |
| `coolify-redis`           | `redis:7-alpine`                     | `6379`           | Redis cache for Coolify (internal)                |
| `coolify-db`              | `postgres:15-alpine`                 | `5432`           | PostgreSQL database for Coolify (internal)        |
| `filebrowser`             | `filebrowser/filebrowser:s6`         | `80`             | Web-based file manager                            |

-----

This documentation should provide a solid foundation for your homelab setup and serve as a valuable reference. Happy homelabbing\! 🚀
If need further help rely on the official documentation
