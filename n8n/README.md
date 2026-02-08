# n8n Workflow Automation - Deployment Guide

n8n is a powerful workflow automation tool that lets you connect apps and automate tasks with a visual interface. This guide is specifically tailored for deployment on your **Raspberry Pi 4** running at **damianferencz.org**.

## Your Setup Overview

- **Pi Model**: Raspberry Pi 4 Model B
- **OS**: Raspberry Pi OS (64-bit)
- **Domain**: damianferencz.org
- **n8n URL**: https://n8n.damianferencz.org
- **Pi Path**: `/home/damian/docker`
- **SSH User**: damian
- **Existing Services**: Nginx Proxy Manager, Portainer, and all media services running

## Architecture

This n8n deployment includes:
- **n8n**: Workflow automation platform (port 5678)
- **PostgreSQL 15**: Database for workflow storage and execution logs
- **Isolated Network**: Dedicated `n8n_network` bridge for security
- **HTTPS Access**: Via Nginx Proxy Manager with Let's Encrypt SSL

## Pre-Deployment Checklist

### 1. Generate Encryption Key

On your **workstation**, generate a secure encryption key:

```bash
openssl rand -hex 32
```

Copy the output (it will look like: `a1b2c3d4e5f6...`) - you'll need this in the next step.

### 2. Update Environment Variables

On your **workstation**, edit the `.env` file:

```bash
cd /Users/damianferencz/GolandProjects/pi-docker
nano .env
```

Update these values in the n8n section:

```bash
# n8n - Workflow Automation
N8N_HOST=n8n.damianferencz.org
N8N_PROTOCOL=https
N8N_WEBHOOK_URL=https://n8n.damianferencz.org/
TIMEZONE=Europe/London  # Or your timezone

# n8n - PostgreSQL Database
N8N_DB_NAME=n8n
N8N_DB_USER=n8n
N8N_DB_PASSWORD=<GENERATE_STRONG_PASSWORD>  # Change this!
N8N_ENCRYPTION_KEY=<PASTE_KEY_FROM_STEP_1>  # Paste the key from step 1
```

**Generate a strong database password**:
```bash
openssl rand -base64 32
```

Save the file (`Ctrl+O`, `Enter`, `Ctrl+X`).

### 3. Verify Configuration

Double-check your `.env` file has:
- ✓ `N8N_ENCRYPTION_KEY` with 64 hex characters
- ✓ `N8N_DB_PASSWORD` with a strong password (not the example value)
- ✓ `N8N_HOST=n8n.damianferencz.org`
- ✓ `N8N_PROTOCOL=https`

## Deployment Steps

### Step 1: Sync Files to Raspberry Pi

From your **workstation**:

```bash
# Sync the entire project to your Pi
rsync -avz --exclude='.git' --exclude='node_modules' \
  /Users/damianferencz/GolandProjects/pi-docker/ \
  damian@damianferencz.org:/home/damian/docker/
```

This will copy:
- The new `n8n/` directory with `docker-compose.yml`
- The updated `.env` file with n8n configuration
- All other project files

Expected output:
```
sending incremental file list
n8n/
n8n/docker-compose.yml
.env
...
sent X bytes  received Y bytes
```

### Step 2: SSH into Your Raspberry Pi

```bash
ssh damian@damianferencz.org
```

You should now be connected to your Pi.

### Step 3: Navigate to n8n Directory

```bash
cd /home/damian/docker/n8n
```

### Step 4: Verify Files Are in Place

```bash
# Check docker-compose.yml exists
ls -la
# Output should show: docker-compose.yml

# Verify environment variables are loaded
cat /home/damian/docker/.env | grep N8N
```

You should see your n8n configuration printed.

### Step 5: Prepare Data Directory and Permissions

Before starting n8n, ensure the data directory has correct permissions:

```bash
# Create data directory if it doesn't exist
mkdir -p data

# Fix permissions (n8n runs as UID 1000)
sudo chown -R 1000:1000 data
```

### Step 6: Export Environment Variables and Start n8n

