# API Access and Integration

This guide covers procedures for accessing the Ollama API from edge applications and verifying API functionality.

## Overview

The Ollama service exposes a REST API on port 11434 for model inference and management. This guide covers API connectivity, authentication, and common integration patterns.

## API Endpoint

**Default Endpoint:** `http://localhost:11434`

**Available from:**
- Same host as container
- Other containers on same network
- External clients (if firewall configured)

## API Connectivity Testing

### Basic Connectivity Test

```bash
# Test API is responding
curl -sS http://localhost:11434/api/version
```

**Expected Response:**
```json
{
  "version": "0.1.17"
}
```

### Test from Remote Host

```bash
# Replace <edge-node-ip> with actual IP
curl -sS http://<edge-node-ip>:11434/api/version
```

**If Connection Fails:**
```bash
# Check firewall
sudo ufw status | grep 11434

# Allow port if needed
sudo ufw allow 11434/tcp

# Verify container port binding
docker port $(docker ps -qf "ancestor=ollama/ollama:latest")
```

### Health Check Endpoint

```bash
# Check service health
curl -f http://localhost:11434/api/version || echo "Service unhealthy"
```

## API Authentication

### Current Status

**Ollama does not currently support built-in authentication.** The API is open to anyone who can reach the endpoint.

### Security Recommendations

#### Option 1: Network-Level Security

```bash
# Restrict access to specific IPs
sudo iptables -A INPUT -p tcp --dport 11434 -s 192.168.1.0/24 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 11434 -j DROP

# Or with UFW
sudo ufw allow from 192.168.1.0/24 to any port 11434
sudo ufw deny 11434
```

#### Option 2: Reverse Proxy with Authentication

**Using Nginx:**
```nginx
# /etc/nginx/sites-available/ollama
server {
    listen 8080;
    
    location / {
        auth_basic "Ollama API";
        auth_basic_user_file /etc/nginx/.htpasswd;
        
        proxy_pass http://localhost:11434;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Create password file:**
```bash
sudo apt-get install apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd admin
sudo systemctl restart nginx
```

**Access with authentication:**
```bash
curl -u admin:password http://localhost:8080/api/version
```

#### Option 3: VPN Access Only

```bash
# Configure service to listen only on VPN interface
# Update service definition:
{
  "ports": [
    {
      "HostPort": "10.8.0.1:11434:11434/tcp",
      "HostIP": "10.8.0.1"
    }
  ]
}
```

## API Usage Examples

### Generate Text

```bash
# Simple generation
curl -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt": "Why is the sky blue?",
  "stream": false
}'
```

**Response:**
```json
{
  "model": "llama2",
  "created_at": "2024-01-15T15:30:00.000Z",
  "response": "The sky appears blue because...",
  "done": true,
  "context": [...],
  "total_duration": 5000000000,
  "load_duration": 1000000000,
  "prompt_eval_count": 10,
  "prompt_eval_duration": 500000000,
  "eval_count": 50,
  "eval_duration": 3500000000
}
```

### Streaming Response

```bash
# Stream response tokens
curl -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt": "Tell me a story",
  "stream": true
}'
```

**Response (multiple JSON objects):**
```json
{"model":"llama2","created_at":"...","response":"Once","done":false}
{"model":"llama2","created_at":"...","response":" upon","done":false}
{"model":"llama2","created_at":"...","response":" a","done":false}
...
{"model":"llama2","created_at":"...","response":"","done":true}
```

### Chat Completion

```bash
# Chat with context
curl -X POST http://localhost:11434/api/chat -d '{
  "model": "llama2",
  "messages": [
    {
      "role": "user",
      "content": "Hello!"
    }
  ],
  "stream": false
}'
```

**Response:**
```json
{
  "model": "llama2",
  "created_at": "2024-01-15T15:30:00.000Z",
  "message": {
    "role": "assistant",
    "content": "Hello! How can I help you today?"
  },
  "done": true
}
```

### List Models

```bash
# Get available models
curl -sS http://localhost:11434/api/tags
```

**Response:**
```json
{
  "models": [
    {
      "name": "llama2:latest",
      "modified_at": "2024-01-15T10:00:00.000Z",
      "size": 3826793677,
      "digest": "sha256:78e26419b4...",
      "details": {
        "format": "gguf",
        "family": "llama",
        "families": ["llama"],
        "parameter_size": "7B",
        "quantization_level": "Q4_0"
      }
    }
  ]
}
```

### Show Model Information

```bash
# Get model details
curl -X POST http://localhost:11434/api/show -d '{
  "name": "llama2"
}'
```

### Pull Model

```bash
# Pull a model via API
curl -X POST http://localhost:11434/api/pull -d '{
  "name": "mistral"
}'
```

### Delete Model

```bash
# Remove a model via API
curl -X DELETE http://localhost:11434/api/delete -d '{
  "name": "mistral"
}'
```

## Integration Patterns

### Python Integration

```python
import requests
import json

