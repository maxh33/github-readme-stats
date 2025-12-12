# Deployment Guide - GitHub Readme Stats

## Overview

This guide explains how to deploy github-readme-stats to your VPS using GitHub Actions CI/CD pipeline.

## Prerequisites

- VPS with Docker and Docker Compose installed
- SSH access to VPS with key-based authentication
- GitHub Container Registry (ghcr.io) access
- Shared monitoring stack with Traefik running on VPS (from my-portfolio setup)

## Required GitHub Secrets

You need to add these secrets to your GitHub repository:

### Already Configured ✅
- `VPS_HOST` - Your VPS IP address or hostname
- `VPS_USER` - SSH username for VPS
- `VPS_KEY` - SSH private key for authentication
- `VPS_PASSPHRASE` - SSH key passphrase (if your key is encrypted)

### Required - Need to Add ❌

#### 1. GHCR_TOKEN (GitHub Container Registry Token)
**Purpose:** Allows GitHub Actions to push Docker images to GitHub Container Registry

**How to create:**
1. Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click "Generate new token (classic)"
3. Name: `GHCR_TOKEN`
4. Scopes needed:
   - ✅ `write:packages` - Upload packages to GitHub Package Registry
   - ✅ `read:packages` - Download packages from GitHub Package Registry
   - ✅ `delete:packages` - Delete packages from GitHub Package Registry (optional)
