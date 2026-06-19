## Context

This design establishes the initial Open Horizon service configuration for deploying Ollama as a managed edge service. The current state is a repository without service definitions or deployment documentation. The target state is a fully configured Open Horizon service with comprehensive operational documentation.

**Constraints:**
- Must follow Open Horizon service patterns and best practices
- Service must be containerized and deployable to edge nodes
- Configuration must support both x86_64 and ARM64 architectures
- Documentation must cover complete lifecycle from deployment to updates

**Stakeholders:**
- Edge administrators deploying Ollama services
- Developers integrating Ollama into edge applications
- Operations teams managing model updates and service maintenance

## Goals / Non-Goals

**Goals:**
- Create Open Horizon service definition files (service.definition.json, policies)
- Establish container build and publish workflow
- Document Day 1 deployment procedures (initial setup and provisioning)
- Document Day 2 operations (model management, service updates)
- Provide clear examples and runbooks for common operations

**Non-Goals:**
- Custom Ollama modifications or forks
- Multi-service orchestration patterns (focus on single Ollama service)
- Production-scale monitoring and alerting infrastructure (basic operational guidance only)
- Custom model training or fine-tuning procedures

## Decisions

### 1. Service Configuration Structure
**Decision:** Place all Open Horizon configuration files in a `horizon/` subdirectory following repository best practices.

**Rationale:** This follows the established pattern in Open Horizon service repositories, keeping service definitions separate from application code and making them easy to locate.

**Alternatives Considered:**
- Root-level configuration files: Rejected due to clutter and lack of organization
- `.horizon/` hidden directory: Rejected as it reduces discoverability

### 2. Container Base Image
**Decision:** Use official Ollama container images from Docker Hub as the base, with minimal customization for Open Horizon integration.

**Rationale:** Leverages official, maintained images and reduces maintenance burden. Open Horizon service wrapper adds only deployment metadata.

**Alternatives Considered:**
- Custom Ollama build: Rejected due to maintenance overhead and divergence from upstream
- Building from source: Rejected as unnecessary complexity for initial configuration

### 3. Model Management Strategy
**Decision:** Document model management as post-deployment operations using Ollama's native CLI and API, rather than baking models into the container image.

**Rationale:** Provides flexibility for users to manage models based on their needs, reduces container image size, and allows dynamic model updates without service redeployment.

**Alternatives Considered:**
- Pre-baked model images: Rejected due to large image sizes and inflexibility
- Separate model service: Rejected as over-engineering for initial implementation

### 4. Documentation Structure
**Decision:** Create separate documentation for Day 1 (deployment) and Day 2 (operations) procedures in a `docs/` directory.

**Rationale:** Clear separation of concerns makes it easier for users to find relevant information based on their current phase of work.

**Alternatives Considered:**
- Single comprehensive guide: Rejected as too lengthy and harder to navigate
- Wiki-based documentation: Rejected to keep everything in repository

### 5. Architecture Support
**Decision:** Support both x86_64 and ARM64 architectures in service definition with appropriate container image references.

**Rationale:** Edge deployments commonly use both architectures, and Ollama provides official images for both.

**Alternatives Considered:**
- x86_64 only: Rejected as it limits edge deployment options
- Additional architectures (ARM32): Deferred to future work based on demand

## Risks / Trade-offs

**Risk:** Container image size may be large (several GB) → **Mitigation:** Document image size expectations and network requirements; provide guidance on local caching strategies

**Risk:** Model downloads can be slow on edge networks → **Mitigation:** Document pre-deployment model caching options and provide examples of model management workflows

**Risk:** Ollama version updates may introduce breaking changes → **Mitigation:** Document version pinning strategy and testing procedures before updates; provide rollback instructions

**Trade-off:** Using official images means less control over Ollama configuration → **Accepted:** Simplicity and maintainability outweigh customization needs for initial implementation

**Trade-off:** Post-deployment model management requires manual steps → **Accepted:** Provides flexibility and avoids container bloat; can be automated in future iterations

## Migration Plan

**Initial Deployment:**
1. Create horizon/ directory with service configuration files
2. Create docs/ directory with deployment and operations guides
3. Add Makefile targets for service publishing and validation
4. Update README with quick start guide

**Validation:**
- Test service deployment on both x86_64 and ARM64 edge nodes
- Verify model download and management procedures
- Validate service update and rollback procedures

**Rollback Strategy:**
- Service can be unregistered from Open Horizon Exchange
- Edge nodes can be unconfigured to remove service
- No data migration concerns as models are managed separately

## Open Questions

- Should we provide example deployment policies for common edge scenarios (e.g., GPU-enabled nodes, resource-constrained devices)?
- What is the recommended approach for persistent model storage across service updates?
- Should we include example integration patterns with other edge services?
