# n8n Deployment on Google Cloud Platform

This guide outlines the steps to deploy n8n on a GCP instance using Docker, Docker Compose, and Nginx with SSL. The setup ensures data persistence, auto-updates, and secure access.

---

## Prerequisites

- A Google Cloud Platform (GCP) account.
- A registered domain name.
- Basic familiarity with SSH and Linux commands.

---

## Step 1: Prepare Your GCP Instance

### 1. Create a Compute Engine Instance
1. Log into the GCP Console and navigate to **Compute Engine > VM Instances**.
2. Click **Create Instance** and configure the following:
   - **Name**: `n8n-instance` (or your preferred name).
   - **Machine Type**: `e2-micro` (free tier eligible).
   - **Boot Disk**: `Ubuntu 22.04 LTS`.
   - **Firewall**: Check both **Allow HTTP** and **Allow HTTPS**.
3. Click **Create** to start the instance.

### 2. SSH into Your Instance
1. Connect via SSH using the GCP Console:
   - Click the **SSH** button next to your instance.
2. Update system packages:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

---

## Step 2: Install Docker and Docker Compose

### 1. Install Docker
1. Install Docker:
   ```bash
   sudo apt install -y docker.io
   ```
2. Enable Docker to start on boot:
   ```bash
   sudo systemctl enable docker
   ```
3. Add your user to the Docker group:
   ```bash
   sudo usermod -aG docker $USER
   ```
4. Log out and back in to apply the group changes or run:
   ```bash
   newgrp docker
   ```
5. Verify Docker installation:
   ```bash
   docker --version
   ```

### 2. Install Docker Compose
1. Download Docker Compose:
   ```bash
   sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   ```
2. Apply executable permissions:
   ```bash
   sudo chmod +x /usr/local/bin/docker-compose
   ```
3. Verify Docker Compose installation:
   ```bash
   docker-compose --version
   ```

---

## Step 3: Configure n8n with Docker Compose

### 1. Set Up Directories
1. Create a directory for n8n:
   ```bash
   mkdir ~/n8n-docker && cd ~/n8n-docker
   ```
2. Create a persistent data directory:
   ```bash
   sudo mkdir -p /var/n8n-data
   sudo chown 1000:1000 /var/n8n-data
   ```

### 2. Create the `docker-compose.yml` File
1. Open the file:
   ```bash
   nano docker-compose.yml
   ```
2. Add the following content (replace `your-domain.com` with your domain):
   ```yaml
   version: "3.1"

   services:
     n8n:
       image: n8nio/n8n:latest
       container_name: n8n
       restart: unless-stopped
       ports:
         - "5678:5678"
       environment:
         - N8N_HOST=your-domain.com
         - WEBHOOK_TUNNEL_URL=https://your-domain.com/
         - WEBHOOK_URL=https://your-domain.com/
         - GENERIC_TIMEZONE=Asia/Kolkata
       volumes:
         - /var/n8n-data:/root/.n8n
   ```
3. Save and close the file.

### 3. Start the n8n Container
1. Launch the container:
   ```bash
   docker-compose up -d
   ```
2. Verify the container is running:
   ```bash
   docker ps
   ```

---

## Step 4: Configure Nginx Reverse Proxy

### 1. Install Nginx
1. Install Nginx:
   ```bash
   sudo apt install -y nginx
   ```

### 2. Set Up Reverse Proxy
1. Create an Nginx configuration file for your domain:
   ```bash
   sudo nano /etc/nginx/sites-available/n8n
   ```
2. Add the following content (replace `your-domain.com` with your domain):
   ```nginx
   server {
       listen 80;
       server_name your-domain.com;

       location / {
           proxy_pass http://localhost:5678;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
       }
   }
   ```
3. Enable the configuration:
   ```bash
   sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl reload nginx
   ```

---

## Step 5: Enable SSL

### Option 1: Using Cloudflare
1. Add your domain to Cloudflare and update DNS records to point to your instance's IP address.
2. Enable **Full (Strict)** SSL mode in Cloudflare settings.

### Option 2: Using Certbot
1. Install Certbot:
   ```bash
   sudo apt install -y certbot python3-certbot-nginx
   ```
2. Obtain and configure an SSL certificate:
   ```bash
   sudo certbot --nginx -d your-domain.com
   ```
3. Verify automatic renewal:
   ```bash
   sudo certbot renew --dry-run
   ```

---

## Step 6: Set Up Watchtower for Auto-Updates

1. Install Watchtower:
   ```bash
   docker run -d \
   --name watchtower \
   --restart unless-stopped \
   -v /var/run/docker.sock:/var/run/docker.sock \
   containrrr/watchtower --cleanup --schedule "0 3 * * *"
   ```

---

## Step 7: Validate Your Setup

1. **Check n8n Logs**:
   ```bash
   docker logs n8n
   ```
2. **Verify Data Persistence**:
   - Create a workflow in n8n.
   - Ensure workflow data is saved in `/var/n8n-data`:
     ```bash
     ls /var/n8n-data
     ```
3. **Test Updates**:
   - Stop and remove the n8n container:
     ```bash
     docker stop n8n && docker rm n8n
     ```
   - Restart using `docker-compose up -d` and verify data persistence.

---

## Final Notes

- **Backups**: Regularly back up the `/var/n8n-data` directory to prevent data loss.
- **Monitoring**: Use tools like UptimeRobot to monitor service availability.
- **Support**: If issues occur, inspect container logs (`docker logs n8n`) and check the configuration files.

This setup ensures a reliable, secure, and auto-updating n8n deployment on GCP.
