# Deployment Init Container Injector

A Kyverno policy that automatically injects init containers into Kubernetes Deployments based on annotations.

## Overview

This policy watches for Deployments with specific annotations and automatically injects an init container with configurable properties like image, registry, commands and arguments.

## Usage

### Prerequisites

- Kubernetes cluster
- Kyverno installed in the cluster

### Installation

Apply the policy to your cluster:

```bash
kubectl apply -f pp.yaml
```

### Configuration

Add the following annotations to your Deployment to trigger the init container injection:

- `initcontainer_injector_args` (required): Comma-separated list of arguments for the init container
- `initcontainer_injector_image` (optional): Container image name (defaults to "default")
- `initcontainer_injector_registry` (optional): Container registry (defaults to "docker.io") 
- `initcontainer_injector_command` (optional): Comma-separated command (defaults to "/bin/sh,-c,echo")

### Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    initcontainer_injector_args: "arg1,arg2,arg3"
    initcontainer_injector_image: "busybox"
    initcontainer_injector_registry: "docker.io"
    initcontainer_injector_command: "/bin/sh,-c,echo"
spec:
  # ... rest of deployment spec
```

This will inject an init container with:
- Image: docker.io/busybox:latest
- Command: ["/bin/sh", "-c", "echo"]
- Args: ["arg1", "arg2", "arg3"]

## How It Works

The policy uses Kyverno's mutation capabilities to:

1. Match Deployments with required annotations
2. Inject an init container with specified configuration
3. Split comma-separated strings into arrays for commands and arguments
4. Use default values when optional annotations are not provided

