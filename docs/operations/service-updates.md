# Service Updates

This guide covers procedures for updating the Ollama service itself, including version upgrades, configuration changes, and rollback procedures.

## Overview

Service updates involve changing the Ollama container version or modifying service configuration. Unlike model management, service updates require redeployment through Open Horizon.

## Update Types

1. **Version Updates**: Upgrading to a newer Ollama release
2. **Configuration Changes**: Modifying service parameters
3. **Policy Updates**: Changing deployment or service policies
4. **Rollback**: Reverting to a previous working version

## Prerequisites

- Access to Open Horizon Exchange
- Service publishing credentials
- Edge node access for verification

## Ollama Version Updates

### Check Current Version

```bash
# From host machine
curl -sS http://localhost:11434/api/version

# Or inside container
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama --version
```

**Example Output:**
```json
{
  "version": "0.1.17"
}
```

### Update Procedure

#### Step 1: Check for New Versions

Visit [Ollama Releases](https://github.com/ollama/ollama/releases) or check Docker Hub:

```bash
# Check available tags
curl -sS https://hub.docker.com/v2/repositories/ollama/ollama/tags | jq '.results[].name' | head -10
```

#### Step 2: Update Service Definition

Update the image version in your service definition:

```bash
# Edit service definition
vi horizon/service.definition.json

# Update version in deployment section
# Change: "image": "docker.io/ollama/ollama:latest"
# To: "image": "docker.io/ollama/ollama:0.1.20"

# Also update service version
# Change: "version": "0.1.0"
# To: "version": "0.1.1"
```

**Example Change:**
```json
{
  "version": "0.1.1",
  "deployment": {
    "services": {
      "ollama": {
        "image": "docker.io/ollama/ollama:0.1.20"
      }
    }
  }
}
```

#### Step 3: Update ARM64 Definition

Don't forget to update the ARM64 service definition:

```bash
vi horizon/service.definition.arm64.json
# Make the same changes as above
```

#### Step 4: Validate Changes

```bash
# Validate service definition
make validate-service

# Check for syntax errors
cat horizon/service.definition.json | jq .
```

#### Step 5: Publish Updated Service

```bash
# Set new version
export SERVICE_VERSION="0.1.1"

# Publish updated service
make publish-service

# Publish for ARM64 if needed
export ARCH="arm64"
make publish-service
export ARCH="amd64"
```

**Expected Output:**
```
==================
PUBLISHING SERVICE
==================
Publishing service definition...
Service published: your-org-id/service-ollama_0.1.1_amd64
```

#### Step 6: Update Deployment Policy

Update the deployment policy to reference the new version:

```bash
# Edit deployment policy
vi horizon/deployment.policy.json

# Update serviceVersions
# Add new version or update existing
```

**Example:**
```json
{
  "service": {
    "serviceVersions": [
      {
        "version": "0.1.1",
        "priority": {},
        "upgradePolicy": {}
      }
    ]
  }
}
```

```bash
# Publish updated deployment policy
make publish-deployment-policy
```

#### Step 7: Trigger Update on Edge Nodes

The update will automatically deploy to registered nodes based on the deployment policy. Monitor the rollout:

```bash
# On edge node, watch for agreement updates
watch hzn agreement list

# Check service log for new version
hzn service log -f service-ollama
```

**Automatic Update Process:**
1. Agbot detects new service version
2. New agreement is proposed
3. Old agreement is cancelled
4. New container is deployed
5. Old container is removed

#### Step 8: Verify Update

```bash
# Check new version
curl -sS http://localhost:11434/api/version

# Verify service is running
docker ps | grep ollama

# Test functionality
curl -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt": "Test after update",
  "stream": false
}'
```

### Version Pinning Strategy

**Production Best Practice:** Pin to specific versions rather than using `latest`:

```json
{
  "deployment": {
    "services": {
      "ollama": {
        "image": "docker.io/ollama/ollama:0.1.20"
      }
    }
  }
}
```

**Benefits:**
- Predictable deployments
- Easier rollback
- Better testing control
- Avoid unexpected breaking changes

**Version Pinning Workflow:**
1. Test new version in staging environment
2. Pin to tested version in production
3. Document version in release notes
4. Plan upgrade windows for version changes

## Configuration Changes

### Modifying Service Parameters

#### Update Environment Variables

```bash
# Edit service definition
vi horizon/service.definition.json

# Modify environment variables
```

**Example: Change Keep-Alive Duration**
```json
{
  "deployment": {
    "services": {
      "ollama": {
        "environment": [
          "OLLAMA_HOST=$OLLAMA_HOST",
          "OLLAMA_MODELS=$OLLAMA_MODELS",
          "OLLAMA_KEEP_ALIVE=10m"
        ]
      }
    }
  }
}
```

#### Update Port Mappings

```json
{
  "deployment": {
    "services": {
      "ollama": {
        "ports": [
          {
            "HostPort": "11435:11434/tcp",
            "HostIP": "0.0.0.0"
          }
        ]
      }
    }
  }
}
```

#### Update Volume Mounts

```json
{
  "deployment": {
    "services": {
      "ollama": {
        "binds": [
          "/mnt/models:/root/.ollama:rw"
        ]
      }
    }
  }
}
```

#### Apply Configuration Changes

```bash
# Increment service version
export SERVICE_VERSION="0.1.2"

# Update version in service definition
sed -i 's/"version": "0.1.1"/"version": "0.1.2"/' horizon/service.definition.json

# Validate and publish
make validate-service
make publish-service
make publish-deployment-policy
```

### Updating User Input Variables

Modify default values for user-configurable parameters:

```json
{
  "userInput": [
    {
      "name": "OLLAMA_KEEP_ALIVE",
      "label": "Duration to keep models loaded in memory",
      "type": "string",
      "defaultValue": "10m"
    }
  ]
}
```

## Service Redeployment Workflow

### Planned Redeployment

For scheduled updates with minimal disruption:

#### Step 1: Prepare Update

```bash
# Update service definition
# Update version number
# Validate changes
make validate-service
```

#### Step 2: Publish During Maintenance Window

```bash
# Publish new version
make publish-service
make publish-deployment-policy
```

#### Step 3: Monitor Rollout

```bash
# On edge nodes
watch hzn agreement list

# Check for new agreements
# Verify old agreements are cancelled
# Confirm new service is running
```

#### Step 4: Verify Deployment

```bash
# Test service functionality
curl http://localhost:11434/api/version

# Check models are still available
docker exec $(docker ps -qf "ancestor=ollama/ollama") ollama list

# Test inference
curl -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt": "Test",
  "stream": false
}'
```

### Emergency Redeployment

For critical updates or fixes:

```bash
# Quick update process
export SERVICE_VERSION="0.1.3"

# Update service definition with fix
vi horizon/service.definition.json

# Fast-track validation and publishing
make validate-service && make publish-service && make publish-deployment-policy

# Force immediate update on critical nodes
# (Unregister and re-register to force new agreement)
hzn unregister -f
hzn register --policy=horizon/node.policy.json
```

## Rollback Procedures

### When to Rollback

- New version causes service failures
- Performance degradation detected
- Compatibility issues with models
- Unexpected behavior or bugs

### Rollback Process

#### Step 1: Identify Previous Working Version

```bash
# List published services
make list-services

# Identify last known good version
# Example: service-ollama_0.1.0_amd64
```

#### Step 2: Update Deployment Policy

```bash
# Edit deployment policy
vi horizon/deployment.policy.json

# Change to previous version
```

**Example:**
```json
{
  "service": {
    "serviceVersions": [
      {
        "version": "0.1.0",
        "priority": {},
        "upgradePolicy": {}
      }
    ]
  }
}
```

#### Step 3: Publish Rollback Policy

```bash
# Publish updated deployment policy
make publish-deployment-policy
```

#### Step 4: Trigger Rollback on Nodes

The rollback will automatically deploy to nodes:

```bash
# On edge node, monitor rollback
watch hzn agreement list

# Verify old version is deployed
curl http://localhost:11434/api/version
```

#### Step 5: Verify Rollback

```bash
# Check service is running
docker ps | grep ollama

# Verify functionality
curl -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt": "Test after rollback",
  "stream": false
}'

# Check models are intact
docker exec $(docker ps -qf "ancestor=ollama/ollama") ollama list
```

### Fast Rollback (Manual)

For immediate rollback on specific nodes:

```bash
# On edge node
hzn unregister -f

# Update node policy to pin to old version
vi horizon/node.policy.json

# Change service version preference
```

**Example:**
```json
{
  "service": [
    {
      "name": "service-ollama",
      "org": "$HZN_ORG_ID",
      "serviceVersions": [
        {
          "version": "0.1.0"
        }
      ]
    }
  ]
}
```

```bash
# Re-register with updated policy
hzn register --policy=horizon/node.policy.json
```

## Update Best Practices

### Pre-Update Checklist

- [ ] Review release notes for breaking changes
- [ ] Test update in staging environment
- [ ] Backup current configuration
- [ ] Document current version
- [ ] Plan rollback procedure
- [ ] Schedule maintenance window
- [ ] Notify stakeholders

### During Update

- [ ] Monitor agreement formation
- [ ] Watch service logs
- [ ] Verify container deployment
- [ ] Test basic functionality
- [ ] Check model availability
- [ ] Monitor resource usage

### Post-Update

- [ ] Verify all nodes updated
- [ ] Test full functionality
- [ ] Monitor for issues (24-48 hours)
- [ ] Document any issues encountered
- [ ] Update runbooks if needed
- [ ] Communicate completion

### Testing Strategy

1. **Staging Environment:**
   ```bash
   # Deploy to test node first
   export HZN_ORG_ID="staging-org"
   make publish-service
   ```

2. **Canary Deployment:**
   - Update single production node
   - Monitor for 24 hours
   - Gradually roll out to more nodes

3. **Blue-Green Deployment:**
   - Maintain two service versions
   - Switch traffic between versions
   - Quick rollback if needed

## Troubleshooting Updates

### Update Not Deploying

**Symptom:** New version not appearing on nodes

**Solutions:**
```bash
# Check deployment policy is published
hzn exchange deployment listpolicy

# Verify service version exists
make list-services

# Check node policy compatibility
make deploy-check

# Force update
hzn unregister -f
hzn register --policy=horizon/node.policy.json
```

### Update Fails Mid-Deployment

**Symptom:** Service stops working during update

**Solutions:**
```bash
# Check event log
hzn eventlog list | tail -20

# Check container status
docker ps -a | grep ollama

# View container logs
docker logs $(docker ps -aqf "ancestor=ollama/ollama")

# Rollback immediately
# (Follow rollback procedure above)
```

### Models Lost After Update

**Symptom:** Models not available after service update

**Solutions:**
```bash
# Check volume mount
docker inspect $(docker ps -qf "ancestor=ollama/ollama") | grep Mounts -A 10

# Verify models directory
ls -la /var/ollama

# Re-pull models if necessary
docker exec $(docker ps -qf "ancestor=ollama/ollama") ollama pull llama2
```

## Monitoring Updates

### Track Update Progress

```bash
# Create monitoring script
cat > monitor-update.sh << 'EOF'
#!/bin/bash
while true; do
  clear
  echo "=== Service Status ==="
  hzn service list
  echo ""
  echo "=== Agreements ==="
  hzn agreement list
  echo ""
  echo "=== Container Status ==="
  docker ps | grep ollama
  echo ""
  echo "=== Version ==="
  curl -sS http://localhost:11434/api/version
  sleep 10
done
EOF

chmod +x monitor-update.sh
./monitor-update.sh
```

### Update Metrics

Track key metrics during updates:
- Agreement formation time
- Container startup time
- API response time
- Model availability
- Resource usage

## Next Steps

- Review [Model Management](model-management.md) for managing models after updates
- See [Monitoring](monitoring.md) for ongoing health checks
- Check [Storage Management](storage-management.md) for persistent data

## Additional Resources

- [Ollama Release Notes](https://github.com/ollama/ollama/releases)
- [Open Horizon Service Updates](https://open-horizon.github.io/docs/service_update/)
- [Docker Image Updates](https://docs.docker.com/engine/reference/commandline/pull/)
