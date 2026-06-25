# Deployment Troubleshooting Guide

This guide provides solutions to common issues encountered during Ollama service deployment on Open Horizon edge nodes.

## Table of Contents

- [Agent Installation Issues](#agent-installation-issues)
- [Service Publishing Problems](#service-publishing-problems)
- [Node Registration Failures](#node-registration-failures)
- [Agreement Formation Issues](#agreement-formation-issues)
- [Container Deployment Problems](#container-deployment-problems)
- [Network Connectivity Issues](#network-connectivity-issues)
- [Resource Constraints](#resource-constraints)
- [Policy Mismatch Problems](#policy-mismatch-problems)

## Agent Installation Issues

### Agent Not Starting

**Symptom:**
```bash
$ hzn version
bash: hzn: command not found
```

**Diagnosis:**
```bash
# Check if agent is installed
which hzn

# Check agent service status
sudo systemctl status horizon
```

**Solutions:**

1. **Agent not installed:**
   ```bash
   # Follow installation guide for your platform
   # Ubuntu/Debian example:
   wget -qO - http://pkg.bluehorizon.network/bluehorizon.network-public.key | sudo apt-key add -
   sudo add-apt-repository "deb [arch=$(dpkg --print-architecture)] http://pkg.bluehorizon.network/linux/ubuntu $(lsb_release -cs)-updates main"
   sudo apt-get update
   sudo apt-get install -y horizon
   ```

2. **Agent service not running:**
   ```bash
   sudo systemctl start horizon
   sudo systemctl enable horizon
   ```

3. **Path issue:**
   ```bash
   # Add to PATH
   export PATH=$PATH:/usr/bin
   # Or create symlink
   sudo ln -s /usr/horizon/bin/hzn /usr/local/bin/hzn
   ```

### Agent Cannot Connect to Exchange

**Symptom:**
```bash
$ hzn exchange status
Error: failed to connect to Exchange
```

**Diagnosis:**
```bash
# Check Exchange URL
echo $HZN_EXCHANGE_URL

# Test connectivity
curl -sS $HZN_EXCHANGE_URL/status

# Check agent configuration
cat /etc/default/horizon
```

**Solutions:**

1. **Incorrect Exchange URL:**
   ```bash
   export HZN_EXCHANGE_URL="https://correct-exchange.example.com/v1"
   # Update agent config
   sudo systemctl restart horizon
   ```

2. **Network connectivity:**
   ```bash
   # Test DNS resolution
   nslookup exchange.example.com
   
   # Test HTTPS connectivity
   curl -v https://exchange.example.com/v1/status
   
   # Check firewall
   sudo iptables -L -n | grep 443
   ```

3. **Invalid credentials:**
   ```bash
   # Verify credentials format: org/user:password
   export HZN_EXCHANGE_USER_AUTH="myorg/myuser:mypassword"
   
   # Test authentication
   hzn exchange user list
   ```

## Service Publishing Problems

### Service Validation Fails

**Symptom:**
```bash
$ make validate-service
Error: invalid service definition
```

**Diagnosis:**
```bash
# Check JSON syntax
cat horizon/service.definition.json | jq .

# Validate manually
hzn dev service verify -f horizon/service.definition.json
```

**Common Issues:**

1. **Invalid JSON syntax:**
   ```bash
   # Use jq to validate and format
   jq . horizon/service.definition.json > temp.json
   mv temp.json horizon/service.definition.json
   ```

2. **Missing required fields:**
   - Ensure `org`, `url`, `version`, `arch` are present
   - Verify `deployment.services` is properly structured
   - Check `userInput` array format

3. **Environment variable not expanded:**
   ```bash
   # Verify variables are set
   echo $HZN_ORG_ID
   
   # Use envsubst to expand
   cat horizon/service.definition.json | envsubst
   ```

### Service Already Exists

**Symptom:**
```bash
$ make publish-service
Error: service already exists
```

**Solutions:**

1. **Remove existing service:**
   ```bash
   make remove-service
   # Then republish
   make publish-service
   ```

2. **Increment version:**
   ```bash
   # Update SERVICE_VERSION
   export SERVICE_VERSION="0.1.1"
   # Update in service.definition.json
   sed -i 's/"version": "0.1.0"/"version": "0.1.1"/' horizon/service.definition.json
   ```

3. **Use force flag:**
   ```bash
   hzn exchange service publish -O -P -f horizon/service.definition.json
   ```

### Permission Denied

**Symptom:**
```bash
$ make publish-service
Error: permission denied
```

**Solutions:**

1. **Check user permissions:**
   ```bash
   # Verify you're in the correct org
   hzn exchange user list
   
   # Check if user has admin privileges
   hzn exchange user list $HZN_ORG_ID/$USER
   ```

2. **Use correct credentials:**
   ```bash
   # Ensure credentials have publish permissions
   export HZN_EXCHANGE_USER_AUTH="admin:adminpassword"
   ```

## Node Registration Failures

### Registration Hangs

**Symptom:**
```bash
$ make agent-run
Registering node...
[hangs indefinitely]
```

**Diagnosis:**
```bash
# Check agent logs
journalctl -u horizon -f

# Check event log
hzn eventlog list

# Verify agent status
hzn node list
```

**Solutions:**

1. **Agent not running:**
   ```bash
   sudo systemctl restart horizon
   sleep 5
   make agent-run
   ```

2. **Previous registration exists:**
   ```bash
   # Unregister first
   hzn unregister -f
   # Then register again
   make agent-run
   ```

3. **Network timeout:**
   ```bash
   # Increase timeout in agent config
   sudo vi /etc/default/horizon
   # Add: HZN_EXCHANGE_TIMEOUT=120
   sudo systemctl restart horizon
   ```

### Node Policy Invalid

**Symptom:**
```bash
$ make agent-run
Error: invalid node policy
```

**Diagnosis:**
```bash
# Validate policy JSON
cat horizon/node.policy.json | jq .

# Check policy format
hzn policy list
```

**Solutions:**

1. **Fix JSON syntax:**
   ```bash
   jq . horizon/node.policy.json > temp.json
   mv temp.json horizon/node.policy.json
   ```

2. **Verify property types:**
   - Memory and storage should be integers
   - Architecture should be string
   - Ensure proper nesting of objects

3. **Check property names:**
   ```bash
   # Standard properties start with "openhorizon."
   # Custom properties should not conflict
   ```

## Agreement Formation Issues

### No Agreements Formed

**Symptom:**
```bash
$ hzn agreement list
[]
```

**Diagnosis:**
```bash
# Run deployment check
make deploy-check

# Check event log for details
hzn eventlog list | grep -i agreement

# Verify policies
hzn policy list
hzn exchange deployment listpolicy
```

**Solutions:**

1. **Policy constraints mismatch:**
   ```bash
   # Run comprehensive check
   hzn deploycheck all -t device \
     -B horizon/deployment.policy.json \
     --service=horizon/service.definition.json \
     --service-pol=horizon/service.policy.json \
     --node-pol=horizon/node.policy.json
   ```

2. **Adjust node properties:**
   ```bash
   # Update node policy to match deployment requirements
   # Example: increase memory
   sed -i 's/"value": 2048/"value": 4096/' horizon/node.policy.json
   
   # Re-register
   hzn unregister -f
   make agent-run
   ```

3. **Service not available:**
   ```bash
   # Verify service is published
   make list-services
   
   # Check deployment policy exists
   hzn exchange deployment listpolicy $HZN_ORG_ID/
   ```

### Agreement Formed But Service Not Running

**Symptom:**
```bash
$ hzn agreement list
[shows agreement]
$ docker ps
[no ollama container]
```

**Diagnosis:**
```bash
# Check agreement details
hzn agreement list -r

# Check service logs
hzn service log service-ollama

# Check event log
hzn eventlog list | tail -20
```

**Solutions:**

1. **Container image pull failed:**
   ```bash
   # Manually pull image
   docker pull ollama/ollama:latest
   
   # Check disk space
   df -h
   
   # Check Docker Hub connectivity
   curl -sS https://hub.docker.com
   ```

2. **Resource constraints:**
   ```bash
   # Check available resources
   free -h
   df -h
   
   # Stop other containers if needed
   docker stop $(docker ps -q)
   ```

3. **Port conflict:**
   ```bash
   # Check if port 11434 is in use
   sudo netstat -tlnp | grep 11434
   
   # Stop conflicting service
   sudo systemctl stop <conflicting-service>
   ```

## Container Deployment Problems

### Container Exits Immediately

**Symptom:**
```bash
$ docker ps -a | grep ollama
[shows exited container]
```

**Diagnosis:**
```bash
# Check container logs
docker logs $(docker ps -aqf "ancestor=ollama/ollama:latest")

# Check exit code
docker inspect $(docker ps -aqf "ancestor=ollama/ollama:latest") | grep ExitCode
```

**Solutions:**

1. **Volume mount issue:**
   ```bash
   # Verify volume exists
   docker volume ls | grep ollama
   
   # Create if missing
   docker volume create ollama-storage
   
   # Check permissions
   ls -la /var/ollama
   sudo chmod 755 /var/ollama
   ```

2. **Environment variable issue:**
   ```bash
   # Check environment in service definition
   cat horizon/service.definition.json | jq '.deployment.services.ollama.environment'
   
   # Verify variables are valid
   ```

3. **Image corruption:**
   ```bash
   # Remove and re-pull image
   docker rmi ollama/ollama:latest
   docker pull ollama/ollama:latest
   ```

### Container Running But API Not Responding

**Symptom:**
```bash
$ docker ps | grep ollama
[shows running container]
$ curl http://localhost:11434
curl: (7) Failed to connect
```

**Diagnosis:**
```bash
# Check container logs
docker logs -f $(docker ps -qf "ancestor=ollama/ollama:latest")

# Check port binding
docker port $(docker ps -qf "ancestor=ollama/ollama:latest")

# Check if process is listening
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") netstat -tlnp
```

**Solutions:**

1. **Port not exposed:**
   ```bash
   # Verify port mapping in service definition
   cat horizon/service.definition.json | jq '.deployment.services.ollama.ports'
   
   # Should show: "HostPort": "11434:11434/tcp"
   ```

2. **Firewall blocking:**
   ```bash
   # Check firewall rules
   sudo iptables -L -n | grep 11434
   
   # Allow port
   sudo ufw allow 11434/tcp
   ```

3. **Service still starting:**
   ```bash
   # Wait for service to fully start
   sleep 30
   curl http://localhost:11434/api/version
   ```

## Network Connectivity Issues

### Cannot Pull Container Image

**Symptom:**
```bash
Error: failed to pull image ollama/ollama:latest
```

**Solutions:**

1. **Check internet connectivity:**
   ```bash
   ping -c 3 8.8.8.8
   curl -sS https://hub.docker.com
   ```

2. **DNS resolution:**
   ```bash
   nslookup hub.docker.com
   
   # If fails, update DNS
   echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
   ```

3. **Proxy configuration:**
   ```bash
   # Configure Docker proxy
   sudo mkdir -p /etc/systemd/system/docker.service.d
   sudo vi /etc/systemd/system/docker.service.d/http-proxy.conf
   # Add:
   # [Service]
   # Environment="HTTP_PROXY=http://proxy.example.com:8080"
   # Environment="HTTPS_PROXY=http://proxy.example.com:8080"
   
   sudo systemctl daemon-reload
   sudo systemctl restart docker
   ```

## Resource Constraints

### Insufficient Memory

**Symptom:**
```bash
Error: cannot allocate memory
```

**Solutions:**

1. **Check available memory:**
   ```bash
   free -h
   ```

2. **Increase swap:**
   ```bash
   sudo fallocate -l 4G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   ```

3. **Adjust deployment policy:**
   ```bash
   # Reduce memory requirement
   sed -i 's/openhorizon.memory >= 4096/openhorizon.memory >= 2048/' horizon/deployment.policy.json
   make publish-deployment-policy
   ```

### Insufficient Storage

**Symptom:**
```bash
Error: no space left on device
```

**Solutions:**

1. **Clean up Docker:**
   ```bash
   docker system prune -a
   docker volume prune
   ```

2. **Move Docker data directory:**
   ```bash
   sudo systemctl stop docker
   sudo vi /etc/docker/daemon.json
   # Add: {"data-root": "/mnt/docker"}
   sudo systemctl start docker
   ```

3. **Use external storage:**
   ```bash
   # Mount external drive
   sudo mount /dev/sdb1 /var/ollama
   # Update fstab for persistence
   ```

## Policy Mismatch Problems

### Deployment Check Failures

**Symptom:**
```bash
$ make deploy-check
Compatibility check failed
```

**Solutions:**

1. **Review detailed output:**
   ```bash
   hzn deploycheck all -t device -B horizon/deployment.policy.json \
     --service=horizon/service.definition.json \
     --service-pol=horizon/service.policy.json \
     --node-pol=horizon/node.policy.json \
     -v
   ```

2. **Common mismatches:**
   - Architecture: Ensure node arch matches service arch
   - Memory: Node must meet minimum memory requirement
   - Storage: Node must have sufficient storage
   - Custom properties: Verify property names match exactly

3. **Fix and retry:**
   ```bash
   # Update policies as needed
   # Republish if deployment policy changed
   make publish-deployment-policy
   
   # Re-register if node policy changed
   hzn unregister -f
   make agent-run
   ```

## Getting Additional Help

### Collect Diagnostic Information

```bash
# System information
uname -a
cat /etc/os-release

# Agent version and status
hzn version
hzn node list

# Service and agreement status
hzn service list
hzn agreement list

# Event log (last 50 entries)
hzn eventlog list | tail -50

# Container status
docker ps -a
docker logs $(docker ps -aqf "ancestor=ollama/ollama:latest")

# Resource usage
free -h
df -h
```

### Enable Debug Logging

```bash
# Enable debug mode
sudo vi /etc/default/horizon
# Add: HZN_AGENT_DEBUG=1

# Restart agent
sudo systemctl restart horizon

# View detailed logs
journalctl -u horizon -f
```

### Contact Support

When contacting support, provide:
- Output from diagnostic commands above
- Relevant log excerpts
- Service definition and policy files
- Description of what you were trying to do
- Any error messages received

## Additional Resources

- [Open Horizon Troubleshooting](https://open-horizon.github.io/troubleshooting/)
- [Docker Troubleshooting](https://docs.docker.com/config/daemon/troubleshoot/)
- [Ollama Issues](https://github.com/ollama/ollama/issues)