class OllamaClient:
    def __init__(self, base_url="http://localhost:11434"):
        self.base_url = base_url
    
    def generate(self, model, prompt, stream=False):
        """Generate text from prompt"""
        url = f"{self.base_url}/api/generate"
        data = {
            "model": model,
            "prompt": prompt,
            "stream": stream
        }
        
        response = requests.post(url, json=data)
        
        if stream:
            for line in response.iter_lines():
                if line:
                    yield json.loads(line)
        else:
            return response.json()
    
    def chat(self, model, messages, stream=False):
        """Chat with context"""
        url = f"{self.base_url}/api/chat"
        data = {
            "model": model,
            "messages": messages,
            "stream": stream
        }
        
        response = requests.post(url, json=data)
        return response.json()
    
    def list_models(self):
        """List available models"""
        url = f"{self.base_url}/api/tags"
        response = requests.get(url)
        return response.json()

# Usage
client = OllamaClient()

# Generate text
result = client.generate("llama2", "Why is the sky blue?")
print(result["response"])

# Chat
messages = [
    {"role": "user", "content": "Hello!"}
]
response = client.chat("llama2", messages)
print(response["message"]["content"])

# List models
models = client.list_models()
for model in models["models"]:
    print(f"{model['name']}: {model['size']} bytes")
```

### JavaScript/Node.js Integration

```javascript
const axios = require('axios');

class OllamaClient {
  constructor(baseUrl = 'http://localhost:11434') {
    this.baseUrl = baseUrl;
  }

  async generate(model, prompt, stream = false) {
    const url = `${this.baseUrl}/api/generate`;
    const data = { model, prompt, stream };
    
    const response = await axios.post(url, data);
    return response.data;
  }

  async chat(model, messages, stream = false) {
    const url = `${this.baseUrl}/api/chat`;
    const data = { model, messages, stream };
    
    const response = await axios.post(url, data);
    return response.data;
  }

  async listModels() {
    const url = `${this.baseUrl}/api/tags`;
    const response = await axios.get(url);
    return response.data;
  }
}

// Usage
const client = new OllamaClient();

// Generate text
client.generate('llama2', 'Why is the sky blue?')
  .then(result => console.log(result.response));

// Chat
const messages = [
  { role: 'user', content: 'Hello!' }
];
client.chat('llama2', messages)
  .then(response => console.log(response.message.content));

// List models
client.listModels()
  .then(data => {
    data.models.forEach(model => {
      console.log(`${model.name}: ${model.size} bytes`);
    });
  });
```

### Bash/Shell Integration

```bash
#!/bin/bash
# ollama-client.sh

OLLAMA_URL="${OLLAMA_URL:-http://localhost:11434}"

generate() {
  local model="$1"
  local prompt="$2"
  
  curl -sS -X POST "$OLLAMA_URL/api/generate" -d "{
    \"model\": \"$model\",
    \"prompt\": \"$prompt\",
    \"stream\": false
  }" | jq -r '.response'
}

chat() {
  local model="$1"
  local message="$2"
  
  curl -sS -X POST "$OLLAMA_URL/api/chat" -d "{
    \"model\": \"$model\",
    \"messages\": [{\"role\": \"user\", \"content\": \"$message\"}],
    \"stream\": false
  }" | jq -r '.message.content'
}

list_models() {
  curl -sS "$OLLAMA_URL/api/tags" | jq -r '.models[].name'
}

