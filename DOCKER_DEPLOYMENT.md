# Docker Deployment - Quick Start Guide

This guide provides a streamlined setup for deploying GitHub README Stats using Docker.

## 📋 Prerequisites

- **Docker** and **Docker Compose** installed on your server
- **GitHub account** with repository admin access
- **VPS/Server** with SSH access
- **Domain** pointing to your VPS (optional but recommended for HTTPS)
- **Reverse proxy** (Traefik, nginx, or Caddy) for SSL/TLS handling

## 🚀 Quick Setup (5 Minutes)

### Step 1: Configure Environment Variables

```bash
# Copy the example environment file
cp .env.example .env

# Edit with your values
nano .env
```

**Required values:**
```env
GITHUB_USER=your-github-username
REGISTRY_URL=ghcr.io/your-github-username
STATS_DOMAIN=stats.yourdomain.com
BIND_PORT=3000
GH_PAT_1=your_github_token
```

### Step 2: Create GitHub Personal Access Token

1. Go to: https://github.com/settings/tokens
2. Click **"Generate new token (classic)"**
3. Select scopes:
   - ✅ `public_repo` - Access public repositories
   - ✅ `read:user` - Read user profile data
4. Copy the token and set as `GH_PAT_1` in `.env`

### Step 3: Start the Container

```bash
# Pull the latest image
docker compose -f docker-compose.prod.yml pull

# Start in detached mode
docker compose -f docker-compose.prod.yml up -d

# Check logs
docker logs -f github-readme-stats
```

### Step 4: Verify Deployment

```bash
# Test the API endpoint
curl http://localhost:3000/api/?username=anuraghazra

# Should return SVG with GitHub stats
```

## 🔐 GitHub Secrets for CI/CD

If using the GitHub Actions workflow for automated deployment, add these secrets:

**Repository Settings → Secrets and variables → Actions**

| Secret Name | Description | Example |
|-------------|-------------|---------|
| `DEPLOY_DOMAIN` | Your deployment domain | `stats.yourdomain.com` |
| `BIND_PORT` | Port to bind on VPS | `3002` |
| `VPS_HOST` | VPS IP or hostname | `192.168.1.100` |
| `VPS_USER` | SSH username | `ubuntu` |
| `VPS_KEY` | SSH private key | `-----BEGIN OPENSSH...` |
| `VPS_PASSPHRASE` | SSH key passphrase | `your-passphrase` |
| `GHCR_TOKEN` | GitHub Container Registry token | `ghp_...` |
| `GH_PAT_1` | GitHub Personal Access Token | `ghp_...` |

## 🌐 DNS Configuration

Point your domain to your VPS:

```
Type: A
Name: stats (or your subdomain)
Value: [Your VPS IP]
TTL: 3600
```

**Result:** `stats.yourdomain.com` → Your VPS IP

## 🔧 Configuration Options

### Port Configuration

Change the bind port if 3000 is already in use:

```env
# .env
BIND_PORT=3002
```

Check for port conflicts:
```bash
# Linux
netstat -tulpn | grep :3000

# Windows
netstat -ano | findstr :3000
```

### Multiple GitHub Tokens (Higher Rate Limits)

Add up to 4 tokens for load balancing:

```env
GH_PAT_1=your_first_token
GH_PAT_2=your_second_token
GH_PAT_3=your_third_token
GH_PAT_4=your_fourth_token
```

**Rate Limits:**
- No token: 60 requests/hour
- 1 token: 5,000 requests/hour
- 4 tokens: 20,000 requests/hour

### Resource Limits

Default limits (defined in docker-compose.prod.yml):
- **CPU:** 0.5 cores max (0.25 guaranteed)
- **Memory:** 512MB max (256MB guaranteed)

Adjust in `docker-compose.prod.yml`:
```yaml
deploy:
  resources:
    limits:
      cpus: '1.0'      # Increase for more CPU
      memory: 1G       # Increase for more memory
    reservations:
      cpus: '0.5'
      memory: 512M
```

## 🔄 Updating Deployment

### Manual Update

```bash
# Pull latest image
docker compose -f docker-compose.prod.yml pull

# Recreate container
docker compose -f docker-compose.prod.yml up -d --force-recreate

# Verify
docker logs github-readme-stats
```

### Automatic Updates (Watchtower)

The container includes Watchtower label for automatic updates:
```yaml
- "com.centurylinklabs.watchtower.enable=true"
```

