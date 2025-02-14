# n8n Deployment on Google Cloud with SSL

<div style="background-color:#f0f8ff; padding:10px; border-radius:5px; border:1px solid #ccc;">
  <h2 style="color:#2e6da4;">Overview</h2>
  <p>This guide walks you through the process of deploying <strong>n8n</strong> on Google Cloud with SSL encryption. It covers everything from creating your Google Cloud project to configuring your virtual machine, setting up Cloud SQL for PostgreSQL, and deploying n8n with Docker. Two reverse proxy flows are provided: one using <strong>Caddy</strong> and an alternate flow using <strong>Nginx</strong>.</p>
</div>

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Google Cloud Setup](#google-cloud-setup)
   - [Create a Project](#create-a-project)
   - [Enable Required APIs](#enable-required-apis)
3. [Compute Engine Configuration](#compute-engine-configuration)
   - [Create a VM Instance](#create-a-vm-instance)
   - [Reserve a Static IP (Optional)](#reserve-a-static-ip)
   - [Open Required Ports](#open-required-ports)
4. [Connecting and Installing Docker](#connecting-and-installing-docker)
5. [Cloud SQL Configuration](#cloud-sql-configuration)
   - [Set Up PostgreSQL Instance](#set-up-postgresql-instance)
   - [Create a Database and User](#create-a-database-and-user)
   - [Allow VM Access to Cloud SQL](#allow-vm-access-to-cloud-sql)
6. [Deploying n8n with Docker](#deploying-n8n-with-docker)
   - [Deployment without a Domain](#deployment-without-a-domain)
   - [Deployment with a Domain](#deployment-with-a-domain)
7. [Reverse Proxy Setup](#reverse-proxy-setup)
   - [Using Caddy](#using-caddy)
   - [Alternate Flow: Using Nginx](#alternate-flow-using-nginx)
8. [Troubleshooting & Potential Errors](#troubleshooting--potential-errors)
9. [Additional Techniques](#additional-techniques)

---

## Prerequisites

- A Google Cloud account.
- A domain name (if deploying with a domain).
- Basic knowledge of Docker, SSH, and PostgreSQL.
- [Optional] Familiarity with reverse proxy servers (Caddy or Nginx).

---

## Google Cloud Setup

### Create a Project
1. Go to the [Google Cloud Console](https://console.cloud.google.com/).
2. Click the project dropdown (top-left) and select **New Project**.
3. Name your project (e.g., `<YOUR_PROJECT_NAME>`).
4. Click **Create**.

### Enable Required APIs
1. Navigate to **APIs & Services → Library**.
2. Enable the following APIs:
   - **Compute Engine API**
   - **Cloud SQL API**

---

## Compute Engine Configuration

### Create a VM Instance
1. Go to **Compute Engine → VM Instances → Create Instance**.
2. Set the following:
   - **Name:** `n8n-server`
   - **Region:** e.g., `us-central1`
   - **Machine Type:** `e2-small` (or as needed)
   - **Boot Disk:** Ubuntu 22.04 LTS (default disk size is 10 GB)
   - **Firewall:** Check **Allow HTTP traffic** and **Allow HTTPS traffic**
3. Click **Create**.

### Reserve a Static IP (Optional)
1. Stop your VM.
2. Edit the VM, click on the **Network interface**.
3. Under **External IPv4 address**, select **Reserve Static IP Address**.
4. Name it (e.g., `<STATIC_IP_NAME>`) and click **Reserve**.
5. Start your VM.

### Open Required Ports
n8n uses port `5678` for HTTP and `5679` for HTTPS.
1. Navigate to **VPC Network → Firewall → Create Firewall Rule**.
2. Configure:
   - **Name:** `allow-n8n`
   - **Targets:** All instances in the network
   - **Source IP ranges:** `0.0.0.0/0` (adjust for your security needs)
   - **Protocols and Ports:** TCP: `5678,5679`
3. Click **Create**.

---

## Connecting and Installing Docker

1. **Connect to Your VM via SSH:**
   - Go to **Compute Engine → VM Instances**
   - Click the **SSH** button for `n8n-server`

2. **Install Docker:**
   ```bash
   curl -fsSL https://get.docker.com -o get-docker.sh
   sudo sh get-docker.sh
   ```

3. **Add Your User to the Docker Group and Reboot:**
   ```bash
   sudo usermod -aG docker $USER
   sudo reboot
   ```

4. **Reconnect via SSH and Verify Docker Installation:**
   ```bash
   docker --version
   ```

## Cloud SQL Configuration

### Set Up PostgreSQL Instance
1. In the GCP Console, navigate to **Cloud SQL → Create Instance**.
2. Select PostgreSQL.
3. Configure:
   - Instance ID: `n8n-db`
   - Database Version: PostgreSQL 15
   - Region: Same as your VM (e.g., us-central1)
   - Machine Type: Choose the smallest option (e.g., 1 vCPU, 0.6 GB RAM)
   - Storage: 10 GB (default)
4. Click Create (this may take a few minutes).

### Create a Database and User
1. Go to your n8n-db instance → Databases → Create Database:
   - Database Name: `n8n_db`
2. Go to Users → Create User Account:
   - Username: `n8n_user`
   - Password: `<YOUR_DB_PASSWORD>`
3. Save your credentials securely.

### Allow VM Access to Cloud SQL
1. In your n8n-db instance → Connections → Networking.
2. Under Add Network, paste your VM's external IP address.
3. Click Done and then Save.

## Deploying n8n with Docker

### Deployment without a Domain
In your VM's SSH terminal, run:
```bash
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -e DB_TYPE=postgresdb \
  -e DB_POSTGRESDB_DATABASE=<YOUR_DB_NAME> \
  -e DB_POSTGRESDB_HOST=<YOUR_CLOUD_SQL_PUBLIC_IP> \
  -e DB_POSTGRESDB_PORT=5432 \
  -e DB_POSTGRESDB_USER=<YOUR_DB_USER> \
  -e DB_POSTGRESDB_PASSWORD=<YOUR_DB_PASSWORD> \
  -e N8N_BASIC_AUTH_ACTIVE=true \
  -e N8N_BASIC_AUTH_USER=<YOUR_N8N_USERNAME> \
  -e N8N_BASIC_AUTH_PASSWORD=<YOUR_N8N_PASSWORD> \
  -e N8N_SECURE_COOKIE=false \
  n8nio/n8n
```

### Deployment with a Domain
1. **Set Up DNS:**
   - In your domain registrar (e.g., Namecheap), create an A record:
     - Host: @ or a subdomain (e.g., n8n)
     - Value: Your VM's External IP
     - TTL: Automatic

2. **Run n8n with Domain Configuration:**
```bash
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -e DB_TYPE=postgresdb \
  -e DB_POSTGRESDB_DATABASE=<YOUR_DB_NAME> \
  -e DB_POSTGRESDB_HOST=<YOUR_CLOUD_SQL_PUBLIC_IP> \
  -e DB_POSTGRESDB_PORT=5432 \
  -e DB_POSTGRESDB_USER=<YOUR_DB_USER> \
  -e DB_POSTGRESDB_PASSWORD=<YOUR_DB_PASSWORD> \
  -e N8N_HOST=<YOUR_DOMAIN> \
  -e N8N_PROTOCOL=https \
  -e N8N_BASIC_AUTH_ACTIVE=true \
  -e N8N_BASIC_AUTH_USER=<YOUR_N8N_USERNAME> \
  -e N8N_BASIC_AUTH_PASSWORD=<YOUR_N8N_PASSWORD> \
  -e WEBHOOK_TUNNEL_URL=https://<YOUR_DOMAIN> \
  -e WEBHOOK_URL=https://<YOUR_DOMAIN> \
  n8nio/n8n
```

## Reverse Proxy Setup

### Using Caddy
1. **Install Caddy:**
```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

2. **Configure Caddy:**
```bash
sudo nano /etc/caddy/Caddyfile
```
Add (replace <YOUR_DOMAIN>):
```caddy
<YOUR_DOMAIN> {
    reverse_proxy localhost:5678
}
```

3. **Restart Caddy:**
```bash
sudo systemctl restart caddy
```

### Alternate Flow: Using Nginx
1. **Install Nginx:**
```bash
sudo apt update
sudo apt install nginx
```

2. **Configure Nginx:**
```bash
sudo nano /etc/nginx/sites-available/n8n.conf
```
Add (replace <YOUR_DOMAIN>):
```nginx
server {
    listen 80;
    server_name <YOUR_DOMAIN>;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name <YOUR_DOMAIN>;

    ssl_certificate /etc/letsencrypt/live/<YOUR_DOMAIN>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<YOUR_DOMAIN>/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

3. **Test and Reload Nginx:**
```bash
sudo nginx -t
sudo systemctl reload nginx
```

## Troubleshooting & Potential Errors

- **psql Client Not Found**:
  ```bash
  sudo apt install postgresql-client
  ```

- **Service Unit Not Found (PostgreSQL)**:
  - Manage restarts via GCP Console instead of systemctl

- **Docker Permissions**:
  ```bash
  sudo usermod -aG docker $USER
  sudo reboot
  ```

- **DNS Propagation Delays**:
  - Use tools like DNS Checker to verify propagation

- **SSL Certificate Errors**:
  - Verify certificate paths and DNS records

## Additional Techniques

- **Automated Backups**:
  Use Cloud SQL's automated exports or:
  ```bash
  pg_dump -h <DB_IP> -U <DB_USER> -d <DB_NAME> > backup.sql
  ```

- **Monitoring**:
  ```bash
  docker logs -f n8n
  ```

- **Scaling**:
  - Use managed instance groups for Compute Engine
  - Scale Cloud SQL based on load

<div style="text-align:center; padding:20px; background-color:#d9edf7; border-radius:5px; border:1px solid #bce8f1;">
  <h3 style="color:#31708f;">Congratulations!</h3>
  <p>Your n8n instance is now deployed on Google Cloud with SSL. Enjoy the powerful workflow automation!</p>
</div>
