# NGINX Ingress VirtualServer Routes with Kustomize

This repository contains a Kustomize-based solution for managing NGINX Ingress VirtualServer and VirtualServerRoute resources at scale.

## Overview

When managing hundreds or thousands of routes in a Kubernetes NGINX Ingress environment, maintaining a single manifest becomes unwieldy. This solution:

1. Separates your base VirtualServer configuration from routes
2. Automatically detects and incorporates new VirtualServerRoute manifests
3. Dynamically generates route entries in the main VirtualServer

## Directory Structure

```
├── base/
│   ├── kustomization.yaml
│   └── virtualserver.yaml
├── routes/
│   ├── login-route.yaml
│   ├── logout-route.yaml
│   ├── profile-route.yaml
│   └── (any new route files)
├── generate-routes.sh
└── kustomization.yaml
```

## Setup Instructions

### 1. Create the Base VirtualServer Template

```yaml
# base/virtualserver.yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: main-application
  namespace: app
spec:
  host: myapp.example.com
  tls:
    secret: myapp-tls
    redirect:
      enable: true
  upstreams:
    - name: main-app
      service: main-app-svc
      port: 80
      lb-method: round_robin
      max-conns: 32
      connect-timeout: 30s
      read-timeout: 30s
      send-timeout: 30s
  routes:
    - path: /
      action:
        pass: main-app
```

### 2. Create Base Kustomization File

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- virtualserver.yaml
```

### 3. Create the Routes Generator Script

```bash
#!/bin/bash
# generate-routes.sh

# Output file to be dynamically created
output_file="generated-routes.yaml"

# Start with YAML header
cat > $output_file << EOF
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: main-application
  namespace: app
spec:
  routes:
    - path: /
      action:
        pass: main-app
EOF

# Find all VirtualServerRoute files and extract their paths
for route_file in routes/*-route.yaml; do
  # Extract the route name (without -route.yaml)
  route_name=$(basename $route_file -route.yaml)
  
  # Add to the routes list
  cat >> $output_file << EOF
    - path: /$route_name
      route: app/$route_name-route
EOF
done
```

### 4. Create Main Kustomization File

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- base
# Wildcard pattern to automatically include all route files
- routes/*-route.yaml

transformers:
- |-
  apiVersion: builtin
  kind: PatchTransformer
  metadata:
    name: add-routes
  patch: |-
    $patch: merge
    spec:
      routes: $(ROUTES)
  target:
    kind: VirtualServer
    name: main-application

configMapGenerator:
- name: routes-generator
  files:
  - generate-routes.sh
  options:
    disableNameSuffixHash: true

vars:
- name: ROUTES
  objref:
    kind: ConfigMap
    name: routes-generator
    apiVersion: v1
  fieldref:
    fieldpath: data["generate-routes.sh"]
```

## Workflow for Adding a New Route

1. **Create a new VirtualServerRoute file** in the `routes/` directory, following the naming convention `name-route.yaml`:

```yaml
# routes/account-route.yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServerRoute
metadata:
  name: account-route
  namespace: app
spec:
  host: myapp.example.com
  upstreams:
    - name: account-service
      service: account-svc
      port: 80
  subroutes:
    - path: /account
      action:
        pass: account-service
```

2. **Run the generator script** to update the routes configuration:

```bash
chmod +x generate-routes.sh
./generate-routes.sh
```

3. **Apply the changes** to your cluster:

```bash
kustomize build . | kubectl apply -f -
```

That's it! Any new VirtualServerRoute file you add to the routes directory will be automatically included in your build without any manual updates to configuration files.

## Notes

- All routes are assumed to be one level deep from root (e.g., `/login`, `/profile`)
- Make sure your VirtualServerRoute file names match the path pattern
- The generator script can be extended for more complex path structures if needed
