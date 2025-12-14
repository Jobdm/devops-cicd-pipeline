# CI/CD Pipeline Documentation

## Overview
Automated pipeline that builds, tests, and deploys the Task Manager application using Jenkins.

## Architecture

```
Code Change → Jenkins Build → Docker Image → Production VM → Live App
```

## Pipeline Stages

### 1. Build Docker Image
- Builds Docker image from `docker/Dockerfile`
- Tags with build number: `task-manager:BUILD_NUMBER`
- Tags as latest: `task-manager:latest`
- Uses Docker layer caching for speed

### 2. Test
- Starts container on port 5001
- Performs health check at `/health`
- Fails build if health check fails
- Cleans up test container
- **Zero failed tests reach production**

### 3. Save Docker Image
- Exports image to `/tmp/task-manager.tar`
- ~50-100MB tar file
- Enables transfer to Production

### 4. Deploy to Production
- Transfers tar to Production via SCP
- Loads image: `docker load`
- Gracefully stops old container
- Starts new container on port 80
- Verifies with health check
- Auto-restart on failure enabled

## Infrastructure

### Jenkins Server
- **IP:** 192.168.122.179
- **Web UI:** http://192.168.122.179:8080
- **Credentials:** admin / admin123
- **Resources:** 4GB RAM, 2 CPUs
- **Disk:** 30GB

### Production Server
- **IP:** 192.168.122.241
- **Application:** http://192.168.122.241
- **Container:** task-manager (port 80)
- **Resources:** 2GB RAM, 1 CPU
- **Disk:** 20GB

## Credentials & Access

### SSH Keys
- **Jenkins → Production:** /home/devops/.ssh/id_ed25519 (on Jenkins VM)
- **Jenkins Credential ID:** production-ssh-key
- **User:** devops (passwordless sudo)

### Ports
- Jenkins Web UI: 8080
- Application: 80 (Production VM)
- Health Check: http://192.168.122.241/health

## Monitoring

### Health Checks
- **Frequency:** Every 5 minutes
- **Location:** Jenkins VM cron
- **Log:** /var/log/production-health.log
- **Endpoint:** http://192.168.122.241/health

### View Logs
```bash
# Production container logs
ssh production "docker logs task-manager"

# Health check logs
ssh jenkins "sudo tail -f /var/log/production-health.log"

# Jenkins build logs
# Web UI → Build Number → Console Output
```

## Manual Operations

### Trigger Build
1. Jenkins Dashboard
2. task-manager-pipeline
3. Build Now

### Deploy New Version
1. Update code in Jenkins workspace
2. Trigger build
3. Wait 2-3 minutes
4. Verify at http://192.168.122.241

### Restart Production App
```bash
ssh production "docker restart task-manager"
```

### View Production Status
```bash
ssh production "docker ps"
ssh production "docker logs task-manager"
curl http://192.168.122.241/health
```

## Performance Metrics

- **Build Time:** 1-2 minutes (after first build)
- **Deployment Time:** 30-60 seconds
- **Total Pipeline:** ~2-3 minutes
- **Downtime:** ~5 seconds (container restart)

## Troubleshooting

### Build Fails at Docker Stage
```bash
# Verify Docker running
ssh jenkins "docker ps"

# Check jenkins in docker group
ssh jenkins "groups jenkins | grep docker"

# Restart Jenkins
ssh jenkins "sudo systemctl restart jenkins"
```

### Deployment Fails (SSH Error)
```bash
# Test SSH connection
ssh jenkins "ssh devops@192.168.122.241 'docker ps'"

# Verify credential exists
# Jenkins → Manage → Credentials → production-ssh-key
```

### Application Not Responding
```bash
# Check container
ssh production "docker ps -a"
ssh production "docker logs --tail 50 task-manager"

# Restart container
ssh production "docker restart task-manager"

# Manual redeploy
# Jenkins → Build Now
```

### Production VM Down
```bash
# Check VM status
sudo virsh list --all

# Start if stopped
sudo virsh start production-server

# Wait 1 minute, then rebuild
```
