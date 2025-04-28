# Docker Disk Space Management Guide for Dokploy/Ubuntu Servers

## Table of Contents
- [Problem Overview](#problem-overview)
- [Diagnosis Steps](#diagnosis-steps)
  - [1. Check System Disk Usage](#1-check-system-disk-usage)
  - [2. List Docker Artifacts](#2-list-docker-artifacts)
- [Cleanup Procedures](#cleanup-procedures)
  - [1. Remove Stopped Containers](#1-remove-stopped-containers)
  - [2. Clean Unused Images](#2-clean-unused-images)
  - [3. Clean Unused Volumes](#3-clean-unused-volumes)
  - [4. Additional Cleanup Options](#4-additional-cleanup-options)
- [Prevention Strategies](#prevention-strategies)
  - [1. Docker Daemon Configuration](#1-docker-daemon-configuration)
  - [2. Deployment Best Practices](#2-deployment-best-practices)
- [Automation Setup](#automation-setup)
  - [1. Cron Job for Regular Cleanup](#1-cron-job-for-regular-cleanup)
  - [2. Post-Deployment Hooks](#2-post-deployment-hooks)
  - [3. Alternative Tools](#3-alternative-tools)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)
- [FAQs](#faqs)

---

## Problem Overview

Disk space gradually increases with each Docker deployment due to:

- **Dangling Docker Images**: Untagged layers from previous builds (10-100MB+ each)
- **Orphaned Containers**: Stopped containers with lingering writable layers (5-50MB each)
- **Unused Volumes**: Persistent storage from old deployments (100MB-10GB+)
- **Build Cache**: Accumulated intermediate layers (500MB-5GB+)
- **Unrotated Logs**: Growing container log files (1MB-1GB+ per container)

---

## Diagnosis Steps

### 1. Check System Disk Usage

```bash
# Overall disk usage
df -h

# Docker-specific usage
docker system df
```
- Sample output analysis:
```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          15        3         8.5GB     7.2GB (84%)  # 7.2GB can be cleaned!
Containers      10        2         500MB     500MB (100%)
Local Volumes   5         2         1.2GB     1.2GB (100%)
```

## 2. List Docker Artifacts

```bash
# All images (look for <none> tags)
docker images -a

# All containers (look for "Exited" status)
docker ps -a

# Dangling volumes
docker volume ls -f dangling=true

# Check log sizes
sudo du -sh /var/lib/docker/containers/*/*-json.log | sort -h
```

# Cleanup Procedures

1. Remove Stopped Containers

```bash
# Remove all stopped containers
docker container prune -f

# Remove specific container
docker rm [container_id]
```

2. Clean Unused Images

```bash
# Remove dangling images only (safe)
docker image prune -f

# Remove ALL unused images (more aggressive)
docker image prune -a -f

# Remove images older than 24h
docker image prune -a -f --filter "until=24h"
```

3. Clean Unused Volumes

```bash
# Remove all unused volumes
docker volume prune -f

# Remove specific volume
docker volume rm [volume_name]
```

4. Additional Cleanup Options

```bash
# Build cache
docker builder prune -f

# Networks
docker network prune -f

# Complete system cleanup (careful!)
docker system prune -a -f --volumes
```

# Prevention Strategies

1. Docker Daemon Configuration 
```/etc/docker/daemon.json:```

```bash
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "compress": "true"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
```

Apply changes:

```bash
sudo systemctl restart docker
```

2. Deployment Best Practices

- Image Management:

```bash
# Use consistent tags
docker build -t myapp:latest .

# Multi-stage builds
FROM node:18 as builder
WORKDIR /app
COPY . .
RUN npm build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

- Container Management:

```bash
# Auto-remove containers
docker run --rm myapp:latest

# Stop and remove old containers
docker stop old_container && docker rm old_container
```

# Automation Setup

1. Cron Job for Regular Cleanup

```bash
# Edit crontab
crontab -e

# Add daily cleanup at 2AM
0 2 * * * /usr/bin/docker system prune -f --filter "until=24h"
```

2. Post-Deployment Hooks
In your deployment script:

```bash
#!/bin/bash
# Deployment commands...
docker-compose up -d --build

# Cleanup
docker image prune -f
docker builder prune -f
```

3. Alternative Tools

- Docuum (LRU-based cleanup):

```bash
cargo install docuum
docuum --threshold 10GB
```

- docker-gc (Spotify's solution):

```bash
curl -L https://raw.githubusercontent.com/spotify/docker-gc/master/docker-gc > /usr/local/bin/docker-gc
chmod +x /usr/local/bin/docker-gc
docker-gc
```

# Monitoring

```bash
# Quick status check
docker system df && df -h /var/lib/docker

# Set up alerts (example for Nagios)
command[check_docker_disk]=/usr/lib/nagios/plugins/check_disk -w 20% -c 10% -p /var/lib/docker
```

# Troubleshooting

Problem: Cleanup doesn't free expected space
Solution:

Check for mounted volumes: ```mount | grep docker```

Verify storage driver: ```docker info | grep "Storage Driver"```

Check for systemd mount leaks: ```systemctl list-units --type mount```


Problem: Docker daemon won't start after config changes
Solution:

```bash
sudo journalctl -u docker.service -n 50 --no-pager
sudo rm /etc/docker/daemon.json && systemctl restart docker
```

# FAQs

### Q: How often should I run cleanup?
**A:** For active deployment servers, run cleanup **daily**. For development machines, run it **weekly**.

### Q: Will this affect running containers?
**A:** No, prune commands only remove **unused resources**. Running containers will not be affected.





      
