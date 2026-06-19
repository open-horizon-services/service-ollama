## Why

The service-ollama project needs initial configuration and documentation to enable users to deploy and operate Ollama as a containerized service. Currently, there is no standardized way to publish the Ollama service container or comprehensive instructions for deployment and ongoing operations.

## What Changes

- Create initial Open Horizon service configuration files for publishing Ollama as a managed service
- Develop comprehensive deployment documentation covering Day 1 (initial deployment and provisioning) operations
- Document Day 2 operations including model management (adding, removing, updating models) and Ollama version updates
- Establish container configuration and publishing workflow
- Define service policies and deployment patterns for edge nodes

## Capabilities

### New Capabilities
- `ollama-deployment`: Initial deployment and provisioning of Ollama service on edge nodes, including container configuration, service definition, and deployment policies
- `ollama-operations`: Ongoing operational procedures for managing Ollama instances, including model lifecycle management and service updates

### Modified Capabilities
<!-- No existing capabilities are being modified -->

## Impact

- **New Files**: Open Horizon service definition files (service.definition.json, deployment policies, node policies)
- **Documentation**: New deployment guides and operational runbooks
- **Container Configuration**: Dockerfile and container build/publish scripts
- **Dependencies**: Ollama container image, Open Horizon agent requirements
- **User Impact**: Enables edge deployment of Ollama with standardized configuration and operational procedures
