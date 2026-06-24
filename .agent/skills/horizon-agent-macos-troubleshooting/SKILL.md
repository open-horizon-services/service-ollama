# Open Horizon Agent macOS Troubleshooting

## Overview
Troubleshooting guide for running the containerized Open Horizon agent on macOS, covering common issues and their solutions.

## Prerequisites
- macOS host with Podman installed
- Open Horizon containerized agent (horizon-container script)
- Access to Open Horizon Exchange with self-signed certificates

## Common Issues

### 1. TLS Certificate Verification Failures

**Symptoms:**
```
Error: tls: failed to verify certificate: x509: certificate signed by unknown authority
```

**Root Cause:**
The containerized agent doesn't trust the self-signed CA certificate used by the Open Horizon management hub.

**Solution:**
1. Ensure the CA certificate file exists on the host (e.g., `/tmp/agent-install.crt`)
2. Add `HZN_MGMT_HUB_CERT_PATH` to the horizon configuration file:
   ```bash
   sudo sh -c 'echo "HZN_MGMT_HUB_CERT_PATH=/tmp/agent-install.crt" >> /etc/default/horizon'
   ```
3. Restart the horizon container:
   ```bash
   /path/to/horizon-container stop
   /path/to/horizon-container start 1 /etc/default/horizon
   ```

The horizon-container script will automatically mount the certificate into the container when `HZN_MGMT_HUB_CERT_PATH` is set.

### 2. ESS (Edge Sync Service) Socket Permission Failure

**Symptoms:**
- Container status shows "Up X hours (starting)" - never reaches "healthy"
- Registration hangs at "Changing Horizon state to configured"
- Registration fails with: `Error setting node state to configured: Put "http://localhost:8081/node/configstate": EOF`
- Container logs show:
  ```
  Failed to setup permission for Unix Socket listening. Error: chmod /private/var/tmp/horizon/horizon1/fss-domain-socket/essapi.sock: invalid argument
  ```

**Root Cause:**
macOS handles Unix socket permissions differently than Linux. The containerized agent's ESS component expects Linux-style socket operations, which fail on macOS with "invalid argument" when attempting to chmod the socket file.

**Impact:**
- Agent API becomes unresponsive (returns EOF)
- Node cannot complete registration
- Services cannot be deployed

**Why This Works on Linux but Not macOS:**
- Linux allows chmod operations on Unix domain sockets
- macOS restricts socket permission changes, treating them as invalid operations
- The ESS component is critical for Model Management System (MMS) functionality
- Without ESS, the agent cannot fully initialize

**Solutions:**

#### Option 1: Use Native Linux Host (Recommended)
Run the containerized agent on a supported Linux distribution:
- Ubuntu 24.04 LTS
- RHEL 8/9
- Other supported Linux distributions

#### Option 2: Use Linux VM on macOS
1. Install a VM solution (UTM, Parallels, VirtualBox, VMware Fusion)
2. Create Ubuntu 24.04 VM
3. Install the containerized agent in the VM
4. Register the VM as the edge node

#### Option 3: Check for Native macOS Agent
- Investigate if a native macOS agent package is available
- Check Open Horizon documentation for macOS-specific installation methods
- Consider using agent-install.sh with different options

## Verification Steps

### Check Agent Status
```bash
# Verify container is running
podman ps -a --filter name=horizon1

# Check agent API connectivity
curl -s http://localhost:8081/status

# View node configuration
hzn node list
```

### Check Certificate Configuration
```bash
# Verify certificate file exists
ls -la /tmp/agent-install.crt

# Check if certificate is mounted in container
podman exec horizon1 ls -la /tmp/agent-install.crt

# Verify configuration includes certificate path
cat /etc/default/horizon | grep CERT
```

### Check ESS Status
```bash
# Check for ESS socket errors in logs
podman logs horizon1 2>&1 | grep -i "ess\|socket"

# Verify socket directory exists
ls -la /private/var/tmp/horizon/horizon1/fss-domain-socket/

# Check container status (should show "Up" not "starting")
podman ps --filter name=horizon1
```

## Configuration Files

### /etc/default/horizon
Required environment variables for containerized agent:
```bash
HZN_DEVICE_ID=<node-id>
HZN_NODE_ID=<node-id>
DOCKER_ENGINE=podman
HZN_AGENT_PORT=8510
HZN_ORG_ID=<org-id>
HZN_EXCHANGE_USER_AUTH=<auth-credentials>
HZN_EXCHANGE_URL=<exchange-url>
HZN_FSS_CSSURL=<css-url>
HZN_AGBOT_URL=<agbot-url>
HZN_MGMT_HUB_CERT_PATH=/tmp/agent-install.crt  # Required for self-signed certs
```

## Known Limitations

### macOS Containerized Agent
- **ESS Socket Issue**: Cannot run containerized agent on macOS due to Unix socket permission limitations
- **Workaround Required**: Must use Linux host or Linux VM
- **Status**: This is a fundamental incompatibility between macOS and the Linux-based containerized agent

### Certificate Handling
- Self-signed certificates require explicit configuration via `HZN_MGMT_HUB_CERT_PATH`
- Certificate must be accessible on the host filesystem
- Certificate is automatically mounted into container by horizon-container script

## Related Documentation
- Open Horizon Agent Installation: https://open-horizon.github.io/
- Containerized Agent Setup: https://github.com/open-horizon/anax/blob/master/agent-install/README.md
- MCP Server Configuration: See `.bob/skills/` for related skills

## Troubleshooting Checklist

- [ ] Verify CA certificate file exists and is readable
- [ ] Confirm `HZN_MGMT_HUB_CERT_PATH` is set in `/etc/default/horizon`
- [ ] Check container is running (not stuck in "starting" state)
- [ ] Verify agent API responds on localhost:8081
- [ ] Review container logs for ESS socket errors
- [ ] Confirm Exchange connectivity with `hzn exchange user list`
- [ ] If on macOS, consider migrating to Linux host or VM

## Success Criteria
- Container status shows "Up" (not "starting")
- Agent API responds to status requests
- `hzn node list` shows agent configuration
- No ESS socket errors in container logs
- Registration completes successfully without EOF errors
