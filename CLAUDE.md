# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the Tekton Operator - a Kubernetes operator that installs, upgrades, and manages Tektoncd components (Pipelines, Dashboard, Triggers, Chains, Hub, Results, and Manual Approval Gates) on Kubernetes and OpenShift clusters.

The operator follows Kubernetes controller/reconciler patterns and uses the Knative injection framework for dependency management.

## Build & Development Commands

### Building and Testing
```bash
# Run unit tests
make test

# Run unit tests with verbose output
make test-unit-verbose

# Run unit tests with race detection
make test-unit-race

# Clean test cache
make test-clean

# Run linters
make lint              # Run all linters
make lint-go           # Go linter only
make lint-yaml         # YAML linter only
```

### Code Generation
```bash
# Generate Kubernetes client code, deepcopy, informers, and listers
./hack/update-codegen.sh

# Verify generated code is up to date
./hack/verify-codegen.sh

# Update Go dependencies
./hack/update-deps.sh
```

### Local Development
```bash
# Set up local development environment with kind and local registry
export KO_DOCKER_REPO="localhost:5000"
make dev-setup

# For podman runtime instead of docker
export CONTAINER_RUNTIME=podman
systemctl --user start podman.socket
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock
make dev-setup
```

### Deploying the Operator

**For Kubernetes:**
```bash
# Set container registry (required by ko)
export KO_DOCKER_REPO=<your-registry>

# Fetch component releases
make get-releases

# Deploy operator to cluster
make apply

# Apply Tekton component CRs (basic profile: Pipeline + Triggers)
make apply-cr
# Or specify custom CR directory
make CR=config/basic apply-cr
```

**For OpenShift:**
```bash
export KO_DOCKER_REPO=<your-registry>
make TARGET=openshift get-releases
make TARGET=openshift apply
make TARGET=openshift apply-cr
```

### Cleaning Up
```bash
# Clean everything (cluster, binaries, manifests)
make clean

# Clean only the cluster
make clean-cluster

# Clean only CRs
make clean-cr

# Clean only binaries
make clean-bin

# Clean only manifests
make clean-manifest
```

### E2E Testing
```bash
# Run E2E tests (requires cluster)
./test/e2e-tests.sh

# Skip cluster creation if cluster already exists
export E2E_SKIP_CLUSTER_CREATION=true
./test/e2e-tests.sh

# Test pre-installed operator
export E2E_SKIP_CLUSTER_CREATION=true
export E2E_SKIP_OPERATOR_INSTALLATION=true
./test/e2e-tests.sh

# Test OpenShift target
E2E_SKIP_CLUSTER_CREATION=true E2E_SKIP_OPERATOR_INSTALLATION=true TARGET=openshift ./test/e2e-tests.sh
```

### Component Version Management
```bash
# Bump component versions (updates components.yaml)
make components/bump COMPONENT=components.yaml

# Bump bugfix version only
make components/bump-bugfix COMPONENT=components.yaml

# Fetch releases for specific component file
make get-releases COMPONENT=components.yaml
```

## Architecture

### Dual-Platform Design

The operator supports two platforms with separate entry points and implementations:
- **Kubernetes**: `cmd/kubernetes/operator/main.go` → `pkg/reconciler/kubernetes/`
- **OpenShift**: `cmd/openshift/operator/main.go` → `pkg/reconciler/openshift/`

Each platform has its own operator binary, webhook binary, and proxy-webhook binary.

### Core CRDs (Custom Resource Definitions)

Located in `pkg/apis/operator/v1alpha1/`:

**Primary CRDs:**
- `TektonConfig` - Top-level configuration managing all Tekton components
- `TektonPipeline` - Manages Tekton Pipelines installation
- `TektonTrigger` - Manages Tekton Triggers installation
- `TektonDashboard` - Manages Tekton Dashboard installation
- `TektonChain` - Manages Tekton Chains (supply chain security)
- `TektonHub` - Manages Tekton Hub integration
- `TektonResult` - Manages Tekton Results (long-term result storage)
- `TektonAddon` - Manages additional Tekton addons (OpenShift-specific)
- `OpenShiftPipelinesAsCode` - Manages Pipelines-as-Code (OpenShift)
- `ManualApprovalGate` - Manages manual approval gates
- `TektonInstallerSet` - Internal CR for managing installation manifests

Each CRD type has associated files:
- `*_types.go` - Type definitions
- `*_lifecycle.go` - Status/lifecycle management
- `*_validation.go` - Validation logic
- `*_defaults.go` - Default values

### Reconciler Pattern

Each component follows the standard Kubernetes operator pattern in `pkg/reconciler/`:

