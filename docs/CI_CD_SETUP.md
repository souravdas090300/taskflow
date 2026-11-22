# CI/CD Setup Guide - GitHub Actions Deployment

This guide explains how to set up automated deployment of TaskFlow to your VPS using GitHub Actions.

## Overview

The CI/CD pipeline will:
1. Build both TaskFlow API and Frontend Docker images
2. Push images to GitHub Container Registry (GHCR)
3. SSH into your VPS
4. Pull and deploy the new images
5. Run database migrations
6. Restart all services

## Prerequisites

- GitHub repository with TaskFlow code
- VPS with SSH access
- Docker and Docker Compose installed on VPS
- Git installed on VPS

## Step 1: Set Up VPS

### 1.1 Create Deployment Directory

SSH into your VPS and create the deployment directory:

```bash
# SSH into your VPS
ssh your-vps-user@your-vps-ip

# Create deployment directory
mkdir -p ~/taskflow
cd ~/taskflow

# Clone your repository
git clone https://github.com/YOUR_USERNAME/TaskFlow.git .
```

### 1.2 Install Docker and Docker Compose

```bash
# Update package list
sudo apt update

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose
sudo apt install docker-compose-plugin

# Add your user to docker group
sudo usermod -aG docker $USER

# Log out and back in for group changes to take effect
```

### 1.3 Verify Installation

```bash
docker --version
docker compose version
```

## Step 2: Generate SSH Key

### 2.1 Generate SSH Key Pair

On your local machine:

```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/taskflow_deploy

# Copy public key to VPS
ssh-copy-id -i ~/.ssh/taskflow_deploy.pub your-vps-user@your-vps-ip
```

### 2.2 Test SSH Connection

```bash
ssh -i ~/.ssh/taskflow_deploy your-vps-user@your-vps-ip
```

## Step 3: Configure GitHub Secrets

Go to your GitHub repository: **Settings → Secrets and variables → Actions → New repository secret**

Add the following secrets:

### Required Secrets

| Secret Name | Description | Example |
|-------------|-------------|---------|
| `VPS_HOST_IP` | Your VPS public IP address | `92.5.144.125` |
| `VPS_USERNAME` | VPS username | `ubuntu` |
| `VPS_SSH_PRIVATE_KEY` | Private SSH key content | Full content of `~/.ssh/taskflow_deploy` |
| `DB_PASSWORD` | PostgreSQL database password | Strong password (avoid `$` — see note) |
| `JWT_KEY` | JWT signing key (min 32 chars) | Long random string |
| `APP_PUBLIC_URL` | Public site URL | `https://taskflow-das.duckdns.org` |
| `GOOGLE_CLIENT_ID` | Google OAuth client ID | `123456789.apps.googleusercontent.com` |
| `GOOGLE_CLIENT_SECRET` | Google OAuth client secret | `GOCSPX-abc123` |
| `STRIPE_SECRET_KEY` | Stripe secret key (test mode) | `sk_test_...` |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook signing secret | `whsec_...` |
| `RESEND_API_KEY` | Resend API key | `re_...` |
| `RESEND__FROMEMAIL` | Resend sender address | `onboarding@resend.dev` or `noreply@yourdomain.com` |
| `DUCKDNS_SUBDOMAIN` | DuckDNS subdomain | `taskflow-das` |
| `DUCKDNS_TOKEN` | DuckDNS authentication token | Your token from duckdns.org |

> **Notes:**
> - `GOOGLE_CLIENT_ID` is reused as the frontend build arg `VITE_GOOGLE_CLIENT_ID`, so the GIS button and API audience check always match.
> - `RESEND__FROMEMAIL` is the actual secret name (double underscore). Store the **literal email address** — never `${{ secrets.RESEND_FROM_EMAIL }}`; an unevaluated expression once landed verbatim in `.env` and broke all email sends.
> - Secret values are written to `.env` with single quotes and `$` escaped to `$$` for Compose interpolation. Avoid `$` in `DB_PASSWORD`/`JWT_KEY` to keep values predictable.

