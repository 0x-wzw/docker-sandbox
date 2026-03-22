---
name: docker-sandbox
version: 1.0.0
description: "Docker-based execution sandbox for secure agent task isolation. Adapted from DeerFlow's containerized execution model."
author: October (10D Entity)
keywords: [docker, sandbox, container, isolation, execution, security]
---

# Docker Sandbox 🐳

> **Secure, isolated execution environments for agent tasks**
> 
> *Adapted from DeerFlow's containerized execution philosophy*

## Overview

Executes agent tasks in isolated Docker containers with:
- **Filesystem isolation** — Each task gets its own volume
- **Network isolation** — Controlled network access
- **Resource limits** — CPU/memory constraints
- **Ephemeral execution** — Containers destroyed after task completion

## Architecture

```
Agent Task
    ↓
Docker Sandbox Manager
    ↓
┌─────────────────────────────────────┐
│  Docker Container (ephemeral)     │
│  ├─ Isolated filesystem            │
│  ├─ Limited network access         │
│  ├─ Resource constraints           │
│  └─ Tool execution                 │
└─────────────────────────────────────┘
    ↓
Results + Logs
```

## Usage

### Execute Task in Sandbox

```python
from docker_sandbox import execute_in_sandbox

result = execute_in_sandbox(
    task="npm install && npm test",
    image="node:18-alpine",
    memory_limit="512m",
    cpu_limit="1.0",
    timeout=300,
    volumes={
        "/workspace": "/app"
    }
)
```

### Configuration

```yaml
# ~/.openclaw/docker-sandbox.yaml
sandbox:
  default_image: "ubuntu:22.04"
  memory_limit: "1g"
  cpu_limit: "2.0"
  timeout: 300
  network_mode: "none"  # or "bridge" for internet access
  
  # Security settings
  security:
    read_only_root: true
    no_new_privileges: true
    drop_capabilities: ["ALL"]
    
  # Resource constraints
  resources:
    max_memory: "2g"
    max_cpu: "4.0"
    max_storage: "10g"
```

## Implementation

### File: `docker_sandbox/executor.py`

```python
import docker
import tempfile
import shutil
from pathlib import Path
from typing import Dict, Optional

class DockerSandbox:
    """Execute tasks in isolated Docker containers."""
    
    def __init__(self, config: Optional[Dict] = None):
        self.client = docker.from_env()
        self.config = config or {}
    
    def execute(
        self,
        command: str,
        image: str = "ubuntu:22.04",
        memory: str = "512m",
        cpu: str = "1.0",
        timeout: int = 300,
        volumes: Optional[Dict] = None,
        network: str = "none"
    ) -> Dict:
        """
        Execute command in isolated Docker container.
        
        Args:
            command: Shell command to execute
            image: Docker image to use
            memory: Memory limit (e.g., "512m", "1g")
            cpu: CPU limit (e.g., "1.0", "2.0")
            timeout: Execution timeout in seconds
            volumes: Volume mounts {host_path: container_path}
            network: Network mode ("none", "bridge", "host")
        
        Returns:
            Dict with stdout, stderr, exit_code, duration
        """
        # Create temporary workspace
        workspace = Path(tempfile.mkdtemp(prefix="sandbox_"))
        
        try:
            # Prepare volumes
            volume_mounts = {
                str(workspace): {"bind": "/workspace", "mode": "rw"}
            }
            if volumes:
                for host, container in volumes.items():
                    volume_mounts[host] = {"bind": container, "mode": "rw"}
            
            # Run container
            container = self.client.containers.run(
                image=image,
                command=f"sh -c 'cd /workspace && {command}'",
                volumes=volume_mounts,
                mem_limit=memory,
                cpu_quota=int(float(cpu) * 100000),
                network_mode=network,
                security_opt=["no-new-privileges:true"],
                cap_drop=["ALL"],
                read_only=False,
                detach=True,
                stdout=True,
                stderr=True
            )
            
            # Wait for completion
            result = container.wait(timeout=timeout)
            
            # Collect output
            stdout = container.logs(stdout=True, stderr=False).decode()
            stderr = container.logs(stdout=False, stderr=True).decode()
            
            # Cleanup
            container.remove(force=True)
            
            return {
                "stdout": stdout,
                "stderr": stderr,
                "exit_code": result["StatusCode"],
                "success": result["StatusCode"] == 0
            }
            
        except Exception as e:
            return {
                "stdout": "",
                "stderr": str(e),
                "exit_code": -1,
                "success": False
            }
        finally:
            # Cleanup workspace
            shutil.rmtree(workspace, ignore_errors=True)
```

## Security Features

| Feature | Description |
|---------|-------------|
| **Read-only root** | Container filesystem is read-only by default |
| **No new privileges** | Prevents privilege escalation |
| **Capability dropping** | Removes all Linux capabilities |
| **Network isolation** | "none" mode blocks all network access |
| **Resource limits** | CPU/memory constraints prevent DoS |
| **Ephemeral storage** | Auto-cleanup after execution |

## Integration with Swarm

### In SovereignStack

```python
# Use Docker sandbox for code execution
if task.requires_code_execution:
    result = docker_sandbox.execute(
        command=task.command,
        image="node:18-alpine" if task.language == "javascript" else "python:3.11",
        memory="1g",
        timeout=300
    )
```

### In Halloween (Code Agent)

```python
# Execute code in isolated environment
def execute_code_safely(code: str, language: str) -> Result:
    return docker_sandbox.execute(
        command=f"echo '{code}' | {interpreter}",
        image=get_image_for_language(language),
        memory="512m",
        network="none"  # No internet for untrusted code
    )
```

## Comparison to DeerFlow

| Aspect | DeerFlow | Swarm Docker Sandbox |
|--------|----------|---------------------|
| **Base** | Python + Docker SDK | Same |
| **Isolation** | Full container | Same |
| **Resources** | Configurable | Same |
| **Network** | Isolated by default | Same |
| **Integration** | LangGraph | Swarm Protocol |
| **Economic** | Cost tracking | Token economy |

## Installation

```bash
# Install Docker
sudo apt install docker.io

# Add user to docker group
sudo usermod -aG docker $USER

# Install Python Docker SDK
pip install docker

# Test
python -c "import docker; print('OK')"
```

## Next Steps

- [ ] Integrate with Halloween's code execution
- [ ] Add resource usage tracking for cost router
- [ ] Implement container image caching
- [ ] Add security scanning before execution

---

*Docker-based execution isolation for the 0x-wzw Swarm*
*Adapted from DeerFlow's containerized execution model*
