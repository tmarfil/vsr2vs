Kustomize can automatically update a VirtualServer manifest when new VirtualServerRoutes are created, focusing on routes that are one level deep from the root path.

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
└── kustomization.yaml
```

## Implementation Steps

1. **Base VirtualServer Template** (without routes):

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

2. **Create a Routes Generator Script** (`generate-routes.sh`):

```bash
#!/bin/bash

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

3. **Create Base Kustomization File**:

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- virtualserver.yaml
```

4. **Create Main Kustomization File** with Command to Generate Routes:

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- base
- routes/login-route.yaml
- routes/logout-route.yaml
- routes/profile-route.yaml
# Any new route files would be added here

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

1. **Create the VirtualServerRoute File**:

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

2. **Add it to the resources in kustomization.yaml**:

```yaml
resources:
- base
- routes/login-route.yaml
- routes/logout-route.yaml
- routes/profile-route.yaml
- routes/account-route.yaml  # New route added here
```

3. **Build and Apply**:

```bash
chmod +x generate-routes.sh
./generate-routes.sh  # Generates the routes configuration
kustomize build . | kubectl apply -f -
```

