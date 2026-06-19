# Storage Management

This guide covers procedures for managing persistent model storage across service updates and node restarts.

## Overview

Ollama stores models in persistent storage to ensure they remain available across container restarts and service updates. Proper storage management is critical for reliable operation.

## Storage Architecture

### Default Storage Configuration

The Ollama service uses the following storage configuration:

**Host Path:** `/var/ollama`
**Container Path:** `/root/.ollama`
**Mount Type:** Bind mount (read-write)

**Service Definition:**
```json
{
  "deployment": {
    "services": {
      "ollama": {
        "binds": [
          "/var/ollama:/root/.ollama:rw"
        ]
      }
    }
  }
}
```

### Storage Layout

```
/var/ollama/
├── models/
│   ├── manifests/
│   │   └── registry.ollama.ai/
│   │       └── library/
│   │           ├── llama2/
│   │           ├── mistral/
│   │           └── codellama/
│   └── blobs/
│       ├── sha256-abc123...
│       ├── sha256-def456...
│       └── sha256-ghi789...
└── tmp/
```

**Key Directories:**
- `models/manifests/`: Model metadata and version information
- `models/blobs/`: Actual model data (deduplicated)
- `tmp/`: Temporary files during model downloads

## Configuring Persistent Storage

### Verify Current Configuration

```bash
# Check mount point
docker inspect $(docker ps -qf "ancestor=ollama/ollama:latest") | jq '.[0].Mounts'

# Check storage usage
df -h /var/ollama

# Check directory contents
ls -lh /var/ollama
```

**Expected Output:**
```json
[
  {
    "Type": "bind",
    "Source": "/var/ollama",
    "Destination": "/root/.ollama",
    "Mode": "rw",
    "RW": true,
    "Propagation": "rprivate"
  }
]
```

### Alternative Storage Locations

If `/var/ollama` is not suitable, configure a different location:

#### Option 1: Use Different Host Directory

```bash
# Create new directory
sudo mkdir -p /mnt/models/ollama
sudo chmod 755 /mnt/models/ollama

# Update service definition
vi horizon/service.definition.json
```

**Change:**
```json
{
  "binds": [
    "/mnt/models/ollama:/root/.ollama:rw"
  ]
}
```

#### Option 2: Use Docker Volume

```bash
# Create named volume
docker volume create ollama-models

# Update service definition
vi horizon/service.definition.json
```

**Change:**
```json
{
  "binds": [
    "ollama-models:/root/.ollama:rw"
  ]
}
```

#### Option 3: Use External Storage

```bash
# Mount external drive
sudo mkdir -p /mnt/external
sudo mount /dev/sdb1 /mnt/external

# Create ollama directory
sudo mkdir -p /mnt/external/ollama
sudo chmod 755 /mnt/external/ollama

# Make mount persistent
echo "/dev/sdb1 /mnt/external ext4 defaults 0 2" | sudo tee -a /etc/fstab

# Update service definition
vi horizon/service.definition.json
```

**Change:**
```json
{
  "binds": [
    "/mnt/external/ollama:/root/.ollama:rw"
  ]
}
```

### Apply Storage Configuration Changes

After modifying storage configuration:

```bash
# Increment service version
export SERVICE_VERSION="0.1.1"

# Update version in service definition
sed -i 's/"version": "0.1.0"/"version": "0.1.1"/' horizon/service.definition.json

# Validate and publish
make validate-service
make publish-service
make publish-deployment-policy
```

## Model Backup Procedures

### Manual Backup

#### Backup All Models

```bash
# Create backup directory
sudo mkdir -p /backup/ollama

# Backup models directory
sudo tar -czf /backup/ollama/models-$(date +%Y%m%d-%H%M%S).tar.gz -C /var/ollama models

# Verify backup
ls -lh /backup/ollama/
```

#### Backup Specific Model

```bash
# Identify model location
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama list

# Backup specific model blobs
MODEL_NAME="llama2"
sudo tar -czf /backup/ollama/${MODEL_NAME}-$(date +%Y%m%d).tar.gz \
  -C /var/ollama/models/manifests/registry.ollama.ai/library/${MODEL_NAME} .
```

### Automated Backup Script