**Structure per component (e.g., `tektonpipeline/`):**
- `controller.go` - Sets up the controller with informers and event handlers
- `reconcile.go` - Main reconciliation logic
- `finalize.go` - Cleanup logic when CR is deleted
- `transform.go` - Manifest transformation before applying
- `metrics.go` - Prometheus metrics (if applicable)

**Shared reconciler utilities:**
- `pkg/reconciler/common/` - Common reconciliation utilities shared by both platforms
- `pkg/reconciler/shared/` - Shared business logic (hash computation, TektonInstallerSet management)
- `pkg/reconciler/platform/` - Platform abstraction layer

### Manifest Management (kodata)

Component manifests are stored in `cmd/{kubernetes,openshift}/operator/kodata/`:
- Each Tekton component has its own subdirectory
- `components.yaml` defines upstream component versions
- `hack/fetch-releases.sh` downloads release manifests from upstream GitHub repos
- Manifests are embedded into operator binary at build time via ko

### Installation Flow

1. User creates a TektonConfig CR (or individual component CR)
2. Reconciler fetches component CR
3. Reconciler creates TektonInstallerSet CR with component manifests
4. TektonInstallerSet controller applies manifests to cluster
5. Status is propagated back up through TektonInstallerSet → Component → TektonConfig

TektonInstallerSet provides:
- Atomic manifest application
- Rollback capability
- Status tracking across multiple resources

### Webhooks

The operator includes admission webhooks for validation and defaulting:
- `cmd/{kubernetes,openshift}/webhook/` - Validation/defaulting webhook
- `cmd/{kubernetes,openshift}/proxy-webhook/` - Proxy webhook for component CRs
- `pkg/webhook/` - Webhook implementation and test data

## Key Patterns to Follow

### When Adding a New CRD or Component

1. Define types in `pkg/apis/operator/v1alpha1/*_types.go`
2. Add lifecycle methods in `*_lifecycle.go` (status management, conditions)
3. Add validation in `*_validation.go`
4. Add defaults in `*_defaults.go`
5. Run `./hack/update-codegen.sh` to generate client code
6. Create reconciler package in `pkg/reconciler/{kubernetes,openshift}/newcomponent/`
7. Implement `controller.go`, `reconcile.go`, and `finalize.go`
8. Register controller in platform package (`kubernetesplatform.go` or `openshiftplatform.go`)
9. Add manifests to `cmd/{target}/operator/kodata/`
10. Add component entry to `components.yaml` if tracking upstream releases

### Component Reconciliation Pattern

```go
// In reconcile.go
func (r *Reconciler) ReconcileKind(ctx context.Context, tc *v1alpha1.TektonComponent) pkgreconciler.Event {
    // 1. Mark as reconciling
    tc.Status.MarkReconciling()

    // 2. Validate
    if err := validate(tc); err != nil {
        tc.Status.MarkNotReady(err.Error())
        return err
    }

    // 3. Create/update TektonInstallerSet with manifests
    installerSet := makeInstallerSet(tc)
    if err := r.createOrUpdateInstallerSet(installerSet); err != nil {
        return err
    }

    // 4. Check installerSet status and propagate
    if installerSet.IsReady() {
        tc.Status.MarkReady()
    }

    return nil
}
```

### Target-Specific Code

Use the `TARGET` environment variable and platform abstraction:
- Kubernetes-specific code goes in `pkg/reconciler/kubernetes/`
- OpenShift-specific code goes in `pkg/reconciler/openshift/`
- Shared code goes in `pkg/reconciler/common/` or `pkg/reconciler/shared/`

### Logging Conventions

Use structured logging with context:
```go
logger := logging.FromContext(ctx)
logger.Infow("Reconciling component", "name", component.Name, "namespace", component.Namespace)
```

Recent commits show focus on improving operator logging clarity for reconcile operations.

## Testing

### Unit Tests
- Place test files alongside source: `*_test.go`
- Use `pkg/reconciler/common/testing` package for test fixtures
- Mock resources in `testdata/` directories

### E2E Tests
- Located in `test/` directory
- See `test/README.md` for detailed test documentation
- Use `test/e2e-tests.sh` as entry point

## Common Gotchas

1. **Always run codegen after API changes**: `./hack/update-codegen.sh`
2. **Use ko for building**: This operator uses ko, not standard docker build
3. **Manifests must be fetched**: Run `make get-releases` before `make apply`
4. **Platform matters**: Always specify `TARGET=openshift` when targeting OpenShift
5. **TektonInstallerSet is internal**: Don't modify TektonInstallerSet directly; it's managed by component reconcilers
6. **Vendored dependencies**: The project uses vendoring; use `./hack/update-deps.sh` not `go mod vendor`

## Release Management

- Releases follow semantic versioning with LTS (Long Term Support) versions
- Release process documented in `tekton/README.md`
- Component versions tracked in `components.yaml` and `components.nightly.yaml`
- OperatorHub bundles managed in `operatorhub/` directory
