# Quick Start Guide

This guide provides step-by-step instructions for deploying the Ollama service on Open Horizon edge nodes.

## Prerequisites

Before starting, ensure you have completed all items in the [Prerequisites](prerequisites.md) document.

## Overview

The deployment process consists of:
1. Publishing the service to the Exchange
2. Publishing service and deployment policies
3. Registering your edge node
4. Verifying the deployment

## Step 1: Configure Environment

Set required environment variables for your deployment:

```bash
# Your Open Horizon organization ID
export HZN_ORG_ID="your-org-id"

# Exchange API endpoint
export HZN_EXCHANGE_URL="https://exchange.example.com/v1"

# Your Exchange credentials (user:password or user:api-key)
export HZN_EXCHANGE_USER_AUTH="your-user:your-password"

# Target architecture (amd64 or arm64)
export ARCH="amd64"

# Service version
export SERVICE_VERSION="0.1.0"
```

**Verify Configuration:**
```bash
hzn exchange status
```

**Expected Output:**
```
Exchange URL: https://exchange.example.com/v1
Exchange version: 2.87.0
Exchange root user: root/root
Your user: your-org-id/your-user
```

## Step 2: Validate Service Definition

Before publishing, validate the service definition files:

```bash
cd /path/to/service-ollama
make validate-service
```

**Expected Output:**
```
=======================
VALIDATING SERVICE
=======================
Service definition validated successfully
```

**If Validation Fails:**
- Check JSON syntax in `horizon/service.definition.json`
- Verify all required fields are present
- Ensure environment variables are set correctly

## Step 3: Publish Service to Exchange

Publish the Ollama service definition to the Open Horizon Exchange:

```bash
make publish-service
```

**Expected Output:**
```
==================
PUBLISHING SERVICE
==================
Publishing service definition...
Service published: your-org-id/service-ollama_0.1.0_amd64
```

**Verify Publication:**
```bash
make list-services
```

You should see your service listed in the output.

## Step 4: Publish Service Policy

Publish the service policy that defines service-side constraints:

```bash
make publish-service-policy
```

**Expected Output:**
```
=========================
PUBLISHING SERVICE POLICY
=========================
Service policy published for: your-org-id/service-ollama_0.1.0_amd64
```

## Step 5: Publish Deployment Policy

Publish the deployment policy that defines where and how the service should be deployed:

```bash
make publish-deployment-policy
```

**Expected Output:**
```
============================
PUBLISHING DEPLOYMENT POLICY
============================
Deployment policy published: your-org-id/policy-service-ollama_0.1.0
```

## Step 6: Configure Node Policy

Before registering your node, review and customize the node policy if needed:

```bash
cat horizon/node.policy.json
```

**Key Configuration Items:**
- `openhorizon.memory`: Available memory in MB (default: 4096)
- `openhorizon.storage`: Available storage in MB (default: 20480)
- `openhorizon.arch`: Node architecture (amd64 or arm64)

**Edit if Necessary:**
```bash
# Update memory value
sed -i 's/"value": 4096/"value": 8192/' horizon/node.policy.json

# Update architecture for ARM64 nodes
sed -i 's/"value": "amd64"/"value": "arm64"/' horizon/node.policy.json
```

## Step 7: Register Edge Node

Register your edge node with the node policy:

```bash
make agent-run
```

**What Happens:**
1. Node registers with the Exchange
2. Node policy is applied
3. Agreement negotiation begins
4. Service container is deployed

**Expected Output:**
```
================
REGISTERING NODE
================
Registering node...
Node registered successfully
Waiting for agreement...
```

The command will watch for agreement formation. Press `Ctrl+C` when you see an agreement listed.

**Agreement Example:**
```json
[
  {
    "name": "Policy for your-org-id/service-ollama merged with your-org-id/policy-service-ollama_0.1.0",
    "current_agreement_id": "abc123...",
    "consumer_id": "IBM/agbot",
    "agreement_creation_time": "2024-01-15 10:30:00 -0500 EST",
    "agreement_accepted_time": "2024-01-15 10:30:15 -0500 EST",
    "agreement_finalized_time": "2024-01-15 10:30:30 -0500 EST",
    "agreement_execution_start_time": "2024-01-15 10:30:45 -0500 EST",
    "agreement_data_received_time": "",
    "agreement_protocol": "Basic",
    "workload_to_run": {
      "url": "service-ollama",
      "org": "your-org-id",
      "version": "0.1.0",
      "arch": "amd64"
    }
  }
]
```