Docker Compose needs access to the environment variables from the parent `.env` file:

```bash
# Export all variables from the .env file
export $(cat /home/damian/docker/.env | grep -v '^#' | xargs)

# Start n8n services
docker-compose up -d
```

Expected output:
```
Creating network "n8n_n8n_network" with driver "bridge"
Creating n8n-postgres ... done
Creating n8n          ... done
```

### Step 7: Verify Services Are Running

```bash
docker-compose ps
```

Expected output:
```
NAME            IMAGE                    STATUS
n8n             n8nio/n8n:latest        Up (healthy)
n8n-postgres    postgres:15-alpine      Up (healthy)
```

Both should show `Up (healthy)` status. If not, wait 30 seconds and check again.

### Step 8: Check Logs

```bash
# View all logs
docker-compose logs -f

# Or just n8n logs
docker-compose logs -f n8n
```

Look for:
```
n8n ready on 0.0.0.0:5678
```

Press `Ctrl+C` to exit logs.

### Step 9: Test Local Access

From the Pi terminal:

```bash
curl -I http://localhost:5678
```

Expected response:
```
HTTP/1.1 200 OK
```

If you see this, n8n is running correctly!

## Configure Nginx Proxy Manager for HTTPS

Now let's set up secure external access via https://n8n.damianferencz.org

### Step 1: Access Nginx Proxy Manager

From your workstation, open:
```
http://<your-pi-ip>:81
```

Or if already configured:
```
https://proxy.damianferencz.org
```

Login with your credentials.

### Step 2: Create Proxy Host for n8n

1. Click **Hosts** → **Proxy Hosts** → **Add Proxy Host**

2. **Details Tab**:
   - **Domain Names**: `n8n.damianferencz.org`
   - **Scheme**: `http`
   - **Forward Hostname / IP**: `n8n` (this is the container name)
   - **Forward Port**: `5678`
   - **Cache Assets**: ✓ Enabled
   - **Block Common Exploits**: ✓ Enabled
   - **Websockets Support**: ✓ **IMPORTANT - Must be enabled!**
   - **Access List**: (leave as "Publicly Accessible" or configure if desired)

3. **SSL Tab**:
   - **SSL Certificate**: Select **Request a new SSL Certificate**
   - **Force SSL**: ✓ Enabled
   - **HTTP/2 Support**: ✓ Enabled
   - **HSTS Enabled**: ✓ Enabled
   - **Email Address for Let's Encrypt**: your-email@example.com
   - **I Agree to the Let's Encrypt Terms of Service**: ✓ Checked

4. Click **Save**

Nginx will now:
- Request an SSL certificate from Let's Encrypt
- Configure the proxy to forward to n8n
- Enable HTTPS redirect

### Step 3: Update DNS (If Not Already Done)

Ensure you have a DNS A record pointing to your Pi:

```
Type: A
Name: n8n
Value: <your-public-ip>
TTL: 300
```

Or if using Cloudflare DDNS (which you have configured), it will auto-update.

### Step 4: Test External Access

From your workstation browser, navigate to:

```
https://n8n.damianferencz.org
```

You should see the n8n setup page!

## Initial n8n Setup

When you first access https://n8n.damianferencz.org, you'll be prompted to create an owner account.

### Step 1: Create Owner Account

Fill in:
- **Email**: your-email@damianferencz.org (or any email)
- **First Name**: Damian
- **Last Name**: Ferencz (or any name)
- **Password**: **Use a strong password!**

Click **Get Started**.

### Step 2: Configure Workflow Settings (Optional)

You can:
- Skip the tour or take it
- Enable/disable telemetry
- Connect to n8n.cloud for community templates (optional)

### Step 3: Create Your First Workflow

Click **+ Add Workflow** and start automating!

## Common Operations

All commands assume you're SSH'd into the Pi and in `/home/damian/docker/n8n`.

### View Logs in Real-Time

```bash
ssh damian@damianferencz.org
cd /home/damian/docker/n8n
docker-compose logs -f
```

### Restart n8n