### How to Add SSH Private Key

```bash
# On your local machine, copy the private key
cat ~/.ssh/taskflow_deploy

# Copy the entire output (including BEGIN/END lines)
# Paste it into the VPS_SSH_PRIVATE_KEY secret in GitHub
```

## Step 4: Enable GitHub Actions

### 4.1 Enable Actions for Repository

Go to: **Settings → Actions → General**

Under "Actions permissions":
- Select: ✅ "Allow all actions and reusable workflows"

### 4.2 Grant GITHUB_TOKEN Permissions

Still in **Settings → Actions → General**:

Under "Workflow permissions":
- Select: ✅ "Read and write permissions"
- Check: ✅ "Allow GitHub Actions to create and approve pull requests"

## Step 5: Initial Manual Deployment

Before CI/CD works, you need to do an initial manual setup on the VPS:

### 5.1 Create .env File on VPS

The deploy workflow generates `.env` on the VPS from GitHub secrets automatically — a manual `.env` is only needed for the very first bootstrap before CI/CD runs. If you create one, use the production keys (`JWT__Key`, `Stripe__SecretKey`, `Resend__ApiKey`, `Resend__FromEmail`, `App__PublicUrl`, `Authentication__Google__ClientId/ClientSecret`, `ConnectionStrings__DefaultConnection`):

```bash
# SSH into VPS
ssh your-vps-user@your-vps-ip
cd ~/taskflow

cat > .env << 'EOF'
DB_PASSWORD=your-strong-password-here
JWT__Key=your-jwt-signing-key
App__PublicUrl=https://taskflow-das.duckdns.org
EOF
# add the remaining keys from .env.prod.example as needed
```

### 5.2 Initial Deploy

```bash
# Login to GitHub Container Registry
echo "YOUR_GITHUB_TOKEN" | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin

# Pull images manually (if they exist)
docker compose -f docker-compose.prod.yml pull

# Start services
docker compose -f docker-compose.prod.yml up -d
```

## Step 6: Configure Nginx Proxy Manager

### 6.1 Access Nginx Proxy Manager

Open your browser: `http://your-vps-ip:81`

**Default credentials:**
- Email: `admin@example.com`
- Password: `changeme`

**Change these immediately after first login!**

### 6.2 Create Proxy Hosts

#### Proxy Host 1: Frontend
- **Domain Names:** `taskflow-das.duckdns.org`
- **Scheme:** `http`
- **Forward Hostname:** `taskflow-frontend`
- **Forward Port:** `80`
- **SSL Certificate:**
  - ✅ Request a new SSL certificate
  - ✅ Force SSL
  - ✅ HTTP/2 Support
  - ✅ HSTS Enabled

#### Proxy Host 2: API
- **Domain Names:** `api.taskflow-das.duckdns.org`
- **Scheme:** `http`
- **Forward Hostname:** `taskflow-api`
- **Forward Port:** `80`
- **SSL Certificate:**
  - ✅ Request a new SSL certificate
  - ✅ Force SSL
  - ✅ HTTP/2 Support
  - ✅ HSTS Enabled

### 6.3 Configure Google Sign-In

The app uses **Google Identity Services** (ID-token button), not the OAuth redirect flow. In Google Cloud Console → Credentials → your OAuth client, configure **Authorized JavaScript origins** (no path):

```
https://taskflow-das.duckdns.org
```

Also add `http://localhost:5173` for local development. Redirect URIs are not needed for this flow. See [GOOGLE_AUTH_SETUP.md](GOOGLE_AUTH_SETUP.md).

## Step 7: Trigger CI/CD

### Automatic Trigger

Push to main branch:
```bash
git add .
git commit -m "Your commit message"
git push origin main
```

### Manual Trigger

Go to: **Actions → Deploy TaskFlow to Production VPS → Run workflow**

## Step 8: Monitor Deployment

### View Workflow Logs

Go to: **Actions → Deploy TaskFlow to Production VPS → Click on the run**

### Check VPS Status

