# Docker Registry v3 Installation on QNAP Container Station

**Languages:** [English](README.md) | [Italiano](README.it.md)

**Version:** 1.2
**Date:** November 29, 2025
**Container Station:** 3.1.1.1451+
**Registry:** 3.0.0

---

## Overview

This guide describes how to install and configure a private Docker Registry on QNAP NAS using Container Station. The procedure uses the Docker CLI to ensure correct port mapping, bypassing the known limitations of the Container Station graphical interface.

---

## Prerequisites

- QNAP NAS with Container Station 3.x installed
- SSH access to the NAS enabled
- SSH client (Terminal on macOS/Linux, PuTTY on Windows)

---

## Registry Storage Architecture

### Directory Structure

The registry organizes data in a content-addressable way:

```
/share/Container/registry-data/
└── docker/
    └── registry/
        └── v2/
            ├── blobs/
            │   └── sha256/
            │       ├── 00/
            │       │   └── 00a123.../
            │       │       └── data        ← Binary layer
            │       ├── 01/
            │       └── ff/
            └── repositories/
                └── <image-name>/
                    ├── _layers/
                    │   └── sha256/
                    │       └── <digest>/
                    │           └── link     ← Pointer to blob
                    ├── _manifests/
                    │   ├── revisions/
                    │   └── tags/
                    │       └── <tag>/
                    │           └── current/
                    │               └── link ← Current tag
                    └── _uploads/            ← Temporary uploads
```

### Components

| Directory | Content | Feature |
|-----------|---------|---------|
| `blobs/sha256/` | Binary layers | Content-addressable, deduplicated |
| `repositories/` | Metadata | Links to blobs |
| `_manifests/tags/` | Tag → digest mapping | Tag resolution |
| `_uploads/` | Uploads in progress | Temporary |

### Benefits for QNAP Snapshots

1. **Natural deduplication**: Shared layers between images exist only once
2. **Efficient incremental snapshots**: Only new blobs are captured
3. **Guaranteed consistency**: Blobs are immutable (write-once)

### Container / Data Separation

| Inside container (ephemeral) | Outside container (persistent) |
|------------------------------|--------------------------------|
| Binary `/bin/registry` | Image blobs |
| Default config | Repository metadata |
| In-memory cache | Tags and manifests |

**Container update**: Data in the mounted volume remains intact. The new container reads the same data.

---

## Installation Procedure

### 1. SSH Connection to NAS

```bash
ssh admin@<NAS_IP>
```

### 2. Create Data Directory

```bash
mkdir -p /share/Container/registry-data
```

### 3. Remove Existing Container (if present)

```bash
docker stop registry
docker rm registry
```

### 4. Create Registry v3 Container

#### Basic Configuration

Minimal configuration for testing and development:

```bash
docker run -d \
  --name registry \
  --restart=always \
  -p 5000:5000 \
  -e OTEL_TRACES_EXPORTER=none \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 \
  -v /share/Container/registry-data:/var/lib/registry \
  registry:3
```

#### Intermediate Configuration (Recommended)

With persistent secret key, resource limits, and log management:

```bash
# 1. Generate persistent secret key
openssl rand -hex 32 > /share/Container/registry-secret.txt

# 2. Start container
docker run -d \
  --name registry \
  --restart=always \
  -p 5000:5000 \
  --memory=1g \
  --memory-swap=1g \
  --cpus=2 \
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  -e OTEL_TRACES_EXPORTER=none \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 \
  -e REGISTRY_HTTP_SECRET=$(cat /share/Container/registry-secret.txt) \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  -v /share/Container/registry-data:/var/lib/registry \
  registry:3
```

#### Production Configuration (with Redis)

For high-traffic environments with many concurrent pulls:

```bash
# 1. Generate persistent secret key
openssl rand -hex 32 > /share/Container/registry-secret.txt

# 2. Start Redis for caching
docker run -d \
  --name registry-redis \
  --restart=always \
  --memory=256m \
  redis:alpine

# 3. Start Registry with Redis cache
docker run -d \
  --name registry \
  --restart=always \
  --link registry-redis:redis \
  -p 5000:5000 \
  --memory=1g \
  --memory-swap=1g \
  --cpus=2 \
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  -e OTEL_TRACES_EXPORTER=none \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 \
  -e REGISTRY_HTTP_SECRET=$(cat /share/Container/registry-secret.txt) \
  -e REGISTRY_STORAGE_DELETE_ENABLED=true \
  -e REGISTRY_STORAGE_CACHE_BLOBDESCRIPTOR=redis \
  -e REGISTRY_REDIS_ADDR=redis:6379 \
  -v /share/Container/registry-data:/var/lib/registry \
  registry:3
```

### Configuration Comparison