```bash
ssh damian@damianferencz.org
cd /home/damian/docker/n8n
docker-compose restart
```

### Stop n8n

```bash
ssh damian@damianferencz.org
cd /home/damian/docker/n8n
docker-compose down
```

### Start n8n (if stopped)

```bash
ssh damian@damianferencz.org
cd /home/damian/docker/n8n
export $(cat /home/damian/docker/.env | grep -v '^#' | xargs)
docker-compose up -d
```

### Update n8n to Latest Version

```bash
ssh damian@damianferencz.org
cd /home/damian/docker/n8n
export $(cat /home/damian/docker/.env | grep -v '^#' | xargs)
docker-compose pull
docker-compose down
docker-compose up -d
```

### Check Resource Usage

```bash
ssh damian@damianferencz.org
docker stats n8n n8n-postgres
```

Press `Ctrl+C` to exit.

### Check Pi Temperature

```bash
ssh damian@damianferencz.org
vcgencmd measure_temp
```

If over 80°C consistently, consider adding cooling.

## Backup and Restore

### Create Backup

From the **Pi**:

```bash
ssh damian@damianferencz.org
cd /home/damian/docker/n8n

# Stop n8n first
docker-compose down

# Create backup with today's date
sudo tar -czf n8n-backup-$(date +%Y%m%d-%H%M%S).tar.gz data/ postgres/

# Restart n8n
docker-compose up -d

# Move backup to safe location
mv n8n-backup-*.tar.gz ~/backups/
```

### Copy Backup to Workstation

From your **workstation**:

```bash
# Create backups directory
mkdir -p ~/Backups/pi-docker-n8n

# Download latest backup
scp damian@damianferencz.org:~/backups/n8n-backup-*.tar.gz ~/Backups/pi-docker-n8n/
```

### Restore from Backup

From the **Pi**:

```bash
ssh damian@damianferencz.org
cd /home/damian/docker/n8n

# Stop n8n
docker-compose down

# Remove existing data
sudo rm -rf data/ postgres/

# Extract backup
sudo tar -xzf ~/backups/n8n-backup-YYYYMMDD-HHMMSS.tar.gz

# Fix permissions
sudo chown -R 1000:1000 data/
sudo chown -R 999:999 postgres/

# Export environment variables and restart n8n
export $(cat /home/damian/docker/.env | grep -v '^#' | xargs)
docker-compose up -d
```

## Automated Backup Script

Create a backup script on your **Pi**:

```bash
ssh damian@damianferencz.org
nano ~/backup-n8n.sh
```

Paste this:

```bash
#!/bin/bash
# n8n Backup Script

BACKUP_DIR="/home/damian/backups"
N8N_DIR="/home/damian/docker/n8n"
ENV_FILE="/home/damian/docker/.env"
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="$BACKUP_DIR/n8n-backup-$DATE.tar.gz"

# Create backup directory if it doesn't exist
mkdir -p $BACKUP_DIR

# Stop n8n
cd $N8N_DIR
docker-compose down

# Create backup
sudo tar -czf $BACKUP_FILE data/ postgres/

# Export environment variables and restart n8n
export $(cat $ENV_FILE | grep -v '^#' | xargs)
docker-compose up -d

# Keep only last 7 backups
ls -t $BACKUP_DIR/n8n-backup-*.tar.gz | tail -n +8 | xargs -r rm

echo "Backup completed: $BACKUP_FILE"
```

Make it executable:

```bash
chmod +x ~/backup-n8n.sh
```

Test it:

```bash
~/backup-n8n.sh
```

### Schedule Weekly Backups

```bash
crontab -e
```

Add this line (runs every Sunday at 3 AM):

```bash
0 3 * * 0 /home/damian/backup-n8n.sh >> /home/damian/backup-n8n.log 2>&1
```

Save and exit.

## Troubleshooting

### Problem: Can't access https://n8n.damianferencz.org

**Solution 1**: Check n8n is running
```bash
ssh damian@damianferencz.org
cd /home/damian/docker/n8n
docker-compose ps
```