# Usage
generate "llama2" "Why is the sky blue?"
chat "llama2" "Hello!"
list_models
```

## Advanced Usage

### Custom Parameters

```bash
# Control generation parameters
curl -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt": "Write a poem",
  "stream": false,
  "options": {
    "temperature": 0.8,
    "top_p": 0.9,
    "top_k": 40,
    "num_predict": 100
  }
}'
```

**Available Options:**
- `temperature`: Randomness (0.0-2.0, default 0.8)
- `top_p`: Nucleus sampling (0.0-1.0, default 0.9)
- `top_k`: Top-k sampling (default 40)
- `num_predict`: Max tokens to generate
- `stop`: Stop sequences
- `seed`: Random seed for reproducibility

### System Prompts

```bash
# Set system context
curl -X POST http://localhost:11434/api/chat -d '{
  "model": "llama2",
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful coding assistant."
    },
    {
      "role": "user",
      "content": "Write a Python function to sort a list"
    }
  ],
  "stream": false
}'
```

### Multi-Turn Conversations

```bash
# Maintain conversation context
curl -X POST http://localhost:11434/api/chat -d '{
  "model": "llama2",
  "messages": [
    {"role": "user", "content": "What is 2+2?"},
    {"role": "assistant", "content": "2+2 equals 4."},
    {"role": "user", "content": "What about 3+3?"}
  ],
  "stream": false
}'
```

## Performance Optimization

### Connection Pooling

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

# Configure session with connection pooling
session = requests.Session()
retry = Retry(total=3, backoff_factor=0.1)
adapter = HTTPAdapter(max_retries=retry, pool_connections=10, pool_maxsize=10)
session.mount('http://', adapter)

# Use session for requests
response = session.post('http://localhost:11434/api/generate', json={
    'model': 'llama2',
    'prompt': 'test'
})
```

### Async Requests

```python
import asyncio
import aiohttp

async def generate_async(session, model, prompt):
    url = 'http://localhost:11434/api/generate'
    data = {'model': model, 'prompt': prompt, 'stream': False}
    
    async with session.post(url, json=data) as response:
        return await response.json()

async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [
            generate_async(session, 'llama2', f'Question {i}')
            for i in range(10)
        ]
        results = await asyncio.gather(*tasks)
        return results

# Run
results = asyncio.run(main())
```

## Error Handling

### Common Errors

**Model Not Found:**
```json
{
  "error": "model 'unknown-model' not found"
}
```

**Solution:** Pull the model first or check available models

**Out of Memory:**
```json
{
  "error": "failed to allocate memory"
}
```

**Solution:** Use smaller model or increase available memory

**Connection Refused:**
```
curl: (7) Failed to connect to localhost port 11434
```

**Solution:** Verify service is running and port is correct

### Retry Logic

```python
import time
import requests

def generate_with_retry(model, prompt, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.post(
                'http://localhost:11434/api/generate',
                json={'model': model, 'prompt': prompt, 'stream': False},
                timeout=30
            )
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Monitoring API Usage

### Request Logging

```bash
# Log all API requests
tail -f /var/log/nginx/access.log | grep ollama

# Or with Docker logs
docker logs -f $(docker ps -qf "ancestor=ollama/ollama:latest") | grep POST
```

### Performance Metrics

```bash
# Measure response time
time curl -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt": "test",
  "stream": false
}'

# Detailed timing
curl -w "@curl-format.txt" -X POST http://localhost:11434/api/generate -d '{
  "model": "llama2",
  "prompt": "test",
  "stream": false
}'
```

**curl-format.txt:**
```
time_namelookup:  %{time_namelookup}\n
time_connect:  %{time_connect}\n
time_appconnect:  %{time_appconnect}\n
time_pretransfer:  %{time_pretransfer}\n
time_redirect:  %{time_redirect}\n
time_starttransfer:  %{time_starttransfer}\n
time_total:  %{time_total}\n
```

## Best Practices

1. **Connection Management:**
   - Use connection pooling for multiple requests
   - Implement proper timeout handling
   - Close connections properly

2. **Error Handling:**
   - Implement retry logic with exponential backoff
   - Handle all error responses
   - Log errors for debugging

3. **Performance:**
   - Use streaming for long responses
   - Batch requests when possible
   - Cache responses when appropriate

4. **Security:**
   - Use network-level security
   - Implement authentication proxy if needed
   - Monitor for unauthorized access

5. **Resource Management:**
   - Limit concurrent requests
   - Monitor API response times
   - Track resource usage

## Next Steps

- Review [Model Management](model-management.md) for managing available models
- See [Monitoring](monitoring.md) for tracking API performance
- Check [Service Updates](service-updates.md) for maintaining the service

## Additional Resources

- [Ollama API Documentation](https://github.com/ollama/ollama/blob/main/docs/api.md)
- [REST API Best Practices](https://restfulapi.net/)
- [HTTP Client Libraries](https://github.com/topics/http-client)
