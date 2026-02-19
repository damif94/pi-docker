# SearXNG - Privacy-Respecting Metasearch Engine

SearXNG is a free, privacy-respecting metasearch engine that aggregates results from multiple search engines without tracking users.

## Features

- Privacy-focused (no tracking, no user profiling)
- Aggregates results from 70+ search engines
- No ads or monetization
- Highly customizable
- Fast and lightweight
- Redis caching for improved performance
- Mobile-friendly interface

## Hardware Requirements

- **Minimum RAM**: 512 MB
- **Recommended RAM**: 1 GB+
- **Storage**: ~200 MB for application + cache
- **CPU**: Low CPU usage, suitable for Raspberry Pi 4
- **Architecture**: ARM64 compatible

## Port Configuration

- **Port 8080**: SearXNG web interface (HTTP)
- **Internal**: Redis cache (port 6379, not exposed)

## Environment Variables

Add the following to your root `.env` file:

```bash
# SearXNG Configuration
SEARXNG_BASE_URL=https://search.damianferencz.org
SEARXNG_SECRET_KEY=your_secure_secret_key_here
```

### Generating Secret Key

Generate a secure secret key using:

```bash
openssl rand -hex 32
```

Or using Python:

```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```

## Initial Setup

### 1. Prepare Data Directory

SSH into your Raspberry Pi:

```bash
ssh damian@damianferencz.org
cd /home/damian/docker
```

Create and set permissions for data directories:

```bash
sudo mkdir -p /srv/toshiba/data/searxng/redis
sudo chown -R 1000:1000 /srv/toshiba/data/searxng
```

### 2. Configure Environment Variables

Ensure your `/home/damian/docker/.env` file contains:

```bash
SEARXNG_BASE_URL=https://search.damianferencz.org
SEARXNG_SECRET_KEY=<your_generated_secret_key>
```

### 3. Deploy SearXNG

Navigate to the searxng directory and start the services:

```bash
cd /home/damian/docker/searxng
docker compose up -d --env-file /home/damian/docker/.env
```

### 4. Verify Deployment

Check the logs to ensure everything started correctly:

```bash
docker compose logs -f
```

You should see:
- SearXNG starting on port 8080
- Redis successfully connecting

### 5. Configure Nginx Proxy Manager

1. Access Nginx Proxy Manager at `http://your-pi-ip:81`
2. Add a new Proxy Host:
   - **Domain Names**: `search.damianferencz.org`
   - **Scheme**: `http`
   - **Forward Hostname/IP**: `searxng` (container name)
   - **Forward Port**: `8080`
   - **Cache Assets**: Enabled
   - **Block Common Exploits**: Enabled
   - **Websockets Support**: Disabled
3. Configure SSL:
   - Request a new Let's Encrypt certificate
   - Force SSL: Enabled
   - HTTP/2 Support: Enabled
   - HSTS: Enabled

## Configuration

### Custom Settings

SearXNG configuration is stored in `/srv/toshiba/data/searxng/settings.yml`. On first run, a default configuration is created.

To customize SearXNG:

1. Stop the service:
   ```bash
   docker compose down
   ```

2. Edit the settings file on the Pi:
   ```bash
   sudo nano /srv/toshiba/data/searxng/settings.yml
   ```

3. Restart the service:
   ```bash
   docker compose up -d
   ```

### Key Configuration Options

Common settings you might want to customize:

```yaml
# General settings
general:
  instance_name: "SearXNG"
  privacypolicy_url: false
  donation_url: false
  contact_url: false
  enable_metrics: false

# Server settings
server:
  secret_key: "${SEARXNG_SECRET_KEY}"
  limiter: true
  image_proxy: true
  method: "POST"

# Search settings
search:
  safe_search: 0  # 0=off, 1=moderate, 2=strict
  autocomplete: "google"
  default_lang: "en"
  formats:
    - html
    - json

# UI settings
ui:
  default_locale: "en"
  theme_args:
    simple_style: auto  # auto, light, dark
  query_in_title: true
  infinite_scroll: false
  center_alignment: false

# Redis configuration (for caching)
redis:
  url: redis://searxng-redis:6379/0
```

### Enabling/Disabling Engines

To customize which search engines to use, edit the `engines:` section in `settings.yml`:

```yaml
engines:
  - name: google
    disabled: false
    weight: 1

  - name: bing
    disabled: true  # Set to false to enable

  - name: duckduckgo
    disabled: false
```

## Usage

### Accessing SearXNG

Once deployed and configured through Nginx Proxy Manager:
- **External**: https://search.damianferencz.org
- **Local (development)**: http://your-pi-ip:8080

### Search Syntax

SearXNG supports various search operators:

- `!bang` - Direct search on specific engines (e.g., `!g` for Google)
- `site:example.com` - Search within specific site
- `filetype:pdf` - Search for specific file types
- `"exact phrase"` - Search for exact phrase

### Setting as Default Search Engine

#### Firefox
1. Visit your SearXNG instance
2. Right-click the address bar
3. Click "Add Search Engine"

#### Chrome/Edge
1. Go to Settings → Search Engine → Manage search engines
2. Click "Add"
3. Fill in:
   - **Name**: SearXNG
   - **Keyword**: search
   - **URL**: `https://search.damianferencz.org/?q=%s`

## Maintenance

### View Logs

```bash
cd /home/damian/docker/searxng
docker compose logs -f
```

### Restart Service

```bash
docker compose restart
```

### Update to Latest Version

```bash
docker compose pull
docker compose up -d
```

Watchtower will automatically update the containers if configured.

### Backup Configuration

Backup your settings:

```bash
sudo cp /srv/toshiba/data/searxng/settings.yml /srv/toshiba/data/searxng/settings.yml.backup
```

## Troubleshooting

### Service Won't Start

Check logs for errors:
```bash
docker compose logs searxng
```

Common issues:
- **Permission denied**: Ensure `/srv/toshiba/data/searxng` is owned by `1000:1000`
- **Port already in use**: Check if port 8080 is available
- **Secret key missing**: Verify `SEARXNG_SECRET_KEY` is set in `.env`

### No Search Results

1. Check if search engines are enabled in `settings.yml`
2. Verify internet connectivity from container
3. Check if you're being rate-limited by search engines
4. Try increasing timeout values in settings

### Slow Performance

1. Verify Redis is running:
   ```bash
   docker compose ps
   ```

2. Check Redis connectivity:
   ```bash
   docker compose exec searxng-redis redis-cli ping
   ```
   Should return `PONG`

3. Reduce number of enabled engines in `settings.yml`

### Rate Limiting

If you're being rate-limited:
1. Enable fewer search engines
2. Increase request delays in `settings.yml`
3. Consider using a VPN or proxy

## Security Considerations

- SearXNG runs with minimal capabilities (see `cap_drop`/`cap_add` in docker-compose)
- No user data is stored or tracked
- All searches are proxied (no direct connection to search engines)
- Image proxy enabled to prevent tracking via images
- Limiter enabled to prevent abuse

## Resources

- **Official Documentation**: https://docs.searxng.org/
- **GitHub Repository**: https://github.com/searxng/searxng
- **Supported Engines**: https://docs.searxng.org/user/configured_engines.html
- **Public Instances**: https://searx.space/

## Performance Notes

### On Raspberry Pi 4

- Expected RAM usage: 100-200 MB (SearXNG) + 20-50 MB (Redis)
- Search latency: 1-3 seconds (depends on enabled engines and network)
- Concurrent users: Can handle 5-10 simultaneous searches comfortably
- Redis caching significantly improves repeat query performance

## Advanced Configuration

### Custom Themes

SearXNG supports custom themes. Place theme files in:
```
/srv/toshiba/data/searxng/themes/
```

### Adding Custom Engines

You can define custom search engines in `settings.yml`. See [documentation](https://docs.searxng.org/dev/engines/index.html) for details.

### JSON API

SearXNG provides a JSON API for programmatic access:

```bash
curl 'https://search.damianferencz.org/search?q=raspberry+pi&format=json'
```

### OpenSearch Integration

SearXNG supports OpenSearch XML for browser integration. Access at:
```
https://search.damianferencz.org/opensearch.xml
```

## Network Architecture

```
Internet
    ↓
Nginx Proxy Manager (ports 80/443)
    ↓ (internal, port 8080)
SearXNG
    ↓ (internal redis connection)
Redis Cache
```

All external traffic goes through Nginx Proxy Manager with SSL/TLS termination.