Both containers should show `Up (healthy)`.

**Solution 2**: Check Nginx Proxy Manager
- Visit http://<your-pi-ip>:81
- Check that the proxy host for `n8n.damianferencz.org` exists
- Verify it points to `n8n:5678` with websockets enabled

**Solution 3**: Check DNS
```bash
nslookup n8n.damianferencz.org
```

Should return your public IP.

**Solution 4**: Check SSL certificate
- In Nginx Proxy Manager, go to SSL Certificates
- Verify the certificate for `n8n.damianferencz.org` is valid

### Problem: n8n won't start

Check logs:
```bash
ssh damian@damianferencz.org
cd /home/damian/docker/n8n
docker-compose logs n8n
```

**Common issues**:

1. **Missing encryption key**:
   ```
   Error: N8N_ENCRYPTION_KEY is required
   ```
   Fix: Add the encryption key to `/home/damian/docker/.env`

2. **Database connection failed**:
   ```
   Error: Connection to PostgreSQL failed
   ```
   Fix: Check PostgreSQL is healthy:
   ```bash
   docker-compose logs n8n-postgres
   docker exec n8n-postgres pg_isready
   ```

3. **Port already in use**:
   ```
   Error: Port 5678 is already allocated
   ```
   Fix: Check what's using the port:
   ```bash
   sudo netstat -tulpn | grep 5678
   ```

4. **Permission denied error**:
   ```
   Error: EACCES: permission denied, open '/home/node/.n8n/config'
   ```
   Fix: The data directory needs correct permissions:
   ```bash
   cd /home/damian/docker/n8n
   sudo chown -R 1000:1000 data
   export $(cat /home/damian/docker/.env | grep -v '^#' | xargs)
   docker-compose restart
   ```

### Problem: Workflows execute slowly

**Check resources**:
```bash
ssh damian@damianferencz.org
docker stats n8n n8n-postgres
```

**If CPU/Memory is high**:
- Optimize workflows (reduce polling frequency, use webhooks)
- Close unused Chrome/Headless browser nodes
- Consider upgrading to SSD storage

**Check temperature**:
```bash
vcgencmd measure_temp
```

If over 80°C, add cooling.

### Problem: PostgreSQL won't start

Check logs:
```bash
ssh damian@damianferencz.org
cd /home/damian/docker/n8n
docker-compose logs n8n-postgres
```

**Solution 1**: Permission issues
```bash
sudo chown -R 999:999 postgres/
export $(cat /home/damian/docker/.env | grep -v '^#' | xargs)
docker-compose up -d
```

**Solution 2**: Corrupted data
```bash
docker-compose down
sudo rm -rf postgres/
export $(cat /home/damian/docker/.env | grep -v '^#' | xargs)
docker-compose up -d
```

Warning: This deletes all workflow execution history.

### Problem: "Webhooks not working"

**Verify webhook URL**:
```bash
ssh damian@damianferencz.org
cat /home/damian/docker/.env | grep N8N_WEBHOOK_URL
```

Should be: `N8N_WEBHOOK_URL=https://n8n.damianferencz.org/`

**Check Nginx Proxy Manager**:
- Websockets support must be enabled
- SSL must be configured

**Test webhook**:
Create a webhook node in n8n, copy the URL, test with curl:
```bash
curl -X POST https://n8n.damianferencz.org/webhook-test/<your-webhook-id>
```

## Performance Optimization

### Monitor Resource Usage

```bash
ssh damian@damianferencz.org
docker stats n8n n8n-postgres
htop
```

### Optimize PostgreSQL

Edit `docker-compose.yml` on your **workstation**:

```yaml
  n8n-postgres:
    image: postgres:15-alpine
    container_name: n8n-postgres
    command:
      - postgres
      - -c
      - shared_buffers=256MB
      - -c
      - max_connections=100
      - -c
      - work_mem=4MB
```