| Parameter | Basic | Intermediate | Production |
|-----------|-------|--------------|------------|
| Resource limits | No | Yes | Yes |
| Persistent key | No | Yes | Yes |
| Log management | No | Yes | Yes |
| Image deletion | No | Yes | Yes |
| Redis cache | No | No | Yes |
| Recommended use | Testing | Normal use | High traffic |

### 5. Installation Verification

```bash
# Running container
docker ps | grep registry

# Port mapping
docker port registry
# Output: 5000/tcp -> 0.0.0.0:5000

# Clean logs
docker logs registry
# Output: level=info msg="listening on 0.0.0.0:5000"
```

### 6. Functionality Test

```bash
# Local test
curl http://localhost:5000/v2/

# Remote test
curl http://<NAS_IP>:5000/v2/

# Expected response: {}
```

---

## Docker Client Configuration

### Linux

Edit `/etc/docker/daemon.json`:

```json
{
  "insecure-registries": ["<NAS_IP>:5000"]
}
```

Restart Docker:

```bash
sudo systemctl restart docker
```

### macOS / Windows (Docker Desktop)

Docker Desktop → Settings → Docker Engine → add:

```json
{
  "insecure-registries": ["<NAS_IP>:5000"]
}
```

Apply & Restart.

---

## Registry Usage

### Push image

```bash
docker tag alpine:latest <NAS_IP>:5000/alpine:latest
docker push <NAS_IP>:5000/alpine:latest
```

### Pull image

```bash
docker pull <NAS_IP>:5000/alpine:latest
```

### List images

```bash
curl http://<NAS_IP>:5000/v2/_catalog
```

### List tags

```bash
curl http://<NAS_IP>:5000/v2/<image-name>/tags/list
```

---

## Backup with QNAP Snapshots

### Recommended Strategy

1. **Scheduled snapshots** on `/share/Container/` every 4-6 hours
2. **Snapshot replication** to a second NAS or cloud
3. **Retention**: keep N snapshots for point-in-time recovery

### Snapshot Consistency

Blobs are immutable, so snapshots are always consistent. For critical snapshots on high-traffic registries:

```bash
# Temporary pause (optional)
docker pause registry
# → Execute QNAP snapshot
docker unpause registry
```

### What to Include in Backup

| Path | Content | Priority |
|------|---------|----------|
| `/share/Container/registry-data/` | Images and metadata | **Critical** |
| `/share/Container/registry-secret.txt` | Secret key | **Critical** |

### Restore

```bash
# 1. Restore QNAP snapshot
# 2. Recreate container with same command (section 4)
# 3. Verify: curl http://localhost:5000/v2/_catalog
```

---

## QNAP Optimizations

### Storage

- Use **SSD/NVMe** for `/share/Container/`
- Enable **SSD Cache** in Storage & Snapshots
- Move **Docker Root to SSD** in Container Station → Preferences

### Network

- **Jumbo Frames 9000** if supported by the network
- **Host Network Mode** (`--network=host`) to eliminate NAT overhead

### System

- Disable **unnecessary QTS services**
- Exclude `/share/Container/` from **antivirus**
- Expand **RAM to 8GB+** for intensive use

### Verify Storage Driver

```bash
docker info | grep "Storage Driver"
# Optimal: overlay2
```

---

## Maintenance

### Garbage Collection

Reclaim space by removing orphaned layers:

```bash
# Dry-run (verification)
docker exec registry bin/registry garbage-collect \
  --dry-run /etc/distribution/config.yml

# Execution
docker exec registry bin/registry garbage-collect \
  /etc/distribution/config.yml
```

### Image Deletion

```bash
# Get digest
DIGEST=$(curl -s -I -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  http://<NAS_IP>:5000/v2/<image>/manifests/<tag> | \
  grep -i docker-content-digest | awk '{print $2}' | tr -d '\r')

# Delete
curl -X DELETE http://<NAS_IP>:5000/v2/<image>/manifests/$DIGEST

# Garbage collection
docker exec registry bin/registry garbage-collect /etc/distribution/config.yml
```

### Registry Update

```bash
docker pull registry:3
docker stop registry
docker rm registry
# Execute the docker run command again for your chosen configuration
```

---

## Troubleshooting

### Port unreachable

1. Check firewall: Control Panel → Security
2. Add `10.0.0.0/8` to allowed networks

### HTTPS error

Configure `insecure-registries` on the client (see Client Configuration section).

### Network verification

```bash
netstat -tlnp | grep 5000
# Output: tcp 0 0 0.0.0.0:5000 ... docker-proxy
```

### Resource monitoring

```bash
docker stats registry
```

---

## References

- [CNCF Distribution Documentation](https://distribution.github.io/distribution/)
- [Docker Registry API v2](https://docs.docker.com/registry/spec/api/)
- [QNAP Container Station](https://www.qnap.com/en/software/container-station)