```bash
#!/bin/bash
# backup-models.sh

BACKUP_DIR="/backup/ollama"
SOURCE_DIR="/var/ollama"
RETENTION_DAYS=30

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Create backup
BACKUP_FILE="$BACKUP_DIR/models-$(date +%Y%m%d-%H%M%S).tar.gz"
echo "Creating backup: $BACKUP_FILE"
tar -czf "$BACKUP_FILE" -C "$SOURCE_DIR" models

# Verify backup
if [ $? -eq 0 ]; then
  echo "✓ Backup created successfully"
  ls -lh "$BACKUP_FILE"
else
  echo "✗ Backup failed"
  exit 1
fi

# Remove old backups
echo "Removing backups older than $RETENTION_DAYS days"
find "$BACKUP_DIR" -name "models-*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup complete"
```

**Schedule with Cron:**
```bash
# Edit crontab
crontab -e

# Add daily backup at 2 AM
0 2 * * * /usr/local/bin/backup-models.sh >> /var/log/ollama-backup.log 2>&1
```

### Remote Backup

```bash
# Backup to remote server via rsync
rsync -avz --progress /var/ollama/models/ backup-server:/backups/ollama/models/

# Backup to S3 (requires AWS CLI)
aws s3 sync /var/ollama/models/ s3://my-bucket/ollama-backups/models/

# Backup to network share
sudo mount -t cifs //nas-server/backups /mnt/nas -o credentials=/etc/nas-creds
sudo rsync -av /var/ollama/models/ /mnt/nas/ollama/models/
```

## Model Restoration

### Restore from Backup

#### Full Restore

```bash
# Stop service (if running)
docker stop $(docker ps -qf "ancestor=ollama/ollama:latest")

# Clear existing models (optional)
sudo rm -rf /var/ollama/models/*

# Restore from backup
sudo tar -xzf /backup/ollama/models-20240115-020000.tar.gz -C /var/ollama

# Verify restoration
ls -lh /var/ollama/models/

# Restart service
# (Service will restart automatically via Open Horizon)
```

#### Restore Specific Model

```bash
# Extract specific model
MODEL_NAME="llama2"
sudo tar -xzf /backup/ollama/${MODEL_NAME}-20240115.tar.gz \
  -C /var/ollama/models/manifests/registry.ollama.ai/library/${MODEL_NAME}

# Verify model
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama list
```

### Verify Models After Restoration

```bash
# List models
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama list

# Test each model
for model in $(docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama list | tail -n +2 | awk '{print $1}'); do
  echo "Testing $model..."
  docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama run "$model" "test" --verbose
done
```

## Model Persistence Across Service Restarts

### Verify Persistence

```bash
# List current models
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama list

# Restart container
docker restart $(docker ps -qf "ancestor=ollama/ollama:latest")

# Wait for restart
sleep 10

# Verify models still available
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama list
```

**Expected Behavior:**
- Models remain available after restart
- No re-download required
- Same model list before and after

### Troubleshooting Persistence Issues

**Problem: Models Lost After Restart**

```bash
# Check mount configuration
docker inspect $(docker ps -qf "ancestor=ollama/ollama:latest") | jq '.[0].Mounts'

# Verify host directory exists
ls -la /var/ollama/models/

# Check permissions
ls -ld /var/ollama
# Should show: drwxr-xr-x or similar

# Fix permissions if needed
sudo chmod 755 /var/ollama
sudo chown -R root:root /var/ollama
```

**Problem: Models Corrupted After Restart**

```bash
# Check for disk errors
dmesg | grep -i error

# Verify filesystem
sudo fsck /dev/sda1  # Adjust device as needed

# Restore from backup
sudo tar -xzf /backup/ollama/models-latest.tar.gz -C /var/ollama
```

## Storage Monitoring

### Check Storage Usage

```bash
# Overall disk usage
df -h /var/ollama

# Detailed breakdown
du -sh /var/ollama/*
du -sh /var/ollama/models/*

# Largest files
du -h /var/ollama/models/blobs/ | sort -h | tail -20
```

### Monitor Storage Growth

```bash
#!/bin/bash
# monitor-storage.sh

while true; do
  USAGE=$(df -h /var/ollama | tail -1 | awk '{print $5}' | sed 's/%//')
  USED=$(df -h /var/ollama | tail -1 | awk '{print $3}')
  AVAIL=$(df -h /var/ollama | tail -1 | awk '{print $4}')
  
  echo "$(date): Used: $USED | Available: $AVAIL | Usage: $USAGE%"
  
  if [ "$USAGE" -gt 80 ]; then
    echo "WARNING: Storage usage above 80%"
  fi
  
  sleep 3600  # Check every hour
done
```

### Storage Alerts