```bash
# SSH into VPS
ssh your-vps-user@your-vps-ip
cd ~/taskflow

# Check running containers
docker compose -f docker-compose.prod.yml ps

# View logs
docker compose -f docker-compose.prod.yml logs -f
```

## Troubleshooting

### Issue: SSH Connection Failed

**Solution:**
1. Verify SSH key is correct in GitHub secrets
2. Check VPS firewall allows SSH (port 22)
3. Test SSH connection manually: `ssh -i ~/.ssh/taskflow_deploy your-vps-user@your-vps-ip`

### Issue: Docker Build Failed

**Solution:**
1. Check Dockerfile syntax
2. Verify .github/workflows/deploy.yml has correct context paths
3. Check Actions logs for specific error

### Issue: Container Registry Login Failed

**Solution:**
1. Verify repository is public or you have correct permissions
2. Check GITHUB_TOKEN has read/write permissions
3. Ensure workflow permissions are set correctly

### Issue: Database Migration Failed

**Solution:**
1. Check database is running: `docker compose -f docker-compose.prod.yml ps db`
2. Verify DB_PASSWORD is correct in secrets
3. Check database logs: `docker compose -f docker-compose.prod.yml logs db`

### Issue: Nginx Proxy Manager Not Accessible

**Solution:**
1. Check container is running: `docker compose -f docker-compose.prod.yml ps nginx-proxy-manager`
2. Verify port 81 is not blocked by firewall
3. Check container logs: `docker compose -f docker-compose.prod.yml logs nginx-proxy-manager`

## Security Best Practices

1. **Rotate Secrets Regularly**
   - Change database passwords every 90 days
   - Rotate SSH keys periodically
   - Update DuckDNS tokens if exposed

2. **Use Strong Passwords**
   - Minimum 16 characters
   - Mixed case, numbers, symbols
   - Use a password manager

3. **Enable Firewall**
   ```bash
   # On VPS
   sudo ufw allow 22/tcp    # SSH
   sudo ufw allow 80/tcp    # HTTP
   sudo ufw allow 443/tcp   # HTTPS
   sudo ufw allow 81/tcp    # Nginx Proxy Manager admin
   sudo ufw enable
   ```

4. **Regular Backups**
   ```bash
   # Backup database
   docker compose -f docker-compose.prod.yml exec db pg_dump -U taskflow taskflow > backup.sql

   # Backup volumes
   docker run --rm -v taskflow_postgres-data:/data -v $(pwd):/backup ubuntu tar czf /backup/postgres-backup.tar.gz /data
   ```

## Rollback Procedure

If deployment fails:

```bash
# SSH into VPS
ssh your-vps-user@your-vps-ip
cd ~/taskflow

# Rollback to previous image
docker compose -f docker-compose.prod.yml down
docker compose -f docker-compose.prod.yml up -d

# Or use specific image tag
docker compose -f docker-compose.prod.yml up -d --scale api=0
docker compose -f docker-compose.prod.yml up -d
```

## Performance Optimization

### 1. Enable Build Caching

The workflow already uses GitHub Actions cache for faster builds.

### 2. Use Multi-Stage Builds

Your Dockerfiles already use multi-stage builds for smaller images.

### 3. Optimize Database

```yaml
# In docker-compose.prod.yml, add to db service:
command:
  - "postgres"
  - "-c"
  - "shared_buffers=256MB"
  - "-c"
  - "max_connections=100"
```

## Monitoring

### 1. Container Health Checks

```bash
# Check all container health
docker compose -f docker-compose.prod.yml ps
```

### 2. Resource Usage

```bash
# Check container resource usage
docker stats
```

### 3. Logs Aggregation

Consider setting up centralized logging (e.g., ELK Stack, Loki, or CloudWatch).

## Related Documentation

- [Environment Setup](ENVIRONMENT_SETUP.md)
- [Production Deployment](PRODUCTION_DEPLOYMENT.md)
- [Docker Deployment](DEPLOYMENT_DOCKER.md)
- [Security Audit](../SECURITY_AUDIT.md)