## Step 8: Verify Service Deployment

### Check Node Status

```bash
hzn node list
```

**Expected Output:**
```json
{
  "id": "your-node-id",
  "organization": "your-org-id",
  "pattern": "",
  "name": "your-node-name",
  "nodeType": "device",
  "token_last_valid_time": "2024-01-15 10:30:00 -0500 EST",
  "token_valid": true,
  "ha": false,
  "configstate": {
    "state": "configured",
    "last_update_time": "2024-01-15 10:30:00 -0500 EST"
  }
}
```

### Check Running Agreements

```bash
hzn agreement list
```

You should see at least one active agreement for the Ollama service.

### Check Container Status

```bash
docker ps | grep ollama
# or for Podman
podman ps | grep ollama
```

**Expected Output:**
```
CONTAINER ID   IMAGE                    COMMAND       CREATED         STATUS         PORTS                      NAMES
abc123def456   ollama/ollama:latest     "/bin/ollama"  2 minutes ago   Up 2 minutes   0.0.0.0:11434->11434/tcp   ollama
```

### Test Ollama API

```bash
curl -sS http://localhost:11434/api/version
```

**Expected Output:**
```json
{
  "version": "0.1.17"
}
```

### Check Service Logs

```bash
hzn service log -f service-ollama
```

**Expected Output:**
```
time=2024-01-15T15:30:00Z level=info msg="Ollama server starting"
time=2024-01-15T15:30:01Z level=info msg="Listening on 0.0.0.0:11434"
```

## Step 9: Pull Your First Model

Once the service is running, pull a model to test functionality:

```bash
# Access the container
docker exec -it $(docker ps -qf "ancestor=ollama/ollama:latest") /bin/bash

# Inside the container, pull a model
ollama pull llama2

# Exit the container
exit
```

**Expected Output:**
```
pulling manifest
pulling 8934d96d3f08... 100% ▕████████████████▏ 3.8 GB
pulling 8c17c2ebb0ea... 100% ▕████████████████▏ 7.0 KB
pulling 7c23fb36d801... 100% ▕████████████████▏ 4.8 KB
pulling 2e0493f67d0c... 100% ▕████████████████▏   59 B
pulling fa304d675061... 100% ▕████████████████▏   91 B
pulling 42ba7f8a01dd... 100% ▕████████████████▏  557 B
verifying sha256 digest
writing manifest
success
```

### Test the Model

```bash
curl -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt": "Why is the sky blue?",
  "stream": false
}'
```

## Deployment Complete!

Your Ollama service is now running on your edge node. 

**Next Steps:**
- Review [Model Management](../operations/model-management.md) for managing models
- See [Monitoring](../operations/monitoring.md) for health checks and diagnostics
- Check [API Access](../operations/api-access.md) for integration examples

## Common Issues

### Service Not Starting

**Symptom:** No container running after registration

**Check:**
```bash
hzn eventlog list
```

**Common Causes:**
- Insufficient resources (memory/storage)
- Network connectivity issues
- Policy mismatch between node and deployment

**Solution:**
1. Verify node meets resource requirements
2. Check node policy matches deployment policy constraints
3. Review event log for specific errors

### Agreement Not Forming

**Symptom:** `hzn agreement list` shows no agreements

**Check:**
```bash
hzn deploycheck all -t device -B horizon/deployment.policy.json \
  --service=horizon/service.definition.json \
  --service-pol=horizon/service.policy.json \
  --node-pol=horizon/node.policy.json
```

**Common Causes:**
- Policy constraints don't match
- Service not published correctly
- Node properties don't satisfy deployment policy

**Solution:**
1. Run deploycheck to identify mismatches
2. Adjust node policy or deployment policy as needed
3. Re-register node if policy changes

### Container Fails to Pull

**Symptom:** Agreement formed but container not running

**Check:**
```bash
docker pull ollama/ollama:latest
```

**Common Causes:**
- No internet connectivity
- Docker Hub rate limiting
- Insufficient disk space

**Solution:**
1. Verify network connectivity to Docker Hub
2. Check available disk space: `df -h`
3. Wait and retry if rate limited

## Unregistering the Node

To remove the service and unregister the node:

```bash
make agent-stop
```

This will:
1. Cancel all agreements
2. Stop and remove service containers
3. Unregister the node from the Exchange

## Additional Resources

- [Open Horizon Documentation](https://open-horizon.github.io/)
- [Ollama Documentation](https://github.com/ollama/ollama)
- [Troubleshooting Guide](troubleshooting.md)
