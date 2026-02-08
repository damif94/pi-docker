# pi-docker Project Context

This file contains project-specific context and configuration details for Claude Code.

## Raspberry Pi Hardware Configuration

### Hardware Specifications
- **Model**: Raspberry Pi 4 Model B
- **Architecture**: ARM64 (aarch64)
- **Operating System**: Raspberry Pi OS (64-bit)
- **Docker**: Installed and configured
- **Docker Compose**: Installed and configured

### Access Method
- **Primary Access**: SSH
- **SSH Username**: damian
- **Project Path on Pi**: `/home/damian/docker`
- **Domain**: damianferencz.org
- **Network**: Local network with external domain access via Nginx Proxy Manager

## Project Structure

This is a Docker-based home server setup running multiple services:
- **nginx**: Nginx Proxy Manager with MariaDB backend (ports 80, 81, 443) ✓ Running
- **portainer**: Container management UI (port 9000) ✓ Running
- **watchtower**: Automated container updates with Telegram notifications ✓ Running
- **bazarr**: Subtitle management ✓ Running
- **emby**: Media server ✓ Running
- **prowlarr**: Indexer manager ✓ Running
- **qbittorrent**: BitTorrent client ✓ Running
- **radarr**: Movie management ✓ Running
- **sonarr**: TV show management ✓ Running
- **tdarr**: Media transcoding ✓ Running
- **n8n**: Workflow automation (port 5678) → Deploying

### Service Access URLs
- **Nginx Proxy Manager**: http://your-pi-ip:81
- **Portainer**: http://your-pi-ip:9000
- **n8n** (after deployment): https://n8n.damianferencz.org

## Architecture Patterns

### Service Organization
- Each service has its own directory with a `docker-compose.yml` file
- Services are started independently: `cd <service> && docker compose up -d`
- Data persistence: Each service uses local `./data` volumes
- Database services: Local `./mysql`, `./postgres` volumes

### Environment Variables
- `.env` file at project root (encrypted with git-crypt)
- `.env.example` provides template for all required variables
- Services reference environment variables using `${VAR_NAME}` syntax

### Networking
- Most services use isolated Docker networks (e.g., `npm_network`, `n8n_network`)
- External ports exposed only where necessary
- Proxy Manager handles SSL/TLS termination

## Security Considerations

### git-crypt
- `.env` file is encrypted using git-crypt
- Secrets are protected in the repository
- `.gitattributes` configures which files are encrypted

### Secrets Management
- Telegram bot tokens for notifications
- Database passwords for MySQL/PostgreSQL
- Cloudflare API tokens for DDNS
- n8n encryption keys

## Deployment Workflow

### Standard Service Deployment
1. **From workstation**: Commit and push changes
   ```bash
   git add .
   git commit -m "Your commit message"
   git push origin master
   ```

2. **SSH into Pi**:
   ```bash
   ssh damian@damianferencz.org
   ```

3. **Navigate to project directory and pull changes**:
   ```bash
   cd /home/damian/docker
   git pull origin master
   ```

4. **Navigate to service directory**:
   ```bash
   cd /home/damian/docker/<service>
   ```

5. **Ensure `.env` file has required variables**:
   ```bash
   cat /home/damian/docker/.env
   ```

6. **Start service**:
   ```bash
   docker compose up -d
   ```

7. **Check logs**:
   ```bash
   docker compose logs -f
   ```

8. **Configure via web UI** if applicable

### ARM64 Compatibility
- All services use multi-arch images compatible with ARM64
- Official images generally support `linux/arm64` platform
- Pi 4 with 64-bit OS provides optimal performance

## n8n Specific Notes

### Prerequisites
- PostgreSQL 15 database (included in compose file)
- Encryption key must be generated: `openssl rand -hex 32`
- Webhook URL configuration for external access
- Optional: Nginx Proxy Manager for SSL/TLS

### Performance
- Pi 4 handles n8n well for light-medium workflow automation
- Monitor memory usage for complex workflows
- PostgreSQL provides better performance than SQLite for n8n

## Useful Commands

### Docker Management
```bash
# View all running containers
docker ps

# View container logs
docker compose logs -f <service>

# Restart a service
docker compose restart

# Update and restart
docker compose pull && docker compose up -d

# Clean up unused resources
docker system prune -a
```

### System Monitoring
```bash
# Check system resources
htop

# Check disk usage
df -h

# Monitor Docker stats
docker stats
```