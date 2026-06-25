# Model Management

This guide covers Day 2 operations for managing Ollama models on deployed edge nodes.

## Overview

Ollama models are managed independently from the service container. This allows you to add, remove, and update models without redeploying the service.

## Prerequisites

- Ollama service deployed and running
- Access to the edge node (SSH or local)
- Sufficient storage for models (varies by model size)

## Model Operations

### Listing Available Models

To see what models are currently installed:

```bash
# Access the container
docker exec -it $(docker ps -qf "ancestor=ollama/ollama:latest") /bin/bash

# List installed models
ollama list
```

**Example Output:**
```
NAME                    ID              SIZE    MODIFIED
llama2:latest          78e26419b446    3.8 GB  2 hours ago
codellama:7b           8fdf8f752f6e    3.8 GB  1 day ago
mistral:latest         61e88e884507    4.1 GB  3 days ago
```

**Alternative (from host):**
```bash
docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") ollama list
```

### Pulling/Adding New Models

To add a new model to your Ollama instance:

```bash
# Access the container
docker exec -it $(docker ps -qf "ancestor=ollama/ollama:latest") /bin/bash

# Pull a model
ollama pull <model-name>
```

**Common Models:**

```bash
# Small models (good for resource-constrained devices)
ollama pull phi           # 2.7 GB
ollama pull tinyllama     # 637 MB

# Medium models (balanced performance)
ollama pull llama2        # 3.8 GB
ollama pull mistral       # 4.1 GB
ollama pull codellama:7b  # 3.8 GB

# Large models (high performance, requires more resources)
ollama pull llama2:13b    # 7.3 GB
ollama pull codellama:13b # 7.3 GB
ollama pull llama2:70b    # 39 GB
```

**Pull Progress:**
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

**Pull Specific Version:**
```bash
# Pull specific version tag
ollama pull llama2:13b
ollama pull codellama:7b-instruct
ollama pull mistral:7b-instruct-v0.2
```

### Verifying Model Availability

After pulling a model, verify it's available and functional:

```bash
# List models to confirm
ollama list

# Test the model with a simple prompt
ollama run llama2 "Hello, how are you?"
```

**Expected Output:**
```
Hello! I'm doing well, thank you for asking. How can I assist you today?
```

**API Verification:**
```bash
# From host machine
curl -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt": "Test prompt",
  "stream": false
}'
```

### Removing Models

To free up storage space by removing unused models:

```bash
# Access the container
docker exec -it $(docker ps -qf "ancestor=ollama/ollama:latest") /bin/bash

# Remove a specific model
ollama rm <model-name>
```

**Example:**
```bash
# Remove a model
ollama rm codellama:7b

# Verify removal
ollama list
```

**Remove Multiple Models:**
```bash
# Remove several models
ollama rm llama2:13b
ollama rm mistral:latest
ollama rm codellama:13b
```

**Check Storage Freed:**
```bash
# Check disk usage before
df -h /root/.ollama

# Remove model
ollama rm large-model

# Check disk usage after
df -h /root/.ollama
```

### Updating Existing Models

Models are updated by pulling the latest version:

```bash
# Access the container
docker exec -it $(docker ps -qf "ancestor=ollama/ollama:latest") /bin/bash

# Pull the latest version (will update if newer available)
ollama pull llama2:latest
```

**Update Process:**
1. Ollama checks for newer version
2. Downloads only changed layers (efficient)
3. Replaces old version with new version
4. Old version is automatically removed

**Example Output:**
```
pulling manifest
pulling 8934d96d3f08... already exists
pulling 8c17c2ebb0ea... 100% ▕████████████████▏ 7.1 KB (updated)
pulling 7c23fb36d801... already exists
verifying sha256 digest
writing manifest
success
```

**Update All Models:**
```bash
# List all models
ollama list | tail -n +2 | awk '{print $1}' > models.txt

# Update each model
while read model; do
  echo "Updating $model..."
  ollama pull "$model"
done < models.txt
```

## Model Storage Management

### Check Model Storage Location

```bash
# Inside container
echo $OLLAMA_MODELS
# Default: /root/.ollama/models

# Check storage usage
du -sh /root/.ollama/models
```

### Check Available Storage

```bash
# From host
df -h /var/ollama

# Inside container
df -h /root/.ollama
```

### Model Storage Best Practices

1. **Plan Storage Capacity:**
   - Small models: 1-3 GB each
   - Medium models: 4-8 GB each
   - Large models: 10-40 GB each
   - Reserve 20% extra for temporary files

2. **Regular Cleanup:**
   ```bash
   # Remove unused models monthly
   ollama list
   ollama rm <unused-model>
   ```

