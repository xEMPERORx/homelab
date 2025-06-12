# Setting up n8n with Docker and Cloudflare Tunnel

This guide will walk you through deploying n8n using Docker Compose, configuring it with a PostgreSQL database and Redis for queueing, and securely exposing it to the internet using Cloudflare Tunnel.

## Table of Contents

1.  [Overview](#1-overview)
2.  [Prerequisites](#2-prerequisites)
3.  [n8n Docker Compose Setup](#3-n8n-docker-compose-setup)
      * [Directory Structure](#directory-structure)
      * [Environment Variables (`.env`)](#environment-variables-env)
      * [Docker Compose File (`docker-compose.yml`)](#docker-compose-file-docker-composeyml)
      * [PostgreSQL Initialization Script (`init-data.sh`)](#postgresql-initialization-script-init-data.sh)
      * [Running n8n](#running-n8n)
4.  [Cloudflare Tunnel Configuration](#4-cloudflare-tunnel-configuration)
      * [Why Cloudflare Tunnel?](#why-cloudflare-tunnel)
      * [Update `homelab.yml`](#update-homelab.yml)
      * [Add DNS Record](#add-dns-record)
      * [Restart Cloudflare Tunnel Service](#restart-cloudflare-tunnel-service)
5.  [Accessing n8n](#5-accessing-n8n)
6.  [Troubleshooting](#6-troubleshooting)

## 1\. Overview

This setup utilizes Docker for containerization, providing a robust and isolated environment for n8n and its dependencies. We'll use PostgreSQL as the primary database for n8n's data and Redis for handling execution queues. Cloudflare Tunnel will be used to securely expose your n8n instance to the internet without requiring any port forwarding on your router.

## 2\. Prerequisites

Before you begin, ensure you have:

  * A server with Docker and Docker Compose installed (refer to [My Homelab Documentation - Docker Installation](https://www.google.com/search?q=%234-docker-and-docker-compose-installation) if you followed that guide).
  * A Cloudflare account with a registered domain name.
  * `cloudflared` installed and authenticated on your server (refer to [My Homelab Documentation - Cloudflare Tunnel Setup](https://www.google.com/search?q=%236-cloudflare-tunnel-setup-secure-remote-access)).
  * Basic understanding of Linux command line and Docker.

## 3\. n8n Docker Compose Setup

We will create a dedicated directory for n8n within your homelab structure, along with its configuration files.

### Directory Structure

Create a directory for your n8n setup, for example, `/srv/homelab/n8n/`.

```
/srv/homelab/n8n/
├── .env
├── docker-compose.yml
└── init-data.sh
```

### Environment Variables (`.env`)

Create a file named `.env` in the `/srv/homelab/n8n/` directory. This file will store sensitive information and configuration parameters for your n8n setup.

**Important:** Replace the placeholder values with your actual desired credentials and a strong encryption key.

```bash
# PostgreSQL Database Credentials (for n8n's data)
POSTGRES_USER=admin
POSTGRES_PASSWORD=your_strong_admin_password_here
POSTGRES_DB=n8n_db

# n8n's Non-Root PostgreSQL User (for n8n application to connect to the database)
POSTGRES_NON_ROOT_USER=n8n
POSTGRES_NON_ROOT_PASSWORD=your_strong_n8n_password_here

# n8n Encryption Key (IMPORTANT: Generate a strong, random key and keep it secret!)
# You can generate one using: head /dev/urandom | tr -dc A-Za-z0-9_.- | head -c 32
ENCRYPTION_KEY=your_very_secure_encryption_key_32_chars_long

# Webhook URL for n8n (This is the public URL through Cloudflare Tunnel)
# IMPORTANT: Replace 'n8n.labdiy.xyz' with your actual Cloudflare Tunnel subdomain.
WEBHOOK_URL=https://n8n.yourdomain.com
```

**Explanation of variables:**

  * `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`: These are for the root PostgreSQL user and the database that n8n will use.
  * `POSTGRES_NON_ROOT_USER`, `POSTGRES_NON_ROOT_PASSWORD`: These credentials are used by the n8n application to connect to the PostgreSQL database. It's best practice to use a non-root user for applications.
  * `ENCRYPTION_KEY`: **Crucial for security\!** This key encrypts sensitive data within n8n. **Generate a unique, strong key and never share it.**
  * `WEBHOOK_URL`: This tells n8n what its public URL is. It's essential for webhooks to function correctly. **Ensure this matches the `hostname` you'll configure in your Cloudflare Tunnel.**

### Docker Compose File (`docker-compose.yml`)

Create a file named `docker-compose.yml` in the `/srv/homelab/n8n/` directory.

```yaml
version: '3.8'

volumes:
  db_storage:
  n8n_storage:
  redis_storage:

x-shared: &shared
  restart: always
  image: docker.n8n.io/n8nio/n8n
  environment:
    - DB_TYPE=postgresdb
    - DB_POSTGRESDB_HOST=postgres
    - DB_POSTGRESDB_PORT=5432
    - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
    - DB_POSTGRESDB_USER=${POSTGRES_NON_ROOT_USER}
    - DB_POSTGRESDB_PASSWORD=${POSTGRES_NON_ROOT_PASSWORD}
    - EXECUTIONS_MODE=queue
    - QUEUE_BULL_REDIS_HOST=redis
    - QUEUE_HEALTH_CHECK_ACTIVE=true
    - N8N_ENCRYPTION_KEY=${ENCRYPTION_KEY}
    - WEBHOOK_URL=${WEBHOOK_URL} # Dynamically loaded from .env
  links:
    - postgres
    - redis
  volumes:
    - n8n_storage:/home/node/.n8n
  depends_on:
    redis:
      condition: service_healthy
    postgres:
      condition: service_healthy

services:
  postgres:
    image: postgres:16
    restart: always
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_NON_ROOT_USER=${POSTGRES_NON_ROOT_USER}
      - POSTGRES_NON_ROOT_PASSWORD=${POSTGRES_NON_ROOT_PASSWORD}
    volumes:
      - db_storage:/var/lib/postgresql/data
      - ./init-data.sh:/docker-entrypoint-initdb.d/init-data.sh
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -h localhost -U ${POSTGRES_USER} -d ${POSTGRES_DB}']
      interval: 5s
      timeout: 5s
      retries: 10

  redis:
    image: redis:6-alpine
    restart: always
    volumes:
      - redis_storage:/data
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 5s
      timeout: 5s
      retries: 10

  n8n:
    <<: *shared
    ports:
      - 5678:5678

  n8n-worker:
    <<: *shared
    command: worker
    depends_on:
      - n8n
```

### PostgreSQL Initialization Script (`init-data.sh`)

Create a file named `init-data.sh` in the `/srv/homelab/n8n/` directory. This script will ensure the non-root user for n8n is created and has the correct permissions to the database.

```bash
#!/bin/bash
set -e

psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
    CREATE USER "$POSTGRES_NON_ROOT_USER" WITH PASSWORD '$POSTGRES_NON_ROOT_PASSWORD';
    GRANT ALL PRIVILEGES ON DATABASE "$POSTGRES_DB" TO "$POSTGRES_NON_ROOT_USER";
EOSQL
```

Make sure this script is executable:

```bash
chmod +x /srv/homelab/n8n/init-data.sh
```

### Running n8n

Navigate to your n8n directory and start the Docker containers:

```bash
cd /srv/homelab/n8n/
docker compose up -d
```

This command will:

  * Create the `postgres`, `redis`, `n8n`, and `n8n-worker` containers.
  * Initialize the PostgreSQL database using `init-data.sh`.
  * Start n8n, connected to PostgreSQL and Redis.
  * Keep the containers running in the background (`-d`).

You can check the status of your containers with:

```bash
docker compose ps
docker compose logs -f
```

## 4\. Cloudflare Tunnel Configuration

Now that n8n is running internally, we'll use Cloudflare Tunnel to expose it to the internet securely.

### Why Cloudflare Tunnel?

Cloudflare Tunnel allows you to expose your local services to the internet without opening ports on your router, providing enhanced security and bypassing issues with dynamic IP addresses.

### Update `homelab.yml`

You need to modify your existing Cloudflare Tunnel configuration file, typically located at `/etc/cloudflared/homelab.yml`, to include a route for n8n.

1.  Open the file for editing:

    ```bash
    sudo nano /etc/cloudflared/homelab.yml
    ```

2.  Add the following entry under the `ingress:` section. **Ensure `n8n.yourdomain.com` matches the `WEBHOOK_URL` you set in your `.env` file.**

    ```yaml
    # ... (existing tunnel and credentials-file lines)

    ingress:
      # ... (existing hostname entries)
      - hostname: n8n.yourdomain.com # Replace with your actual desired subdomain
        service: http://n8n:5678    # This points to the 'n8n' service in your docker-compose.yml
      # ... (http_status:404 fallback)
    ```

    **Important Note:** The `service: http://n8n:5678` refers to the `n8n` service name within your `docker-compose.yml` file, and its exposed internal port. Docker's internal networking allows Cloudflare Tunnel to resolve `n8n` to the correct container IP address.

### Add DNS Record

Create the necessary CNAME DNS record in Cloudflare for your n8n subdomain.

```bash
cloudflared tunnel route dns homelab-tunnel n8n.yourdomain.com
```

Replace `homelab-tunnel` with your actual tunnel name and `n8n.yourdomain.com` with your chosen subdomain. This command automatically sets up the CNAME record in your Cloudflare DNS settings.

### Restart Cloudflare Tunnel Service

For the changes in `homelab.yml` and the new DNS record to take effect, you need to restart the `cloudflared` service:

```bash
sudo systemctl restart cloudflared-homelab.service
sudo systemctl status cloudflared-homelab.service
journalctl -u cloudflared-homelab.service -f
```

Verify that the service is running without errors and that the logs indicate the tunnel is healthy and routing traffic for your n8n subdomain.

## 5\. Accessing n8n

Once the Cloudflare Tunnel is active and routing correctly, you can access your n8n instance by navigating to your configured URL in a web browser:

`https://n8n.yourdomain.com`

You should be greeted by the n8n setup wizard to create your first user account.

## 6\. Troubleshooting

  * **n8n container not starting:**
      * Check `docker compose logs n8n`. Look for database connection errors.
      * Ensure your `.env` variables (`POSTGRES_USER`, `POSTGRES_PASSWORD`, etc.) are correctly set and match the `init-data.sh` script.
      * Verify `init-data.sh` has executable permissions (`chmod +x`).
  * **Cannot access n8n via public URL:**
      * Verify the `cloudflared-homelab.service` is running and healthy (`sudo systemctl status cloudflared-homelab.service`).
      * Check `journalctl -u cloudflared-homelab.service -f` for any routing errors or connection issues.
      * Ensure the `hostname` in `homelab.yml` exactly matches the `WEBHOOK_URL` in your `.env` file and the DNS record you created.
      * Confirm the CNAME record in your Cloudflare DNS dashboard is present and correctly configured.
      * Make sure n8n is running on `http://n8n:5678` within the Docker network. You can test this internally on your server: `curl http://localhost:5678` (if you briefly expose port 5678 for testing, but remember to remove it afterwards for security). Alternatively, you can `docker exec -it n8n-n8n-1 bash` into the n8n container and try `curl http://localhost:5678` from there to confirm the application itself is running.
  * **Webhooks not working:**
      * Double-check that the `WEBHOOK_URL` in your `.env` file is exactly `https://n8n.yourdomain.com` (using `https` is important as Cloudflare provides SSL).
      * Ensure your Cloudflare Tunnel is correctly handling HTTPS traffic to n8n.

Happy Automating with n8n\!