Then redeploy:
```bash
rsync -avz /Users/damianferencz/GolandProjects/pi-docker/n8n/ damian@damianferencz.org:/home/damian/docker/n8n/
ssh damian@damianferencz.org "cd /home/damian/docker/n8n && export \$(cat /home/damian/docker/.env | grep -v '^#' | xargs) && docker-compose down && docker-compose up -d"
```

### Use SSD Instead of MicroSD

If you have a USB 3.0 SSD:
1. Mount SSD to `/mnt/ssd`
2. Move n8n data:
   ```bash
   sudo mv /home/damian/docker/n8n /mnt/ssd/
   sudo ln -s /mnt/ssd/n8n /home/damian/docker/n8n
   ```

## Security Best Practices

### 1. Strong Passwords

- ✓ Use strong password for n8n owner account
- ✓ Use strong password for `N8N_DB_PASSWORD` in `.env`
- ✓ Never commit real passwords to git

### 2. HTTPS Only

- ✓ Always use `https://n8n.damianferencz.org`
- ✓ Never expose port 5678 directly to the internet
- ✓ Keep Nginx Proxy Manager handling SSL/TLS

### 3. Regular Updates

Update monthly:
```bash
ssh damian@damianferencz.org
cd /home/damian/docker/n8n
export $(cat /home/damian/docker/.env | grep -v '^#' | xargs)
docker-compose pull
docker-compose down
docker-compose up -d
```

### 4. Access Control

In n8n Settings:
- Enable "User Management" if you want multiple users
- Configure "Execution Mode" (main vs queue)
- Set "Timezone" correctly

### 5. Backup Regularly

Use the automated backup script above. Keep backups offsite.

## Useful n8n Features

### Community Nodes

Install additional nodes:
1. Go to https://n8n.damianferencz.org/settings/community-nodes
2. Search for nodes (e.g., "notion", "airtable")
3. Click "Install"
4. n8n will restart automatically

### Workflow Templates

Browse templates:
- https://n8n.io/workflows/
- Import JSON directly into n8n

### Webhooks

Every workflow can have a webhook trigger:
1. Add "Webhook" node
2. Set path (e.g., `/my-webhook`)
3. Full URL: `https://n8n.damianferencz.org/webhook/my-webhook`

### Scheduled Workflows

Use "Schedule Trigger" node for cron jobs:
- Every hour: `0 * * * *`
- Every day at 9 AM: `0 9 * * *`
- Every Monday: `0 0 * * 1`

## Quick Reference

### SSH into Pi
```bash
ssh damian@damianferencz.org
```

### n8n Directory
```bash
cd /home/damian/docker/n8n
```

### Export Environment Variables
**Important**: Always export environment variables before running `docker-compose up`:
```bash
export $(cat /home/damian/docker/.env | grep -v '^#' | xargs)
```

### View Logs
```bash
docker-compose logs -f
```

### Restart n8n
```bash
docker-compose restart
```

### Update n8n
```bash
export $(cat /home/damian/docker/.env | grep -v '^#' | xargs) && docker-compose pull && docker-compose down && docker-compose up -d
```

### Access n8n
```
https://n8n.damianferencz.org
```

### Access Nginx Proxy Manager
```
http://<pi-ip>:81
```

### Check System Stats
```bash
docker stats
htop
vcgencmd measure_temp
```

## Resources

- **n8n Documentation**: https://docs.n8n.io/
- **Community Forum**: https://community.n8n.io/
- **Workflow Templates**: https://n8n.io/workflows/
- **Community Nodes**: https://www.npmjs.com/search?q=n8n-nodes
- **n8n GitHub**: https://github.com/n8n-io/n8n

## Support Contacts

For deployment issues specific to this setup:
- Check container logs: `docker-compose logs -f`
- Verify environment: `cat /home/damian/docker/.env | grep N8N`
- Test database: `docker exec n8n-postgres pg_isready`

For n8n application issues:
- Community Forum: https://community.n8n.io/
- GitHub Issues: https://github.com/n8n-io/n8n/issues

---

**Deployment completed successfully!**

Your n8n instance is now running at **https://n8n.damianferencz.org**

Happy automating!