3. **Monitor Storage:**
   ```bash
   # Set up monitoring alert
   THRESHOLD=80
   USAGE=$(df -h /var/ollama | tail -1 | awk '{print $5}' | sed 's/%//')
   if [ $USAGE -gt $THRESHOLD ]; then
     echo "Warning: Storage usage at ${USAGE}%"
   fi
   ```

## Common Model Operations Examples

### Example 1: Deploy Multiple Models for Different Use Cases

```bash
# Access container
docker exec -it $(docker ps -qf "ancestor=ollama/ollama:latest") /bin/bash

# Pull models for different purposes
ollama pull llama2          # General purpose
ollama pull codellama:7b    # Code generation
ollama pull mistral         # Instruction following
ollama pull phi             # Lightweight tasks

# Verify all models
ollama list
```

### Example 2: Replace Large Model with Smaller Alternative

```bash
# Current: llama2:70b (39 GB)
# Target: llama2:13b (7.3 GB)

# Pull smaller model
ollama pull llama2:13b

# Test smaller model
ollama run llama2:13b "Test prompt"

# If satisfied, remove large model
ollama rm llama2:70b

# Verify storage freed
df -h /root/.ollama
```

### Example 3: Batch Model Update Script

```bash
#!/bin/bash
# update-models.sh

MODELS=(
  "llama2:latest"
  "mistral:latest"
  "codellama:7b"
)

for model in "${MODELS[@]}"; do
  echo "Updating $model..."
  docker exec $(docker ps -qf "ancestor=ollama/ollama:latest") \
    ollama pull "$model"
  
  if [ $? -eq 0 ]; then
    echo "✓ $model updated successfully"
  else
    echo "✗ Failed to update $model"
  fi
done

echo "Update complete"
```

## Model Performance Considerations

### Model Size vs. Performance

| Model Size | RAM Required | Inference Speed | Use Case |
|------------|--------------|-----------------|----------|
| < 3B params | 2-4 GB | Fast | Simple tasks, constrained devices |
| 7B params | 4-8 GB | Medium | General purpose, balanced |
| 13B params | 8-16 GB | Slower | Complex tasks, better quality |
| 70B+ params | 32+ GB | Slow | Highest quality, powerful hardware |

### Optimizing Model Selection

1. **Start Small:**
   ```bash
   # Begin with smaller models
   ollama pull phi
   ollama pull tinyllama
   ```

2. **Test Performance:**
   ```bash
   # Benchmark response time
   time ollama run phi "Explain quantum computing"
   ```

3. **Scale Up if Needed:**
   ```bash
   # If quality insufficient, try larger model
   ollama pull llama2:13b
   ```

## Troubleshooting

### Model Pull Fails

**Symptom:** Download fails or times out

**Solutions:**
```bash
# Check network connectivity
curl -sS https://ollama.ai

# Check available storage
df -h /var/ollama

# Retry with verbose output
ollama pull llama2 --verbose

# Try different model registry if available
```

### Model Not Found After Pull

**Symptom:** Model pulled but not listed

**Solutions:**
```bash
# Verify model directory
ls -la /root/.ollama/models/manifests/registry.ollama.ai/library/

# Check for corruption
ollama list

# Re-pull if necessary
ollama rm <model>
ollama pull <model>
```

### Out of Memory During Model Load

**Symptom:** Model fails to load with OOM error

**Solutions:**
```bash
# Check available memory
free -h

# Try smaller model
ollama pull phi  # Instead of llama2:70b

# Adjust keep-alive to unload faster
export OLLAMA_KEEP_ALIVE=1m
```

### Slow Model Performance

**Symptom:** Model responses are very slow

**Solutions:**
```bash
# Check CPU/memory usage
top

# Use smaller model
ollama pull tinyllama

# Reduce concurrent requests
# Ensure only one model loaded at a time
```

## Best Practices

1. **Model Lifecycle:**
   - Pull models during off-peak hours
   - Test new models before removing old ones
   - Keep at least one working model at all times

2. **Storage Management:**
   - Monitor storage usage regularly
   - Remove unused models promptly
   - Plan for model size before pulling

3. **Version Control:**
   - Document which models are in use
   - Pin specific versions for production
   - Test updates in staging first

4. **Performance Optimization:**
   - Match model size to hardware capabilities
   - Use appropriate models for specific tasks
   - Monitor inference times and adjust

## Next Steps

- Review [Service Updates](service-updates.md) for updating Ollama itself
- See [Monitoring](monitoring.md) for tracking model performance
- Check [API Access](api-access.md) for using models via API

## Additional Resources

- [Ollama Model Library](https://ollama.ai/library)
- [Model Performance Benchmarks](https://github.com/ollama/ollama#model-library)
- [Ollama CLI Reference](https://github.com/ollama/ollama/blob/main/docs/cli.md)
