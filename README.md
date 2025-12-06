# Deployment Init Container Injector

A Kyverno policy that automatically injects init containers into Kubernetes Deployments based on annotations.

## Overview

This policy watches for `Deployments` with specific annotations and automatically injects an init container. It handles complex logic such as setting default values for registries and images, and splitting comma-separated strings into command and argument arrays.

## Repository Contents

  * `deployment-initcontainer-injector.yaml`: The Kyverno ClusterPolicy definition.
  * `deployment-with-init-inject-annotation.yaml`: A sample Deployment manifest demonstrating usage.

## Usage

### Prerequisites

  * Kubernetes cluster
  * Kyverno installed in the cluster (v1.6+)

### Installation

Apply the policy manifest to your cluster:

```bash
kubectl apply -f deployment-initcontainer-injector.yaml
```

### Configuration

Add the following annotations to your Deployment metadata to trigger the injection.

| Annotation Key | Required | Default | Description |
| :--- | :---: | :--- | :--- |
| `initcontainer_injector_args` | **Yes** | N/A | Comma-separated list of arguments (e.g., `arg1,arg2`). |
| `initcontainer_injector_image` | No | `default` | The container image name. |
| `initcontainer_injector_registry` | No | `docker.io` | The container registry URL. |
| `initcontainer_injector_command` | No | `/bin/sh,-c,echo` | Comma-separated command string. |

-----

## Policy Definition

Below is the source code for **`deployment-initcontainer-injector.yaml`**. It uses Kyverno variable substitution to parse annotations and apply defaults.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deployment-initcontainer-injector
spec:
  rules:
  - name: inject-init-container
    match:
      any:
      - resources:
          kinds:
          - Deployment
    # Only apply if the required args annotation is present
    preconditions:
      all:
      - key: "{{ request.object.metadata.annotations.initcontainer_injector_args || '' }}"
        operator: NotEquals
        value: ""
    mutate:
      foreach:
      - list: "request.object.metadata.annotations"
        patchStrategicMerge:
          spec:
            template:
              spec:
                initContainers:
                - name: injected-init-container
                  # Construct image from registry and image annotations (or defaults)
                  image: "{{ request.object.metadata.annotations.initcontainer_injector_registry || 'docker.io' }}/{{ request.object.metadata.annotations.initcontainer_injector_image || 'default' }}"
                  # Split comma-separated command string into array
                  command: "{{ request.object.metadata.annotations.initcontainer_injector_command || '/bin/sh,-c,echo' | split(@, ',') }}"
                  # Split comma-separated args string into array
                  args: "{{ request.object.metadata.annotations.initcontainer_injector_args | split(@, ',') }}"
```

-----

## Example Deployment

Below is the content of **`deployment-with-init-inject-annotation.yaml`**. This demonstrates how to annotate a standard Nginx deployment to trigger the injection.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
  annotations:
    # REQUIRED: Arguments for the init container
    initcontainer_injector_args: "arg1,arg2,arg3"
    # OPTIONAL: Overrides for image and command
    initcontainer_injector_image: "busybox"
    initcontainer_injector_registry: "docker.io"
    initcontainer_injector_command: "/bin/sh,-c,echo"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: nginx
        image: nginx:latest
```

### Result

After applying the example above, the resulting Pod will contain an init container configured as follows:

  * **Image:** `docker.io/busybox`
  * **Command:** `["/bin/sh", "-c", "echo"]`
  * **Args:** `["arg1", "arg2", "arg3"]`

## How It Works

The policy uses Kyverno's mutation capabilities to:

1.  **Match:** Identifies Deployments containing the annotation `initcontainer_injector_args`.
2.  **Process Variables:**
      * It checks for optional annotations (`image`, `registry`, `command`).
      * If missing, it assigns the defined hardcoded default values.
3.  **Transform:** It uses JMESPath filters (`split`) to convert the comma-separated annotation strings into JSON arrays required by the Kubernetes API.
4.  **Inject:** It patches the `spec.template.spec.initContainers` list with the new container definition.
