Here's the information formatted properly for a Markdown (MD) file, as you requested.

````markdown
# Docker Compose Deployment for AFFiNE

Docker Compose is the recommended method for self-hosting AFFiNE.

## Prerequisites

You need Docker or another Docker-compatible container service installed on your system to use the `docker compose` command.

## Steps

### 1. Create folders for persisted data

First, create a dedicated directory for your AFFiNE data and navigate into it:

```bash
mkdir affine
cd affine
````

### 2\. Get the latest `docker-compose.yml`

You have two options to obtain the `docker-compose.yml` file:

**Option A: Download the latest release (Recommended)**

Download the `docker-compose.yml` file tagged with the latest AFFiNE release directly:

```bash
wget -O docker-compose.yml [https://github.com/toeverything/affine/releases/latest/download/docker-compose.yml](https://github.com/toeverything/affine/releases/latest/download/docker-compose.yml)
```

**Option B: Manually copy the content**

Alternatively, you can go to the [docker-compose.yml on GitHub](https://www.google.com/search?q=https://github.com/toeverything/affine/blob/master/docker-compose.yml), copy its content, and then paste it into a manually created `docker-compose.yml` file within the `affine` folder you created in Step 1.

### 3\. Get the `.env` file

A `.env` file is crucial for configuring Docker volume mappings (where user data will be persisted) and other required environment variables.

**Option A: Download the default example (Recommended)**

Download the `default.env.example` file and rename it to `.env`:

```bash
wget -O .env [https://github.com/toeverything/affine/releases/latest/download/default.env.example](https://github.com/toeverything/affine/releases/latest/download/default.env.example)
```

**Option B: Manually copy the content**

Alternatively, you can go to the [.env.example on GitHub](https://www.google.com/search?q=https://github.com/toeverything/affine/blob/master/default.env.example), copy its content, and then paste it into a manually created `.env` file within the `affine` folder.

### Update the `.env` file

After obtaining the `.env` file, you **must** open it and update the fields to match your desired configurations. Here are some key fields to review:

  * **`DB_DATA_LOCATION`**: The folder for Postgres data. This is usually the folder created in Step 1 with a `/postgres` suffix.
      * Example: `DB_DATA_LOCATION=./postgres`
  * **`UPLOAD_LOCATION`**: The folder for uploaded blobs/files. This is usually the folder created in Step 1 with a `/storage` suffix.
      * Example: `UPLOAD_LOCATION=./storage`
  * **`CONFIG_LOCATION`**: The folder for custom configurations. This is usually the folder created in Step 1 with a `/config` suffix.
      * Example: `CONFIG_LOCATION=./config`
  * **Database Credentials and Names**:
      * `DB_USERNAME=affine`
      * `DB_PASSWORD=` (You **should** set a strong password here)
      * `DB_DATABASE=affine`

**⚠️ WARNING:** Most of the values in the `.env` file should **not** be changed once your host has initialized and data has been written. If you need to change the location of the `affine` folder or other critical paths, please refer to the AFFiNE documentation on **Backup and Restore** for guidance to prevent data loss.

### 4\. Start the containers

Once your `docker-compose.yml` and `.env` files are set up, start the AFFiNE containers in detached mode (runs in the background):

```bash
docker compose up -d
```

### 5\. Validate the deployment in your browser

If everything deploys successfully, you should be able to access your AFFiNE instance in your web browser. By default, it's usually accessible at:

```
http://localhost:3010
```

*Note: You might need to update the port if you configured a different one in your `.env` file or `docker-compose.yml`.*

```
```