If Watchtower is running, it will automatically pull and restart on new images.

## 🐛 Troubleshooting

### Container Won't Start

```bash
# Check container status
docker ps -a | grep github-readme-stats

# View logs
docker logs github-readme-stats

# Check for port conflicts
netstat -tulpn | grep [YOUR_PORT]
```

### Stats Not Loading

```bash
# Check if service is responding
curl http://localhost:3000/api/

# Check GitHub token validity
docker logs github-readme-stats | grep -i "rate limit"

# Verify environment variables
docker exec github-readme-stats env | grep PAT
```

### Certificate Issues (HTTPS)

If using Traefik:
```bash
# Check Traefik is running
docker ps | grep traefik

# View Traefik logs
docker logs traefik | grep -i "stats.yourdomain.com"

# Verify DNS
dig stats.yourdomain.com
```

## 📊 Monitoring

### Health Check

The container includes a built-in health check:

```bash
# Check health status
docker inspect github-readme-stats --format='{{.State.Health.Status}}'

# View health check logs
docker inspect github-readme-stats --format='{{range .State.Health.Log}}{{.Output}}{{end}}'
```

### Resource Usage

```bash
# Monitor resource consumption
docker stats github-readme-stats

# View container details
docker inspect github-readme-stats
```

### API Requests

```bash
# Follow logs in real-time
docker logs -f github-readme-stats

# Filter for API requests
docker logs github-readme-stats | grep "GET /api"

# Check for errors
docker logs github-readme-stats | grep -i error
```

## 🔗 Using Your Self-Hosted Instance

Once deployed, use your domain in GitHub README:

```markdown
![GitHub Stats](https://stats.yourdomain.com/api/?username=YOUR_USERNAME&show_icons=true&theme=radical)
```

**Example:**
```markdown
![GitHub Stats](https://stats.yourdomain.com/api/?username=maxh33&count_private=true&show_icons=true&theme=apprentice)
```

## 🎯 API Endpoints

Your self-hosted instance supports all standard endpoints:

- **Stats Card:** `/api/?username=USERNAME`
- **Top Languages:** `/api/top-langs/?username=USERNAME`
- **Repo Card:** `/api/pin/?username=USERNAME&repo=REPO`
- **Gist Card:** `/api/gist?id=GIST_ID`
- **Wakatime:** `/api/wakatime?username=USERNAME`

See full API documentation: https://github.com/anuraghazra/github-readme-stats#all-demos

## 📚 Advanced Topics

### Using with Traefik

Example `docker-compose.prod.yml` labels for Traefik:
```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.github-readme-stats.rule=Host(`${STATS_DOMAIN}`)"
  - "traefik.http.routers.github-readme-stats.entrypoints=websecure"
  - "traefik.http.routers.github-readme-stats.tls.certresolver=myresolver"
```

### Using with nginx

Example nginx reverse proxy config:
```nginx
server {
    listen 443 ssl http2;
    server_name stats.yourdomain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Caching with Redis

For high-traffic deployments, consider adding Redis caching:
```yaml
services:
  redis:
    image: redis:7-alpine
    restart: unless-stopped

  github-readme-stats:
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
```

(Note: Requires code modifications to support Redis)

## 🆘 Getting Help

- **Docker Issues:** See [DOCKER_SETUP.md](DOCKER_SETUP.md) for detailed troubleshooting
- **Deployment Issues:** See [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md) for comprehensive guide
- **Post-Deployment:** See [POST_DEPLOYMENT_STEPS.md](POST_DEPLOYMENT_STEPS.md) for verification checklist

## 📝 Summary Checklist

- [ ] Docker and Docker Compose installed
- [ ] Created `.env` from `.env.example`
- [ ] Set `GITHUB_USER` and `REGISTRY_URL`
- [ ] Set `STATS_DOMAIN` (your custom domain)
- [ ] Created GitHub Personal Access Token
- [ ] Set `GH_PAT_1` in `.env`
- [ ] Configured DNS A record
- [ ] Started container: `docker compose -f docker-compose.prod.yml up -d`
- [ ] Verified logs: `docker logs github-readme-stats`
- [ ] Tested endpoint: `curl http://localhost:3000/api/`
- [ ] HTTPS working (if using reverse proxy)
- [ ] Updated GitHub README with self-hosted URL

---

**Ready to deploy?** Start with Step 1 above! 🚀

For questions or issues, check [DOCKER_SETUP.md](DOCKER_SETUP.md) for troubleshooting.