```bash
#!/bin/bash
# storage-alert.sh

THRESHOLD=85
USAGE=$(df -h /var/ollama | tail -1 | awk '{print $5}' | sed 's/%//')

if [ "$USAGE" -gt "$THRESHOLD" ]; then
  echo "ALERT: Ollama storage usage at ${USAGE}%"
  echo "Consider removing unused models or expanding storage"
  
  # List models by size
  echo "Models by size:"
  docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama list
  
  # Send notification (example)
  # mail -s "Ollama Storage Alert" admin@example.com < /tmp/alert.txt
fi
```

## Storage Optimization

### Remove Unused Models

```bash
# List models with sizes
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama list

# Remove unused models
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama rm <model-name>

# Check freed space
df -h /var/ollama
```

### Clean Temporary Files

```bash
# Remove temporary files
sudo rm -rf /var/ollama/tmp/*

# Verify cleanup
du -sh /var/ollama/tmp/
```

### Deduplicate Blobs

Ollama automatically deduplicates model data using content-addressable storage. No manual deduplication needed.

**Verify Deduplication:**
```bash
# Count blob files
ls /var/ollama/models/blobs/ | wc -l

# Check for shared blobs between models
# (Blobs with same SHA256 are shared)
```

## Storage Migration

### Migrate to New Location

```bash
# Stop service
docker stop $(docker ps -qf "ancestor=ollama/ollama:latest")

# Create new location
sudo mkdir -p /new/location/ollama
sudo chmod 755 /new/location/ollama

# Copy data
sudo rsync -av /var/ollama/ /new/location/ollama/

# Verify copy
diff -r /var/ollama/models/ /new/location/ollama/models/

# Update service definition
vi horizon/service.definition.json
# Change binds to: "/new/location/ollama:/root/.ollama:rw"

# Publish updated service
export SERVICE_VERSION="0.1.2"
make publish-service
make publish-deployment-policy

# Verify new location
docker inspect $(docker ps -qf "ancestor=ollama/ollama:latest") | jq '.[0].Mounts'
```

### Migrate to Larger Volume

```bash
# Add new disk
# Partition and format new disk
sudo fdisk /dev/sdb
sudo mkfs.ext4 /dev/sdb1

# Mount new disk
sudo mkdir -p /mnt/models
sudo mount /dev/sdb1 /mnt/models

# Copy data
sudo rsync -av /var/ollama/ /mnt/models/ollama/

# Update fstab
echo "/dev/sdb1 /mnt/models ext4 defaults 0 2" | sudo tee -a /etc/fstab

# Update service definition
# Change binds to: "/mnt/models/ollama:/root/.ollama:rw"

# Publish and verify
```

## Best Practices

1. **Regular Backups:**
   - Daily automated backups
   - Retain backups for 30 days
   - Test restoration periodically

2. **Storage Monitoring:**
   - Monitor usage daily
   - Alert at 80% capacity
   - Plan expansion before 90%

3. **Model Management:**
   - Remove unused models promptly
   - Document model inventory
   - Track model sizes

4. **Disaster Recovery:**
   - Maintain off-site backups
   - Document restoration procedures
   - Test recovery annually

5. **Performance:**
   - Use fast storage (SSD preferred)
   - Ensure adequate IOPS
   - Monitor disk latency

## Troubleshooting

### Storage Full

```bash
# Check usage
df -h /var/ollama

# Identify large files
du -h /var/ollama | sort -h | tail -20

# Remove unused models
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama rm <model>

# Clean temporary files
sudo rm -rf /var/ollama/tmp/*
```

### Slow Storage Performance

```bash
# Test write speed
dd if=/dev/zero of=/var/ollama/test bs=1M count=1000
rm /var/ollama/test

# Check disk I/O
iostat -x 1

# Check for errors
dmesg | grep -i error
```

### Permission Issues

```bash
# Fix ownership
sudo chown -R root:root /var/ollama

# Fix permissions
sudo chmod 755 /var/ollama
sudo chmod -R 644 /var/ollama/models/
sudo find /var/ollama/models/ -type d -exec chmod 755 {} \;
```

## Next Steps

- Review [API Access](api-access.md) for testing functionality
- See [Monitoring](monitoring.md) for ongoing health checks
- Check [Model Management](model-management.md) for managing models

## Additional Resources

- [Ollama Storage](https://github.com/ollama/ollama/blob/main/docs/faq.md#where-are-models-stored)
- [Docker Storage](https://docs.docker.com/storage/)
- [Linux Filesystem](https://www.kernel.org/doc/html/latest/filesystems/)
