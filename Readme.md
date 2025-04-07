# NGINX Ingress VirtualServer Routes with Kustomize

This repository demonstrates a [Kustomize-based approach](https://kustomize.io/) for managing NGINX Ingress VirtualServer and VirtualServerRoute resources, particularly useful when dealing with a large number of routes.

## Overview

Managing hundreds or thousands of routes within a single Kubernetes VirtualServer manifest can become complex and error-prone. This solution addresses this by:

1.  Separating the base VirtualServer configuration from individual route definitions (VirtualServerRoutes).
2.  Using Kustomize to include all VirtualServerRoute manifests automatically.
3.  Providing a helper script to generate a Kustomize patch file, dynamically adding route references to the main VirtualServer based on the discovered VirtualServerRoute files.

## Directory Structure

```
.
├── base/
│   ├── kustomization.yaml
│   └── virtualserver.yaml         # Base VS definition
├── routes/
│   ├── login-route.yaml         # Example VSR
│   ├── logout-route.yaml        # Example VSR
│   ├── profile-route.yaml       # Example VSR
│   └── (any new route files).yaml # Add new VSRs here
├── generate-routes-patch.sh     # Script to generate the patch
├── kustomization.yaml           # Main Kustomize file
└── routes-patch.yaml            # Generated patch file (add to .gitignore)
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
```

### 2. Create Base Kustomization File

This file simply includes the base VirtualServer manifest.

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- virtualserver.yaml
```

### 3. Create the Routes Patch Generator Script

This script finds all `*-route.yaml` files in the `routes/` directory and generates a Kustomize Strategic Merge Patch file (`routes-patch.yaml`) containing the necessary `route` entries for the main VirtualServer.

```bash
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
```

Here's an example of what the `routes-patch.yaml` file might look like with six VirtualServerRoute entries:

```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: main-application
  namespace: app
spec:
  routes:
    - path: /login
      route: app/login-route
    - path: /dashboard
      route: app/dashboard-route
    - path: /profile
      route: app/profile-route
    - path: /settings
      route: app/settings-route
    - path: /api
      route: app/api-route
    - path: /account
      route: app/account-route
```

**Note:** Make the script executable: `chmod +x generate-routes-patch.sh`

### 4. Create Main Kustomization File

This file brings together the base configuration, all individual VirtualServerRoute resources, and applies the generated patch.

```yaml
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
```

### 5. Git Ignore

Add the generated patch file to your `.gitignore` to avoid committing it to version control, as it's dynamically created.

```
# .gitignore
routes-patch.yaml
```

## Workflow for Adding a New Route

1.  **Create a new VirtualServerRoute file** in the `routes/` directory. Ensure the filename follows the `name-route.yaml` convention (e.g., `account-route.yaml`). The `metadata.name` inside the file should match the filename stem (e.g., `account-route`).

    ```yaml
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
    ```

2.  **Run the generator script** to update the `routes-patch.yaml` file:

    ```bash
    ./generate-routes-patch.sh
    ```

3.  **(Optional) Verify the changes** using Kustomize build:

    ```bash
    kustomize build .
    ```
    
    `kustomize build .` is a dry-run operation that shows what would be applied to a cluster.
    No changes are made to your Kubernetes cluster with this command alone.
   
    Inspect the output to ensure the new route entry referencing `app/account-route` (or similar) has been added to the main VirtualServer's `spec.routes` list.

5.  **Apply the changes** to your cluster:

    ```bash
    kustomize build . | kubectl apply -f -
    ```

Any new VirtualServerRoute file added to the `routes/` directory will be included in the build. Just remember to run the `generate-routes-patch.sh` script before building and applying to update the route references in the main VirtualServer.


## Workflow for Removing an Existing Route

Let's trace the workflow when you remove, for example, `routes/profile-route.yaml`:

1.  **Delete the file:** You remove `routes/profile-route.yaml`.
2.  **Run the script:** You execute `./generate-routes-patch.sh`.
3.  **Patch Regeneration:** The script scans `routes/`. It finds `login-route.yaml` and `logout-route.yaml`, but *not* `profile-route.yaml`. It regenerates `routes-patch.yaml` containing *only* the routes for login and logout. The entry for `/profile` pointing to `app/profile-route` will **not** be included in the newly generated `routes-patch.yaml`.
4.  **Kustomize Build:** You run `kustomize build .`.
    *   Kustomize includes `base/virtualserver.yaml`.
    *   Kustomize includes the remaining resources from `routes/` (e.g., `login-route.yaml`, `logout-route.yaml`). It does *not* include the deleted `profile-route.yaml`.
    *   Kustomize applies the *newly generated* `routes-patch.yaml`. This patch overwrites the `spec.routes` in the base VirtualServer. Since the patch no longer contains the `/profile` route, the final VirtualServer manifest generated by Kustomize will *not* have the `/profile` route entry referencing `app/profile-route`.
5.  **Kubectl Apply:** You run `kustomize build . | kubectl apply -f -`.
    *   Kubernetes receives the desired state from Kustomize.
    *   It sees that the `VirtualServer` object `main-application` should no longer have the `/profile` route in its `spec.routes` list. It updates the `VirtualServer` resource accordingly.
    *   It sees that the `VirtualServerRoute` resource `profile-route` is no longer part of the desired state being applied. If `kubectl apply` is used with pruning (`--prune -l your-app-label`) or if the resource is managed declaratively (like via ArgoCD or Flux), the `profile-route` VSR object itself will likely be deleted from the cluster. If you just run `kubectl apply` without pruning, the VSR object might remain orphaned until explicitly deleted, but the reference to it *will* be removed from the parent `VirtualServer`.

**The current solution already handles the removal of paths and routes from the VirtualServer when a VirtualServerRoute YAML manifest is removed**, provided you follow the workflow:

1.  Remove the `*.yaml` file from the `routes/` directory.
2.  **Crucially, run the `./generate-routes-patch.sh` script** to update `routes-patch.yaml`.
3.  Run `kustomize build . | kubectl apply -f -` to apply the changes.

The key is that the script regenerates the patch based on the *current* contents of the `routes/` directory, ensuring that removed files no longer contribute their corresponding route entries to the main VirtualServer configuration.

## Notes

*   The generator script assumes a naming convention where the VirtualServerRoute filename is `prefix-route.yaml`, the resource name inside the file is `prefix-route`, and the desired path prefix in the main VirtualServer is `/prefix`. Adjust the script's logic (`path_prefix` extraction) if your convention differs.
*   Ensure the `namespace:` specified in the VirtualServer, VirtualServerRoutes, and the `NAMESPACE` variable in the script are consistent.
*   This approach relies on a manual script execution step before `kustomize build`. For fully automated GitOps workflows, consider alternative approaches like custom KRM Functions executed via `kustomize fn run`, which can perform similar logic directly within the Kustomize build process without requiring external script execution.