5. Click "Generate token"
6. Copy the token immediately (you won't see it again!)
7. Go to your repository → Settings → Secrets and variables → Actions
8. Click "New repository secret"
9. Name: `GHCR_TOKEN`
10. Value: Paste the token
11. Click "Add secret"

#### 2. GH_PAT_1 (GitHub Personal Access Token #1)
**Purpose:** Primary token for GitHub API requests to fetch user statistics

**How to create:**
1. Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click "Generate new token (classic)"
3. Name: `github-readme-stats-api-1`
4. Scopes needed:
   - ✅ `public_repo` - Access public repositories
   - ✅ `read:user` - Read user profile data
5. Expiration: No expiration (or set to 1 year and renew)
6. Click "Generate token"
7. Copy the token
8. Add to GitHub repository secrets as `GH_PAT_1`

#### 3. GH_PAT_2, GH_PAT_3, GH_PAT_4 (Optional Additional Tokens)
**Purpose:** Additional tokens for load balancing and higher rate limits

**Note:** These are optional. Start with just `GH_PAT_1`. Add more tokens if you need higher rate limits.

- Without tokens: 60 requests/hour
- With 1 token: 5,000 requests/hour
- With 4 tokens: 20,000 requests/hour

Create these the same way as `GH_PAT_1` if needed.

## GitHub Secrets Summary

### Current Status

| Secret Name      | Status | Purpose                          |
|------------------|--------|----------------------------------|
| VPS_HOST         | ✅      | VPS connection                   |
| VPS_USER         | ✅      | VPS SSH username                 |
| VPS_KEY          | ✅      | VPS SSH private key              |
| VPS_PASSPHRASE   | ✅      | VPS SSH key passphrase           |
| GHCR_TOKEN       | ❌      | Push Docker images to ghcr.io    |
| GH_PAT_1         | ❌      | GitHub API access (primary)      |
| GH_PAT_2         | 🔶      | GitHub API access (optional)     |
| GH_PAT_3         | 🔶      | GitHub API access (optional)     |
| GH_PAT_4         | 🔶      | GitHub API access (optional)     |

Legend:
- ✅ Already configured
- ❌ Required - must add
- 🔶 Optional - add if you need higher rate limits

## Deployment Architecture

```
GitHub Actions (CI/CD)
    ↓
Build Docker Image (Dockerfile.prod)
    ↓
Push to ghcr.io/maxh33/github-readme-stats:prod
    ↓
SSH to VPS
    ↓
Pull image and start container
    ↓
Traefik (Reverse Proxy)
    ↓
https://stats.maxhaider.dev
```

## Deployment Flow

### 1. Push to Repository
When you push to `master` or `main` branch, GitHub Actions automatically:

1. **Build Stage:**
   - Builds Docker image using `Dockerfile.prod`
   - Tags as `ghcr.io/maxh33/github-readme-stats:prod`
   - Pushes to GitHub Container Registry

2. **Deploy Stage:**
   - SSHs into your VPS
   - Copies deployment files (docker-compose.prod.yml, docs)
   - Creates `.env` file with secrets
   - Pulls latest image from ghcr.io
   - Stops old container
   - Starts new container
   - Verifies deployment

### 2. Container Configuration
The production container:
- Connects to `monitoring_shared_monitoring_network` (Traefik network)
- Binds to `127.0.0.1:3001` (not exposed publicly)
- Traefik routes `stats.maxhaider.dev` → container
- Uses non-root user (UID 1001) for security
- Read-only filesystem with tmpfs for temp files
- Resource limits: 0.5 CPU, 512MB RAM

### 3. DNS Configuration

**IMPORTANT:** You need to configure DNS for the stats subdomain.

Add an A record in your DNS provider:
```
Type: A
Name: stats
Value: [Your VPS IP]
TTL: 3600 (or default)
```

Result: `stats.maxhaider.dev` → Your VPS IP

Traefik will automatically handle SSL/TLS certificate via Let's Encrypt.

## Files Overview

### Production Files
- `Dockerfile.prod` - Production-optimized Docker image
- `docker-compose.prod.yml` - Production Docker Compose configuration
- `.github/workflows/deploy.yml` - CI/CD pipeline

### Documentation Files
- `DOCKER_SETUP.md` - Docker troubleshooting and local setup
- `claude.md` - Project overview and development guide
- `DEPLOYMENT_GUIDE.md` - This file

## Testing Deployment

### Local Test (Before Pushing)
```bash
# Build production image locally
docker build -t github-readme-stats:test -f Dockerfile.prod .

# Test the image
docker run -p 3000:3000 \
  -e PAT_1=your_github_token \
  github-readme-stats:test

# Test endpoint
curl "http://localhost:3000/api/?username=anuraghazra"
```

### After Deployment

1. **Check GitHub Actions:**
   - Go to repository → Actions tab
   - Verify workflow completed successfully
   - Check logs for any errors

2. **SSH to VPS and verify:**
```bash
ssh your-vps-user@your-vps-ip

# Check container is running
docker ps --filter "name=github-readme-stats"

# Check logs
docker logs github-readme-stats

# Test locally on VPS
curl "http://localhost:3001/api/"
```

3. **Test production URL:**
```bash
curl "https://stats.maxhaider.dev/api/?username=anuraghazra"
```

## Monitoring

The container is configured with Prometheus labels for monitoring:
```yaml
- "prometheus.scrape=true"
- "prometheus.port=3000"
- "prometheus.path=/api/metrics"
```

Access monitoring through your existing Grafana dashboard at `https://monitor.maxhaider.dev`

## Troubleshooting

### Workflow Fails at Build Stage
**Cause:** Missing GHCR_TOKEN secret

**Solution:**
1. Create GitHub personal access token with `write:packages` scope
2. Add as `GHCR_TOKEN` secret in repository settings

### Workflow Fails at Deploy Stage
**Cause:** Missing GH_PAT_1 secret

**Solution:**
1. Create GitHub personal access token with `public_repo` and `read:user` scopes
2. Add as `GH_PAT_1` secret in repository settings

### Container Not Starting
**Cause:** Could be several reasons

**Debug steps:**
```bash
# SSH to VPS
ssh your-user@your-vps-ip

cd ~/github-readme-stats

# Check container status
docker ps -a --filter "name=github-readme-stats"

# Check logs
docker logs github-readme-stats

# Common issues:
# 1. Missing .env file - check if it was created
cat .env

# 2. Traefik not running
docker ps | grep traefik

# 3. Image pull failed
docker compose pull
```

### Stats Not Loading
**Cause:** Invalid or missing GitHub PAT tokens

**Solution:**
1. Check tokens are valid and not expired
2. Verify tokens have correct scopes
3. Check container logs for API errors:
```bash
docker logs github-readme-stats | grep -i error
```

### SSL Certificate Issues
**Cause:** Traefik not able to issue certificate

**Solution:**
1. Verify DNS is configured correctly:
```bash
dig stats.maxhaider.dev
# Should return your VPS IP
```

2. Check Traefik logs:
```bash
cd ~/shared/monitoring
docker logs traefik
```

3. Ensure port 443 is open on your VPS firewall

## Updating Deployment

To update the deployment, simply push to master/main branch:

```bash
git add .
git commit -m "Update configuration"
git push origin master
```

GitHub Actions will automatically:
1. Build new Docker image
2. Push to registry
3. Deploy to VPS
4. Restart container

## Manual Deployment (Emergency)

If CI/CD fails, you can deploy manually:

```bash
# SSH to VPS
ssh your-user@your-vps-ip

cd ~/github-readme-stats

# Pull latest code manually
# (or copy files via scp)

# Rebuild and restart
docker compose -f docker-compose.prod.yml down
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d --force-recreate
```

## Security Considerations

1. **Secrets Management:**
   - Never commit `.env` files
   - Never expose PAT tokens in logs
   - Rotate tokens periodically

2. **Container Security:**
   - Runs as non-root user (UID 1001)
   - Read-only filesystem
   - Minimal capabilities
   - Resource limits enforced

3. **Network Security:**
   - Only accessible via Traefik
   - Not exposed directly to internet
   - SSL/TLS enforced

## Rate Limits

GitHub API rate limits per hour:
- No authentication: 60 requests
- 1 PAT token: 5,000 requests
- 4 PAT tokens: 20,000 requests

Monitor your rate limit usage in container logs.

## Next Steps

1. ✅ Add `GHCR_TOKEN` secret to GitHub repository
2. ✅ Add `GH_PAT_1` secret to GitHub repository
3. ✅ Configure DNS for `stats.maxhaider.dev`
4. ✅ Push to master branch to trigger deployment
5. ✅ Monitor deployment in GitHub Actions
6. ✅ Test production URL
7. ✅ Set up monitoring alerts (optional)

## Support

For issues or questions:
1. Check [DOCKER_SETUP.md](DOCKER_SETUP.md) for Docker-specific issues
2. Check [claude.md](claude.md) for project architecture
3. Review GitHub Actions logs for deployment errors
4. Check container logs on VPS for runtime errors
