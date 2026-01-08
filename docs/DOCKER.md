# Docker Usage Guide

This guide explains how to use Trae Agent with Docker for safe, isolated task execution. Docker support is particularly useful for running potentially destructive operations, working with different environments, or ensuring reproducible builds.

## Table of Contents

- [Why Use Docker?](#why-use-docker)
- [Prerequisites](#prerequisites)
- [Docker Modes](#docker-modes)
- [Basic Usage](#basic-usage)
- [Advanced Usage](#advanced-usage)
- [How It Works](#how-it-works)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

## Why Use Docker?

Docker integration provides several benefits:

### Safety
- **Isolation**: Code runs in a container, protecting your host system
- **Sandboxing**: File system access is limited to mounted directories
- **Rollback**: Containers can be easily discarded if things go wrong

### Reproducibility
- **Consistent Environment**: Same behavior across different machines
- **Dependency Management**: All dependencies packaged in the container
- **Version Control**: Pin specific versions of tools and libraries

### Flexibility
- **Multiple Environments**: Test with different Python versions, OS, etc.
- **Clean Slate**: Start fresh for each task
- **Parallel Execution**: Run multiple tasks in different containers

## Prerequisites

### 1. Install Docker

**Linux:**
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```

**macOS:**
```bash
# Install Docker Desktop from:
# https://www.docker.com/products/docker-desktop
```

**Windows:**
```bash
# Install Docker Desktop from:
# https://www.docker.com/products/docker-desktop
```

### 2. Verify Installation

```bash
docker --version
# Should output: Docker version 24.0.0 or higher

docker ps
# Should show running containers (or empty list)
```

### 3. Test Docker

```bash
docker run hello-world
# Should download and run the hello-world container
```

## Docker Modes

Trae Agent supports four Docker modes:

### 1. Use Existing Image

Pull and run a task in a pre-built Docker image.

```bash
trae-cli run "your task" --docker-image python:3.12
```

**Use Cases:**
- Standard Python/Node.js/etc. development
- No custom setup needed
- Quick prototyping

### 2. Build from Dockerfile

Build a custom image from a Dockerfile and run the task.

```bash
trae-cli run "your task" --dockerfile-path /path/to/Dockerfile
```

**Use Cases:**
- Custom environment setup
- Specific dependencies required
- Production-like environment

### 3. Load from Image File

Load an image from a tar archive and run the task.

```bash
trae-cli run "your task" --docker-image-file /path/to/image.tar
```

**Use Cases:**
- Pre-built custom images
- Offline environments
- Sharing environments

### 4. Attach to Running Container

Attach to an existing running container.

```bash
trae-cli run "your task" --docker-container-id abc123def
```

**Use Cases:**
- Long-running development containers
- Persistent environments
- Multiple tasks in same environment

## Basic Usage

### Example 1: Python Development

Run a Python task in a Python 3.12 container:

```bash
# Create a test directory
mkdir ~/my-python-project
cd ~/my-python-project

# Create a simple Python file
echo "print('Hello World')" > hello.py

# Run a task in Python 3.12 container
trae-cli run "Add a function to greet a user by name" --docker-image python:3.12
```

**What happens:**
1. Docker pulls `python:3.12` image (if not cached)
2. Creates a new container
3. Mounts current directory into container
4. Runs the task inside the container
5. Changes are reflected in your local directory
6. Container is removed (by default)

### Example 2: Node.js Development

```bash
mkdir ~/my-node-project
cd ~/my-node-project

# Initialize a Node.js project inside Docker
trae-cli run "Create a Node.js Express server with a /hello endpoint" \
  --docker-image node:20
```

### Example 3: Specific Working Directory

Run a task in a specific directory:

```bash
trae-cli run "Add tests for the auth module" \
  --docker-image python:3.12 \
  --working-dir ~/projects/myapp
```

The `--working-dir` is mounted into the container at `/workspace`.

### Example 4: Keep Container Running

By default, containers are removed after execution. To keep them:

```bash
trae-cli run "Setup the development environment" \
  --docker-image python:3.12 \
  --docker-keep true
```

## Advanced Usage

### Using Custom Dockerfiles

Create a Dockerfile with your project's dependencies:

```dockerfile
# Dockerfile
FROM python:3.12-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install Python packages
RUN pip install --no-cache-dir \
    pytest \
    black \
    mypy \
    requests

# Set working directory
WORKDIR /workspace
```

Run tasks with this Dockerfile:

```bash
trae-cli run "Run the test suite" --dockerfile-path ./Dockerfile
```

### Multi-Stage Dockerfile

For more complex setups:

```dockerfile
# Dockerfile.dev
FROM python:3.12-slim AS base

RUN apt-get update && apt-get install -y git

FROM base AS development

COPY requirements.txt .
RUN pip install -r requirements.txt

WORKDIR /workspace
```

### Using Docker Compose

While Trae Agent doesn't directly support Docker Compose, you can:

1. Start services with Docker Compose
2. Attach Trae Agent to a specific container

```bash
# Start services
docker-compose up -d

# Get container ID
CONTAINER_ID=$(docker-compose ps -q app)

# Run task in that container
trae-cli run "Run integration tests" --docker-container-id $CONTAINER_ID
```

### Environment-Specific Images

Create different images for different tasks:

```bash
# Use Python 3.11 for legacy code
trae-cli run "Fix bug in legacy module" --docker-image python:3.11

# Use Python 3.12 for new features
trae-cli run "Add new feature" --docker-image python:3.12

# Use Alpine for minimal footprint
trae-cli run "Quick script" --docker-image python:3.12-alpine
```

### Saving and Loading Images

Save a configured image for reuse:

```bash
# Build and run a task
trae-cli run "Setup environment" --dockerfile-path ./Dockerfile

# Find the container (while still running with --docker-keep true)
docker ps

# Commit the container to an image
docker commit <container-id> my-project:v1

# Save to tar file
docker save my-project:v1 > my-project-v1.tar

# Use the tar file later
trae-cli run "Run tests" --docker-image-file my-project-v1.tar
```

### Running Tests in Isolation

Ensure tests run in a clean environment:

```bash
# Each test run gets a fresh container
trae-cli run "Run unit tests" --docker-image python:3.12
trae-cli run "Run integration tests" --docker-image python:3.12
trae-cli run "Run e2e tests" --docker-image python:3.12
```

### Cross-Platform Development

Test on different platforms:

```bash
# Linux
trae-cli run "Test on Linux" --docker-image python:3.12

# Alpine Linux (different libc)
trae-cli run "Test on Alpine" --docker-image python:3.12-alpine

# Different architectures (if Docker supports)
trae-cli run "Test on ARM" --docker-image python:3.12 --platform linux/arm64
```

## How It Works

### Architecture

```
Host Machine                      Docker Container
┌─────────────────────┐          ┌─────────────────────┐
│  Trae Agent         │          │  Container          │
│                     │          │                     │
│  ┌───────────────┐  │          │  ┌───────────────┐  │
│  │ Agent Logic   │──┼──────────┼─▶│ Tools         │  │
│  └───────────────┘  │          │  │ - bash        │  │
│                     │          │  │ - edit_tool   │  │
│  ┌───────────────┐  │          │  │ - json_edit   │  │
│  │ LLM Client    │  │          │  └───────────────┘  │
│  └───────────────┘  │          │                     │
│                     │          │  /workspace         │
│  Working Directory  │◀─────────┼─ (mounted)         │
│  /host/path        │  Mount   │                     │
└─────────────────────┘          └─────────────────────┘
```

### Path Translation

Trae Agent automatically translates paths between host and container:

```python
# Your working directory
Host: /home/user/myproject

# Is mounted in container as
Container: /workspace

# Path translations:
Host path: /home/user/myproject/src/main.py
Container path: /workspace/src/main.py
```

The agent handles this translation automatically for all tools.

### Tool Execution Flow

```
1. Agent receives task
   │
2. Agent calls bash tool with command
   │
3. Docker Tool Executor:
   ├─ Translates host paths to container paths
   ├─ Sends command to container via docker exec
   └─ Returns result to agent
   │
4. Agent processes result and continues
```

### Binary Tools

Some tools are pre-compiled and copied into the container:

```
trae_agent/dist/
├── edit_tool              # File editing binary
├── json_edit_tool         # JSON editing binary
└── _internal/            # Dependencies
```

These are built once (first time you use Docker mode) and reused.

## Best Practices

### 1. Use Appropriate Images

✓ **Good:**
```bash
# Use official, specific tags
trae-cli run "task" --docker-image python:3.12-slim
```

✗ **Bad:**
```bash
# Avoid 'latest' tag (unpredictable)
trae-cli run "task" --docker-image python:latest
```

### 2. Keep Images Small

Use slim or alpine variants when possible:

```dockerfile
# Good: Slim image
FROM python:3.12-slim

# Better: Alpine (even smaller)
FROM python:3.12-alpine
```

### 3. Cache Dependencies

In Dockerfiles, copy requirements first:

```dockerfile
# Copy requirements first (cached if unchanged)
COPY requirements.txt .
RUN pip install -r requirements.txt

# Copy code later (changes more frequently)
COPY . .
```

### 4. Clean Up Resources

```bash
# Remove stopped containers
docker container prune

# Remove unused images
docker image prune

# Remove everything unused
docker system prune -a
```

### 5. Use .dockerignore

Create a `.dockerignore` file:

```
# .dockerignore
.git
.venv
__pycache__
*.pyc
.pytest_cache
node_modules
.env
trajectories
```

### 6. Mount Only What's Needed

By default, the working directory is mounted. Be careful with:

```bash
# This mounts your entire home directory!
trae-cli run "task" --working-dir ~ --docker-image python:3.12
```

Prefer specific project directories.

### 7. Version Lock Images

Use specific versions for reproducibility:

```yaml
# trae_config.yaml
# Document the image version used
# docker_image: python:3.12.0-slim
```

## Troubleshooting

### Issue: "Docker daemon not running"

**Symptoms:**
```
Error: Cannot connect to the Docker daemon
```

**Solution:**
```bash
# Linux: Start Docker service
sudo systemctl start docker

# macOS/Windows: Start Docker Desktop
# Open Docker Desktop application
```

### Issue: "Permission denied"

**Symptoms:**
```
Error: Got permission denied while trying to connect to the Docker daemon socket
```

**Solution:**
```bash
# Linux: Add user to docker group
sudo usermod -aG docker $USER

# Log out and back in, or run:
newgrp docker

# Test:
docker ps
```

### Issue: "Cannot build from Dockerfile"

**Symptoms:**
```
Error building Docker image from Dockerfile
```

**Solutions:**
```bash
# 1. Check Dockerfile syntax
docker build -f /path/to/Dockerfile .

# 2. Use absolute path
trae-cli run "task" --dockerfile-path $(pwd)/Dockerfile

# 3. Check Docker buildx is available
docker buildx version
```

### Issue: "Tools not working in container"

**Symptoms:**
```
bash: edit_tool: command not found
```

**Solution:**
```bash
# Rebuild tools
rm -rf trae_agent/dist
trae-cli run "task" --docker-image python:3.12
# This will rebuild tools automatically
```

### Issue: "Changes not reflected on host"

**Symptoms:**
Files created in container don't appear on host

**Solution:**
```bash
# Ensure working directory is specified
trae-cli run "task" --working-dir $(pwd) --docker-image python:3.12

# Check mount is correct
docker inspect <container-id> | grep Mounts -A 10
```

### Issue: "Container keeps running"

**Symptoms:**
Containers not cleaned up after execution

**Solution:**
```bash
# List containers
docker ps -a

# Remove specific container
docker rm <container-id>

# Remove all stopped containers
docker container prune

# Or use --docker-keep false explicitly
trae-cli run "task" --docker-image python:3.12 --docker-keep false
```

### Issue: "Slow performance"

**Symptoms:**
Docker operations are very slow

**Solutions:**
```bash
# 1. Check Docker resource allocation
# Docker Desktop: Preferences → Resources
# Increase CPU/Memory if needed

# 2. Use volume mounts instead of bind mounts
# (Advanced: modify Docker configuration)

# 3. Use local registry for images
docker pull python:3.12
# Now it's cached locally
```

### Issue: "Out of disk space"

**Symptoms:**
```
Error: No space left on device
```

**Solution:**
```bash
# Clean up Docker resources
docker system df  # Check usage

docker system prune -a  # Remove all unused

# Remove specific images
docker images
docker rmi <image-id>
```

## Example Workflows

### Workflow 1: Python Package Development

```bash
# 1. Create Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.12-slim
RUN pip install pytest pytest-cov black mypy
WORKDIR /workspace
EOF

# 2. Develop feature
trae-cli run "Add new feature to utils.py" --dockerfile-path ./Dockerfile

# 3. Run tests
trae-cli run "Run pytest with coverage" --dockerfile-path ./Dockerfile

# 4. Format code
trae-cli run "Format all Python files with black" --dockerfile-path ./Dockerfile
```

### Workflow 2: Multi-Version Testing

```bash
# Test on Python 3.10
trae-cli run "Run tests" --docker-image python:3.10

# Test on Python 3.11
trae-cli run "Run tests" --docker-image python:3.11

# Test on Python 3.12
trae-cli run "Run tests" --docker-image python:3.12
```

### Workflow 3: Isolated Bug Reproduction

```bash
# Create clean environment
trae-cli run "Install dependencies and reproduce bug #123" \
  --docker-image python:3.12 \
  --trajectory-file bug-123-repro.json

# Container is removed after, ensuring clean state
```

## Performance Considerations

### Image Size vs. Startup Time

```
| Image              | Size    | Startup | Use Case            |
|--------------------|---------|---------|---------------------|
| python:3.12        | 1.02 GB | Slow    | Full development    |
| python:3.12-slim   | 130 MB  | Medium  | Most tasks          |
| python:3.12-alpine | 49 MB   | Fast    | Minimal tasks       |
```

**Recommendation:** Use `slim` variants for most tasks.

### Build Time Optimization

```dockerfile
# Optimize layer caching
FROM python:3.12-slim

# Layer 1: System packages (rarely changes)
RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*

# Layer 2: Python packages (changes occasionally)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Layer 3: Application code (changes frequently)
COPY . .
```

Each layer is cached independently.

### Volume Performance

Bind mounts can be slow on some systems. For better performance:

```bash
# Use named volumes (advanced)
docker volume create myproject-data
# Then configure Trae Agent to use it
```

## Security Considerations

### 1. Limit Container Capabilities

Docker containers run with limited privileges by default. Keep it that way.

### 2. Don't Mount Sensitive Directories

Avoid mounting:
- `~/.ssh` (SSH keys)
- `~/.aws` (AWS credentials)
- `~/.config` (System configuration)

### 3. Use Read-Only Mounts

For reference data:
```bash
# Mount as read-only
docker run -v $(pwd)/data:/data:ro python:3.12
```

### 4. Scan Images for Vulnerabilities

```bash
docker scan python:3.12-slim
```

### 5. Use Official Images

Always prefer official images from Docker Hub:
- `python` (official Python)
- `node` (official Node.js)
- `ruby` (official Ruby)

## Related Documentation

- [Getting Started](GETTING_STARTED.md)
- [Configuration Guide](CONFIGURATION.md)
- [Architecture](ARCHITECTURE.md)
- [Workflow Guide](WORKFLOW.md)

## Summary

Docker integration in Trae Agent provides:

✓ **Safety** through isolation  
✓ **Reproducibility** across environments  
✓ **Flexibility** for different setups  

**Quick Start:**
```bash
trae-cli run "your task" --docker-image python:3.12
```

**Best Practice:**
Use slim images, specify versions, and clean up resources regularly.

For most development tasks, `python:3.12-slim` or `node:20-slim` provides the best balance of features, size, and performance.
