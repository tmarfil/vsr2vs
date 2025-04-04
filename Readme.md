# NGINX Ingress VirtualServer Routes with Kustomize

This repository demonstrates a Kustomize-based approach for managing NGINX Ingress VirtualServer and VirtualServerRoute resources, particularly useful when dealing with a large number of routes.

## Overview

Managing hundreds or thousands of routes within a single Kubernetes VirtualServer manifest can become complex and error-prone. This solution addresses this by:

1.  Separating the base VirtualServer configuration from individual route definitions (VirtualServerRoutes).
2.  Using Kustomize to include all VirtualServerRoute manifests automatically.
3.  Providing a helper script to generate a Kustomize patch file, dynamically adding route references to the main VirtualServer based on the discovered VirtualServerRoute files.

## Directory Structure

```
├── base/
│   ├── kustomization.yaml
│   └── virtualserver.yaml # Base VS definition
├── routes/
│   ├── login-route.yaml # Example VSR
│   ├── logout-route.yaml # Example VSR
│   ├── profile-route.yaml # Example VSR
│   └── (any new route files).yaml # Add new VSRs here
├── generate-routes-patch.sh # Script to generate the patch
├── kustomization.yaml # Main Kustomize file
└── routes-patch.yaml # Generated patch file (add to .gitignore)
```

## Setup Instructions

### 1. Create the Base VirtualServer

Define your main VirtualServer without the dynamically generated routes. Include any default routes (like `/`) here.

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
  # Define base/default routes here
  routes:
    - path: /
      action:
        pass: main-app
    # Dynamic routes referencing VirtualServerRoutes will be added via patch
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
IGNORE_WHEN_COPYING_END
2. Create Base Kustomization File

This file simply includes the base VirtualServer manifest.

# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- virtualserver.yaml
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Yaml
IGNORE_WHEN_COPYING_END
3. Create the Routes Patch Generator Script

This script finds all *-route.yaml files in the routes/ directory and generates a Kustomize Strategic Merge Patch file (routes-patch.yaml) containing the necessary route entries for the main VirtualServer.

#!/bin/bash
# generate-routes-patch.sh

# Output patch file
output_file="routes-patch.yaml"
# Namespace where VirtualServerRoutes reside (adjust if needed)
NAMESPACE="app"
# Directory containing route manifests
ROUTES_DIR="routes"

# Start with the patch header targeting the main VirtualServer
cat > $output_file << EOF
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: main-application
  namespace: ${NAMESPACE}
spec:
  routes:
EOF

# Find all VirtualServerRoute files, extract info, and append to patch
# Using find + while read is safer for filenames with spaces/special chars
find "${ROUTES_DIR}" -maxdepth 1 -name '*-route.yaml' -print0 | while IFS= read -r -d $'\0' route_file; do
  if [[ -f "$route_file" ]]; then
    # Extract the resource name (e.g., "login-route") from the filename
    # Assumes filename convention like 'name-route.yaml'
    resource_name=$(basename "$route_file" .yaml)
    # Extract the path prefix (e.g., "login") from the resource name
    # Assumes resource name convention like 'prefix-route'
    path_prefix=$(echo "$resource_name" | sed 's/-route$//')

    # Add the route entry to the patch file
    # This assumes the path in the main VS corresponds to the VSR prefix
    cat >> $output_file << EOF
    # Route added from ${route_file}
    - path: /${path_prefix}
      route: ${NAMESPACE}/${resource_name}
EOF
  fi
done

echo "Generated ${output_file} successfully."
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Note: Make the script executable: chmod +x generate-routes-patch.sh

4. Create Main Kustomization File

This file brings together the base configuration, all individual VirtualServerRoute resources, and applies the generated patch.

# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  # Include the base VirtualServer definition
  - base
  # Automatically include all VirtualServerRoute manifests from the routes/ directory
  - routes/

patchesStrategicMerge:
  # Apply the generated patch file containing route references
  - routes-patch.yaml
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Yaml
IGNORE_WHEN_COPYING_END
5. Git Ignore

Add the generated patch file to your .gitignore to avoid committing it to version control, as it's dynamically created.

# .gitignore
routes-patch.yaml
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
IGNORE_WHEN_COPYING_END
Workflow for Adding a New Route

Create a new VirtualServerRoute file in the routes/ directory. Ensure the filename follows the name-route.yaml convention (e.g., account-route.yaml). The metadata.name inside the file should match the filename stem (e.g., account-route).

# routes/account-route.yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServerRoute
metadata:
  name: account-route # Should match filename stem
  namespace: app      # Should match NAMESPACE in script and VS
spec:
  # host: myapp.example.com # Optional if inherited from VS
  upstreams:
    - name: account-service
      service: account-svc
      port: 80
  subroutes:
    # The path here defines the specific endpoint(s) handled by this VSR
    # The path in the main VS (e.g., /account) acts as the prefix
    - path: /details # Full path becomes /account/details
      action:
        pass: account-service
    - path: /settings # Full path becomes /account/settings
      action:
        pass: account-service
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Yaml
IGNORE_WHEN_COPYING_END

Run the generator script to update the routes-patch.yaml file:

./generate-routes-patch.sh
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

(Optional) Verify the changes using Kustomize build:

kustomize build .
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Inspect the output to ensure the new route entry referencing app/account-route (or similar) has been added to the main VirtualServer's spec.routes list.

Apply the changes to your cluster:

kustomize build . | kubectl apply -f -
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Any new VirtualServerRoute file added to the routes/ directory will be included in the build. Just remember to run the generate-routes-patch.sh script before building and applying to update the route references in the main VirtualServer.

Notes

The generator script assumes a naming convention where the VirtualServerRoute filename is prefix-route.yaml, the resource name inside the file is prefix-route, and the desired path prefix in the main VirtualServer is /prefix. Adjust the script's logic (path_prefix extraction) if your convention differs.

Ensure the namespace: specified in the VirtualServer, VirtualServerRoutes, and the NAMESPACE variable in the script are consistent.

This approach relies on a manual script execution step before kustomize build. For fully automated GitOps workflows, consider alternative approaches like custom KRM Functions executed via kustomize fn run, which can perform similar logic directly within the Kustomize build process without requiring external script execution.
