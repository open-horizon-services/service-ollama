# service-ollama

![Architecture](https://img.shields.io/badge/architecture-x86__64%20%7C%20ARM64-green)
![License](https://img.shields.io/github/license/open-horizon-services/service-ollama)
![Contributors](https://img.shields.io/github/contributors/open-horizon-services/service-ollama)

Open Horizon service configuration for deploying Ollama AI inference server on edge nodes.

## Overview

This repository provides a complete Open Horizon service definition for deploying [Ollama](https://ollama.ai) as a managed edge service. Ollama enables running large language models locally on edge devices, providing AI inference capabilities without cloud dependencies.

**Key Features:**
- 🚀 Easy deployment via Open Horizon policies
- 🏗️ Multi-architecture support (x86_64 and ARM64)
- 📦 Official Ollama container images
- 🔄 Automated service lifecycle management
- 📚 Comprehensive documentation for Day 1 and Day 2 operations
- 🛠️ Model management without service redeployment

## Architecture Support

This service supports the following architectures:

- **x86_64 (amd64)**: Intel/AMD 64-bit processors
- **ARM64 (aarch64)**: ARM 64-bit processors (Raspberry Pi 4+, NVIDIA Jetson, etc.)

## Quick Start

### Prerequisites

Before deploying, ensure you have:

- Open Horizon Management Hub access (or [apply for community hub access](https://wiki.lfedge.org/display/LE/Open+Horizon+Management+Hub+Developer+Instance))
- Edge node with Open Horizon agent installed
- Docker or Podman container runtime
- Minimum 2GB RAM, 10GB storage
- Network access to Open Horizon Exchange and Docker Hub

**Detailed prerequisites:** See [docs/deployment/prerequisites.md](docs/deployment/prerequisites.md)

### Installation

1. **Clone the repository:**
   ```bash
   git clone https://github.com/open-horizon-services/service-ollama.git
   cd service-ollama
   ```

2. **Configure environment:**
   ```bash
   export HZN_ORG_ID="your-org-id"
   export HZN_EXCHANGE_URL="https://exchange.example.com/v1"
   export HZN_EXCHANGE_USER_AUTH="your-user:your-password"
   ```

3. **Publish service:**
   ```bash
   make publish-service
   make publish-service-policy
   make publish-deployment-policy
   ```

4. **Register edge node:**
   ```bash
   make agent-run
   ```

5. **Verify deployment:**
   ```bash
   curl http://localhost:11434/api/version
   ```

**Complete deployment guide:** See [docs/deployment/quickstart.md](docs/deployment/quickstart.md)

## Documentation

### Day 1 - Deployment

- [Prerequisites](docs/deployment/prerequisites.md) - Software, credentials, and network requirements
- [Quick Start Guide](docs/deployment/quickstart.md) - Step-by-step deployment instructions
- [Troubleshooting](docs/deployment/troubleshooting.md) - Common deployment issues and solutions

### Day 2 - Operations

- [Model Management](docs/operations/model-management.md) - Adding, removing, and updating models
- [Service Updates](docs/operations/service-updates.md) - Updating Ollama versions and configuration
- [Monitoring](docs/operations/monitoring.md) - Health checks and diagnostics
- [Storage Management](docs/operations/storage-management.md) - Persistent storage and backups
- [API Access](docs/operations/api-access.md) - Using the Ollama API

## Repository Structure

```
service-ollama/
├── horizon/                    # Open Horizon configuration files
│   ├── service.definition.json       # x86_64 service definition
│   ├── service.definition.arm64.json # ARM64 service definition
│   ├── service.policy.json           # Service-side policy
│   ├── deployment.policy.json        # Deployment policy
│   └── node.policy.json              # Example node policy
├── docs/                       # Documentation
│   ├── deployment/                   # Day 1 deployment guides
│   └── operations/                   # Day 2 operations guides
├── Makefile                    # Build and deployment automation
└── README.md                   # This file
```

## Usage

### Local Testing

Test Ollama locally before deploying:

```bash
# Start Ollama container
make run

# Test API
make test

# Stop container
make stop
```

### Service Management

```bash
# Publish service to Exchange
make publish-service

# Validate service definition
make validate-service

# List published services
make list-services

# Remove service from Exchange
make remove-service
```

### Node Management

```bash
# Register node with policy
make agent-run

# Check node status
hzn node list

# View agreements
hzn agreement list

# Unregister node
make agent-stop
```

### Model Operations

```bash
# Access container
docker exec -it $(docker ps -qf "ancestor=ollama/ollama:latest") /bin/bash

# Pull a model
ollama pull llama2

# List models
ollama list

# Remove a model
ollama rm llama2
```

## API Examples

### Generate Text

```bash
curl -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt": "Why is the sky blue?",
  "stream": false
}'
```

### Chat

```bash
curl -X POST http://localhost:11434/api/chat -d '{
  "model": "llama2",
  "messages": [
    {"role": "user", "content": "Hello!"}
  ],
  "stream": false
}'
```

### List Models

```bash
curl http://localhost:11434/api/tags
```

**More examples:** See [docs/operations/api-access.md](docs/operations/api-access.md)

## Makefile Targets

### Service Operations
- `validate-service` - Validate service definition files
- `publish-service` - Publish service to Exchange
- `sign-service` - Sign and publish service
- `list-services` - List published services
- `remove-service` - Remove service from Exchange

### Policy Operations
- `publish-service-policy` - Publish service policy
- `publish-deployment-policy` - Publish deployment policy
- `remove-service-policy` - Remove service policy
- `remove-deployment-policy` - Remove deployment policy

### Node Operations
- `agent-run` - Register node with policy
- `agent-stop` - Unregister node

### Local Testing
- `init` - Create Docker volume
- `run` - Run container locally
- `stop` - Stop local container
- `test` - Test API endpoint
- `attach` - Connect to container shell
- `clean` - Remove container and volume

### Diagnostics
- `check` - View environment variables
- `deploy-check` - Verify policy compatibility
- `log` - View service and event logs

## Environment Variables

### Required
- `HZN_ORG_ID` - Your Open Horizon organization ID
- `HZN_EXCHANGE_URL` - Exchange API endpoint
- `HZN_EXCHANGE_USER_AUTH` - Exchange credentials (user:password)

### Optional
- `ARCH` - Target architecture (amd64 or arm64, default: amd64)
- `SERVICE_VERSION` - Service version (default: 0.1.0)
- `DOCKER_IMAGE_BASE` - Docker image name (default: ollama/ollama)
- `DOCKER_IMAGE_VERSION` - Docker image tag (default: latest)

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

For major changes, please open an issue first to discuss proposed changes.

## Support

- **Issues:** [GitHub Issues](https://github.com/open-horizon-services/service-ollama/issues)
- **Documentation:** [docs/](docs/)
- **Open Horizon:** [open-horizon.github.io](https://open-horizon.github.io/)
- **Ollama:** [ollama.ai](https://ollama.ai)

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE.md) file for details.

## Acknowledgments

- [Open Horizon](https://www.lfedge.org/projects/openhorizon/) - Edge computing platform
- [Ollama](https://ollama.ai) - Local LLM inference server
- [LF Edge](https://www.lfedge.org/) - Linux Foundation Edge

## Additional Resources

- [Open Horizon Documentation](https://open-horizon.github.io/)
- [Open Horizon Examples](https://github.com/open-horizon/examples)
- [Ollama Documentation](https://github.com/ollama/ollama)
- [Ollama Model Library](https://ollama.ai/library)