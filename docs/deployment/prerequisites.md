# Prerequisites

This document outlines the software, credentials, and network requirements needed to deploy the Ollama service on Open Horizon edge nodes.

## Software Requirements

### Open Horizon Agent

The Open Horizon agent must be installed and configured on your edge node.

**Supported Versions:**
- Open Horizon Agent 2.30.0 or later

**Installation:**
- Follow the [Open Horizon Agent Installation Guide](https://open-horizon.github.io/quick-start/)
- Verify installation: `hzn version`

### Container Runtime

A container runtime is required to run the Ollama service container.

**Supported Runtimes:**
- Docker 20.10 or later
- Podman 3.0 or later

**Installation:**
- Docker: [Install Docker Engine](https://docs.docker.com/engine/install/)
- Podman: [Install Podman](https://podman.io/getting-started/installation)

**Verify Installation:**
```bash
docker --version
# or
podman --version
```

### System Requirements

**Minimum Hardware:**
- CPU: 2 cores
- RAM: 2 GB minimum, 4 GB recommended
- Storage: 10 GB minimum, 20 GB recommended for models
- Architecture: x86_64 (amd64) or ARM64

**Operating System:**
- Linux (Ubuntu 20.04+, RHEL 8+, Debian 11+)
- macOS 11+ (for development/testing)

## Credentials and Access

### Open Horizon Exchange Credentials

You need valid credentials to access the Open Horizon Exchange.

**Required Information:**
- Organization ID (`HZN_ORG_ID`)
- Exchange URL (`HZN_EXCHANGE_URL`)
- User credentials (`HZN_EXCHANGE_USER_AUTH` in format `user:password` or `user:api-key`)

**Obtain Credentials:**
- Contact your Open Horizon administrator
- Or register at your organization's Exchange instance

**Set Environment Variables:**
```bash
export HZN_ORG_ID="your-org-id"
export HZN_EXCHANGE_URL="https://exchange.example.com/v1"
export HZN_EXCHANGE_USER_AUTH="your-user:your-password"
```

### Service Signing Keys (Optional)

If you plan to publish services, you need RSA key pairs for signing.

**Generate Keys:**
```bash
hzn key create "your-company" "your-email@example.com"
```

This creates:
- `~/.hzn/keys/service.private.key`
- `~/.hzn/keys/service.public.pem`

## Network Requirements

### Outbound Connectivity

The edge node must have outbound network access to:

**Open Horizon Services:**
- Exchange API (typically port 443/HTTPS)
- Agbot API (typically port 443/HTTPS)
- CSS (Cloud Sync Service) if used (typically port 443/HTTPS)

**Container Registry:**
- Docker Hub (docker.io) for pulling Ollama images
- Port 443/HTTPS

**Bandwidth Considerations:**
- Initial Ollama container image: ~1-2 GB
- Model downloads: varies by model (500 MB to 10+ GB)
- Recommend stable connection with at least 10 Mbps for initial setup

### Inbound Connectivity (Optional)

If you want to access the Ollama API from external clients:
- Port 11434/TCP (Ollama API default port)

**Firewall Configuration:**
```bash
# Example for UFW (Ubuntu)
sudo ufw allow 11434/tcp

# Example for firewalld (RHEL/CentOS)
sudo firewall-cmd --permanent --add-port=11434/tcp
sudo firewall-cmd --reload
```

## Storage Configuration

### Persistent Storage for Models

Ollama stores models in `/root/.ollama` by default. For production deployments, configure persistent storage.

**Recommended Approach:**
- Create a dedicated volume or directory
- Mount it to the container at `/root/.ollama`

**Example Setup:**
```bash
# Create directory for model storage
sudo mkdir -p /var/ollama
sudo chmod 755 /var/ollama

# Or create a Docker volume
docker volume create ollama-storage
```

The service definition includes this configuration by default.

## Verification Checklist

Before proceeding with deployment, verify:

- [ ] Open Horizon agent installed and running (`hzn version`)
- [ ] Container runtime installed (`docker --version` or `podman --version`)
- [ ] System meets minimum hardware requirements
- [ ] Exchange credentials configured and tested (`hzn exchange status`)
- [ ] Network connectivity to Exchange and Docker Hub
- [ ] Persistent storage configured (if required)
- [ ] Firewall rules configured (if external access needed)

## Next Steps

Once all prerequisites are met, proceed to the [Quick Start Guide](quickstart.md) for deployment instructions.

## Troubleshooting

### Agent Not Connecting to Exchange

**Symptom:** `hzn exchange status` fails or times out

**Solutions:**
1. Verify Exchange URL is correct
2. Check network connectivity: `curl -sS $HZN_EXCHANGE_URL/status`
3. Verify credentials are correct
4. Check firewall rules allow outbound HTTPS

### Container Runtime Issues

**Symptom:** Docker/Podman commands fail

**Solutions:**
1. Verify service is running: `sudo systemctl status docker`
2. Check user permissions: Add user to docker group
3. Restart service: `sudo systemctl restart docker`

### Insufficient Storage

**Symptom:** Container fails to start or model downloads fail

**Solutions:**
1. Check available space: `df -h`
2. Clean up unused containers: `docker system prune`
3. Move model storage to larger volume
