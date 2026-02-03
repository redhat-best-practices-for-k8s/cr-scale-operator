# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

The **cr-scale-operator** is a Kubernetes operator that provides a scalable Custom Resource (CR) for QE testing purposes. It implements a Memcached-based operator that supports scaling via:
- Replica count (spec.size)
- Horizontal Pod Autoscaler (HPA)

This operator is primarily used by the Red Hat Best Practices for Kubernetes team to test certification suite scenarios that require scalable custom resources.

### Key Features
- Custom Resource Definition (CRD) for `Memcached` in the `cache.example.com` API group
- Scale subresource support for integration with `kubectl scale` and HPA
- Automatic Deployment management based on CR spec
- Status tracking with conditions and replica count

## Build Commands

### Development

```bash
# Generate code (DeepCopy methods, etc.)
make generate

# Generate CRD manifests, RBAC, and webhooks from kubebuilder markers
make manifests

# Build the manager binary
make build

# Run go fmt
make fmt

# Run go vet
make vet

# Run the controller locally (requires KUBECONFIG)
make run
```

### Convenience Scripts

```bash
# Compile and install CRDs (runs generate, manifests, install)
./scripts/compile.sh

# Build image, push to registry, and deploy operator
./scripts/build-deploy.sh <image-registry>/cr-scale-operator:<version> <namespace>
```

### Container Images

```bash
# Build container image with Docker
make docker-build IMG=<registry>/cr-scale-operator:<tag>

# Build container image with Podman
make podman-build IMG=<registry>/cr-scale-operator:<tag>

# Push image to registry
make docker-push IMG=<registry>/cr-scale-operator:<tag>
make podman-push IMG=<registry>/cr-scale-operator:<tag>

# Multi-architecture build
make docker-buildx IMG=<registry>/cr-scale-operator:<tag>
make podman-build-multiarch IMG=<registry>/cr-scale-operator:<tag>
```

### Deployment

```bash
# Install CRDs only
make install

# Deploy controller to cluster
make deploy IMG=<registry>/cr-scale-operator:<tag>

# Install sample CRs
kubectl apply -f config/samples/

# Uninstall CRDs
make uninstall

# Undeploy controller
make undeploy
```

### OLM Bundle

```bash
# Generate OLM bundle
make bundle

# Build bundle image
make bundle-build

# Push bundle image
make bundle-push

# Build catalog image
make catalog-build

# Push catalog image
make catalog-push
```

## Test Commands

```bash
# Run unit tests with coverage
make test

# Test output is written to cover.out
```

Tests use Ginkgo/Gomega BDD framework with envtest for Kubernetes API simulation.

## Code Organization

```
cr-scale-operator/
├── api/
│   └── v1/
│       ├── groupversion_info.go    # API group/version registration
│       ├── memcached_types.go      # CRD type definitions (Memcached, MemcachedSpec, MemcachedStatus)
│       └── zz_generated.deepcopy.go # Generated DeepCopy methods
├── controllers/
│   ├── memcached_controller.go     # Main reconciliation logic
│   └── suite_test.go               # Controller test suite setup
├── config/
│   ├── crd/                        # CRD manifests
│   ├── default/                    # Default kustomize overlay
│   ├── manager/                    # Controller manager deployment
│   ├── manifests/                  # OLM manifests
│   ├── prometheus/                 # Prometheus monitoring config
│   ├── rbac/                       # RBAC resources
│   ├── samples/                    # Sample CR instances
│   │   ├── cache_v1_memcached.yaml # Sample Memcached CR
│   │   └── hpa.yaml               # Sample HorizontalPodAutoscaler
│   └── scorecard/                  # Operator SDK scorecard config
├── hack/
│   └── boilerplate.go.txt          # License header template
├── scripts/
│   ├── build-deploy.sh             # Build and deploy script
│   ├── cleanup.sh                  # Cleanup script
│   ├── compile.sh                  # Compile and install CRDs
│   └── test.sh                     # Test runner
├── main.go                         # Application entrypoint
├── Dockerfile                      # Multi-stage container build
├── Makefile                        # Build automation
└── PROJECT                         # Kubebuilder project metadata
```

### API Structure

The operator defines a single CRD:

**Memcached** (`cache.example.com/v1`):
- **Spec**:
  - `size` (int32): Desired number of replicas
- **Status**:
  - `replicas` (int32): Current replica count
  - `selector` (string): Label selector for pods
  - `conditions` ([]Condition): Status conditions (Available, Ready)

The CRD includes scale subresource annotations enabling HPA support:
```go
// +kubebuilder:subresource:scale:specpath=.spec.size,statuspath=.status.replicas,selectorpath=.status.selector
```

### Controller Logic

The `MemcachedReconciler`:
1. Fetches the Memcached CR
2. Sets initial status conditions if not present
3. Creates a Deployment if it doesn't exist (using memcached:1.4.36-alpine image)
4. Syncs Deployment replicas with CR spec.size
5. Updates status with current replica count and selector
6. Sets owner references for garbage collection

## Key Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| Go | 1.25 | Programming language |
| controller-runtime | v0.14.1 | Operator framework |
| Kubernetes client-go | v0.26.0 | Kubernetes API client |
| Ginkgo | v2.6.0 | BDD testing framework |
| Gomega | v1.24.1 | Matcher library |
| Operator SDK | v1.31.0 | Operator tooling |
| Kustomize | v3.8.7 | Kubernetes manifest customization |
| envtest | (bundled) | Kubernetes API testing |

## Development Guidelines

### Go Version
This project uses Go 1.25. Ensure your local environment matches this version.

### Kubebuilder Scaffolding
This project was generated using Kubebuilder v3 with the Operator SDK plugins. The project layout follows standard Kubebuilder conventions.

### Modifying the API
When editing API definitions in `api/v1/memcached_types.go`:
1. Update the struct definitions
2. Run `make generate` to regenerate DeepCopy methods
3. Run `make manifests` to regenerate CRD manifests
4. Run `make install` to apply updated CRDs to cluster

### RBAC Markers
Controller RBAC permissions are defined via kubebuilder markers in `controllers/memcached_controller.go`:
```go
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
```

### Testing with HPA
The operator supports Horizontal Pod Autoscaler via the scale subresource. Sample HPA configuration is provided in `config/samples/hpa.yaml`.

### Container Image
- Default registry: `quay.io/testnetworkfunction/cr-scale-operator`
- Uses distroless base image for minimal attack surface
- Runs as non-root user (UID 65532)

### Environment Variables
When deploying:
- `KUBECONFIG`: Path to cluster kubeconfig
- `IMG`: Container image reference
- `NAMESPACE`: Target namespace (defaults to `cr-scale-operator-system`)

## Common Workflows

### Quick Start Development
```bash
# Install CRDs
make install

# Run controller locally
make run

# In another terminal, create sample CR
kubectl apply -f config/samples/cache_v1_memcached.yaml

# Scale the CR
kubectl scale memcached memcached-sample --replicas=3
```

### Full Build and Deploy
```bash
export IMG=quay.io/yourorg/cr-scale-operator:v0.0.2
export NAMESPACE=my-namespace
./scripts/build-deploy.sh $IMG $NAMESPACE
```

### Testing Scale Subresource
```bash
# Verify scale subresource works
kubectl scale memcached memcached-sample --replicas=5

# Or use HPA
kubectl apply -f config/samples/hpa.yaml
```

## License

Apache License 2.0
