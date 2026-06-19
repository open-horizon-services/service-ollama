# Monitoring and Diagnostics

This guide covers procedures for monitoring Ollama service health and diagnosing operational issues.

## Overview

Effective monitoring ensures your Ollama service remains healthy and performs optimally. This guide covers service status checks, container monitoring, log access, and resource usage tracking.

## Service Status Monitoring

### Check Node Status

```bash
# View node information
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

**Key Fields:**
- `configstate.state`: Should be "configured"
- `token_valid`: Should be true
- `token_last_valid_time`: Recent timestamp

### Check Service Status

```bash
# List running services
hzn service list
```

**Expected Output:**
```json
[
  {
    "url": "service-ollama",
    "org": "your-org-id",
    "version": "0.1.0",
    "arch": "amd64",
    "variables": {
      "OLLAMA_HOST": "0.0.0.0:11434",
      "OLLAMA_MODELS": "/root/.ollama/models",
      "OLLAMA_KEEP_ALIVE": "5m"
    }
  }
]
```

### Check Active Agreements

```bash
# List all agreements
hzn agreement list
```

**Expected Output:**
```json
[
  {
    "name": "Policy for your-org-id/service-ollama merged with your-org-id/policy-service-ollama_0.1.0",
    "current_agreement_id": "abc123def456...",
    "consumer_id": "IBM/agbot",
    "agreement_creation_time": "2024-01-15 10:30:00 -0500 EST",
    "agreement_accepted_time": "2024-01-15 10:30:15 -0500 EST",
    "agreement_finalized_time": "2024-01-15 10:30:30 -0500 EST",
    "agreement_execution_start_time": "2024-01-15 10:30:45 -0500 EST",
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

**Healthy Agreement Indicators:**
- `agreement_execution_start_time` is set
- No `terminated_time` field
- `workload_to_run` matches expected service

### Check Agreement Details

```bash
# Get detailed agreement information
hzn agreement list -r
```

This provides additional details including:
- Resource usage
- Metering information
- Agreement terms

## Container Status Monitoring

### Check Running Containers

```bash
# List Ollama containers
docker ps | grep ollama

# Or with Podman
podman ps | grep ollama
```

**Expected Output:**
```
CONTAINER ID   IMAGE                    COMMAND       CREATED         STATUS         PORTS                      NAMES
abc123def456   ollama/ollama:latest     "/bin/ollama"  2 hours ago     Up 2 hours     0.0.0.0:11434->11434/tcp   ollama
```

**Key Fields:**
- STATUS: Should show "Up" with uptime
- PORTS: Should show port mapping (11434)
- NAMES: Container name

### Check Container Details

```bash
# Inspect container
docker inspect $(docker ps -qf "ancestor=ollama/ollama:latest")
```

**Key Information:**
- State.Status: Should be "running"
- State.Health: If health checks configured
- Mounts: Verify volume mounts
- NetworkSettings: Verify port bindings

### Check Container Resource Usage

```bash
# Real-time resource usage
docker stats $(docker ps -qf "ancestor=ollama/ollama:latest")
```

**Example Output:**
```
CONTAINER ID   NAME     CPU %     MEM USAGE / LIMIT     MEM %     NET I/O           BLOCK I/O
abc123def456   ollama   15.23%    2.5GiB / 8GiB        31.25%    1.2MB / 850KB     45MB / 12MB
```

**Monitor:**
- CPU %: Should be reasonable for workload
- MEM USAGE: Should not approach limit
- NET I/O: Network activity
- BLOCK I/O: Disk activity

## Log Access and Analysis

### View Service Logs (Open Horizon)

```bash
# View service logs
hzn service log service-ollama

# Follow logs in real-time
hzn service log -f service-ollama

# View last N lines
hzn service log service-ollama --tail 100
```

**Example Output:**
```
time=2024-01-15T15:30:00Z level=info msg="Ollama server starting"
time=2024-01-15T15:30:01Z level=info msg="Listening on 0.0.0.0:11434"
time=2024-01-15T15:30:02Z level=info msg="Model loaded: llama2"
```

### View Container Logs (Docker)

```bash
# View container logs
docker logs $(docker ps -qf "ancestor=ollama/ollama:latest")

# Follow logs
docker logs -f $(docker ps -qf "ancestor=ollama/ollama:latest")

# View last 100 lines
docker logs --tail 100 $(docker ps -qf "ancestor=ollama/ollama:latest")

# View logs with timestamps
docker logs -t $(docker ps -qf "ancestor=ollama/ollama:latest")
```

### View Event Log

```bash
# View Open Horizon event log
hzn eventlog list

# View recent events
hzn eventlog list | tail -20

# Filter by severity
hzn eventlog list | grep -i error
hzn eventlog list | grep -i warning
```

**Event Types:**
- Agreement events
- Service lifecycle events
- Error conditions
- Policy changes

### Log Analysis Examples

**Check for Errors:**
```bash
# Search for errors in service logs
hzn service log service-ollama | grep -i error

# Search for warnings
docker logs $(docker ps -qf "ancestor=ollama/ollama:latest") 2>&1 | grep -i warning
```

**Monitor API Requests:**
```bash
# Follow logs and filter for API calls
docker logs -f $(docker ps -qf "ancestor=ollama/ollama:latest") | grep "POST\|GET"
```

**Check Model Loading:**
```bash
# Check for model load events
docker logs $(docker ps -qf "ancestor=ollama/ollama:latest") | grep -i "model"
```

## Resource Usage Monitoring

### CPU Monitoring

```bash
# Check CPU usage
top -b -n 1 | grep ollama

# Or with htop (if installed)
htop -p $(docker inspect -f '{{.State.Pid}}' $(docker ps -qf "ancestor=ollama/ollama:latest"))
```

**Monitor:**
- CPU percentage
- Load average
- Process state

### Memory Monitoring

```bash
# Check memory usage
free -h

# Check container memory
docker stats --no-stream $(docker ps -qf "ancestor=ollama/ollama:latest")

# Detailed memory info
cat /proc/$(docker inspect -f '{{.State.Pid}}' $(docker ps -qf "ancestor=ollama/ollama:latest"))/status | grep -i mem
```

**Key Metrics:**
- Total memory usage
- Memory limit
- Memory percentage
- Swap usage

### Storage Monitoring

```bash
# Check overall disk usage
df -h

# Check model storage
du -sh /var/ollama

# Check container storage
docker system df

# Detailed storage breakdown
du -h /var/ollama | sort -h | tail -20
```

**Monitor:**
- Available space
- Model directory size
- Container layer sizes
- Volume usage

### Network Monitoring

```bash
# Check network connections
netstat -tlnp | grep 11434

# Or with ss
ss -tlnp | grep 11434

# Monitor network traffic
iftop -i eth0

# Check API endpoint
curl -sS http://localhost:11434/api/version
```

**Monitor:**
- Port availability
- Connection count
- Network throughput
- API responsiveness

## Health Checks

### API Health Check

```bash
# Basic health check
curl -f http://localhost:11434/api/version || echo "API not responding"

# With timeout
timeout 5 curl -sS http://localhost:11434/api/version || echo "API timeout"
```

### Model Availability Check

```bash
# List models
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama list

# Test specific model
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama run llama2 "test" --verbose
```

### End-to-End Test

```bash
# Complete functionality test
curl -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt": "Say hello",
  "stream": false
}' | jq .
```

## Automated Monitoring Scripts

### Basic Health Check Script

```bash
#!/bin/bash
# health-check.sh

echo "=== Ollama Service Health Check ==="
echo "Timestamp: $(date)"
echo ""

# Check node status
echo "Node Status:"
hzn node list | jq -r '.configstate.state'

# Check agreements
echo "Active Agreements:"
hzn agreement list | jq -r 'length'

# Check container
echo "Container Status:"
docker ps --filter "ancestor=ollama/ollama:latest" --format "{{.Status}}"

# Check API
echo "API Status:"
if curl -sf http://localhost:11434/api/version > /dev/null; then
  echo "✓ API responding"
else
  echo "✗ API not responding"
fi

# Check resources
echo "Resource Usage:"
docker stats --no-stream $(docker ps -qf "ancestor=ollama/ollama:latest") --format "CPU: {{.CPUPerc}} | Memory: {{.MemUsage}}"

echo ""
echo "=== Health Check Complete ==="
```

### Continuous Monitoring Script

```bash
#!/bin/bash
# monitor.sh

while true; do
  clear
  echo "=== Ollama Service Monitor ==="
  echo "Time: $(date)"
  echo ""
  
  # Service status
  echo "Service Status:"
  hzn service list | jq -r '.[0] | "\(.url) v\(.version) - \(.arch)"'
  
  # Container status
  echo ""
  echo "Container:"
  docker ps --filter "ancestor=ollama/ollama:latest" --format "{{.Names}}: {{.Status}}"
  
  # Resource usage
  echo ""
  echo "Resources:"
  docker stats --no-stream $(docker ps -qf "ancestor=ollama/ollama:latest") --format "CPU: {{.CPUPerc}} | Mem: {{.MemPerc}} | Net I/O: {{.NetIO}}"
  
  # API check
  echo ""
  echo "API:"
  curl -sf http://localhost:11434/api/version | jq -r '"Version: \(.version)"' || echo "API not responding"
  
  # Storage
  echo ""
  echo "Storage:"
  df -h /var/ollama | tail -1 | awk '{print "Used: " $3 " / " $2 " (" $5 ")"}'
  
  sleep 10
done
```

### Alert Script

```bash
#!/bin/bash
# alert.sh

# Thresholds
CPU_THRESHOLD=80
MEM_THRESHOLD=80
DISK_THRESHOLD=85

# Check CPU
CPU_USAGE=$(docker stats --no-stream $(docker ps -qf "ancestor=ollama/ollama:latest") --format "{{.CPUPerc}}" | sed 's/%//')
if (( $(echo "$CPU_USAGE > $CPU_THRESHOLD" | bc -l) )); then
  echo "ALERT: High CPU usage: ${CPU_USAGE}%"
fi

# Check Memory
MEM_USAGE=$(docker stats --no-stream $(docker ps -qf "ancestor=ollama/ollama:latest") --format "{{.MemPerc}}" | sed 's/%//')
if (( $(echo "$MEM_USAGE > $MEM_THRESHOLD" | bc -l) )); then
  echo "ALERT: High memory usage: ${MEM_USAGE}%"
fi

# Check Disk
DISK_USAGE=$(df -h /var/ollama | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$DISK_USAGE" -gt "$DISK_THRESHOLD" ]; then
  echo "ALERT: High disk usage: ${DISK_USAGE}%"
fi

# Check API
if ! curl -sf http://localhost:11434/api/version > /dev/null; then
  echo "ALERT: API not responding"
fi
```

## Performance Diagnostics

### Identify Performance Issues

**Slow Response Times:**
```bash
# Measure API response time
time curl -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt": "Test",
  "stream": false
}'

# Check CPU usage during inference
docker stats $(docker ps -qf "ancestor=ollama/ollama:latest")
```

**High Memory Usage:**
```bash
# Check loaded models
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama list

# Check memory breakdown
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ps aux

# Adjust keep-alive to unload models faster
# (Update service definition with shorter OLLAMA_KEEP_ALIVE)
```

**Disk I/O Issues:**
```bash
# Monitor disk I/O
iostat -x 1

# Check for disk errors
dmesg | grep -i error

# Verify volume performance
dd if=/dev/zero of=/var/ollama/test bs=1M count=1000
rm /var/ollama/test
```

### Common Performance Problems

**Problem: Slow Model Loading**
```bash
# Check model size
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") du -sh /root/.ollama/models/*

# Monitor during load
docker stats $(docker ps -qf "ancestor=ollama/ollama:latest")

# Solution: Use smaller models or increase memory
```

**Problem: High CPU Usage**
```bash
# Identify CPU-intensive processes
top -b -n 1 | head -20

# Check concurrent requests
netstat -an | grep 11434 | grep ESTABLISHED | wc -l

# Solution: Limit concurrent requests or scale horizontally
```

**Problem: Memory Leaks**
```bash
# Monitor memory over time
while true; do
  docker stats --no-stream $(docker ps -qf "ancestor=ollama/ollama:latest") --format "{{.MemUsage}}"
  sleep 60
done

# Solution: Restart service periodically or update to newer version
```

## Monitoring Best Practices

1. **Regular Health Checks:**
   - Run health checks every 5-10 minutes
   - Alert on failures
   - Track trends over time

2. **Resource Monitoring:**
   - Monitor CPU, memory, disk, network
   - Set appropriate thresholds
   - Plan capacity based on trends

3. **Log Management:**
   - Rotate logs regularly
   - Archive important logs
   - Set up centralized logging if possible

4. **Performance Baselines:**
   - Establish normal performance metrics
   - Track deviations from baseline
   - Investigate anomalies promptly

5. **Proactive Monitoring:**
   - Monitor trends, not just current state
   - Predict resource exhaustion
   - Plan upgrades before issues occur

## Next Steps

- Review [Storage Management](storage-management.md) for managing persistent data
- See [API Access](api-access.md) for testing API functionality
- Check [Service Updates](service-updates.md) for maintenance procedures

## Additional Resources

- [Open Horizon Monitoring](https://open-horizon.github.io/docs/monitoring/)
- [Docker Monitoring](https://docs.docker.com/config/containers/runmetrics/)
- [Ollama Performance](https://github.com/ollama/ollama/blob/main/docs/faq.md#performance)
