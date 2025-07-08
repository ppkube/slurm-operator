# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Build and Test
```bash
make build                    # Build container images using docker buildx bake
make test                     # Run tests with coverage (requires 74%+ coverage)
make fmt                      # Format Go code
make vet                      # Run go vet
make tidy                     # Run go mod tidy
make golangci-lint            # Run golangci-lint
make govulncheck              # Run vulnerability checks
```

### Code Generation
```bash
make manifests               # Generate CRDs, webhooks, RBAC
make generate                # Generate deepcopy methods
```

### Development Setup
```bash
make install-dev            # Install development binaries (dlv, kind, etc.)
make run                    # Run controller locally (includes manifests, generate, fmt, tidy, vet)
make install                # Install CRDs into K8s cluster
make uninstall              # Remove CRDs from K8s cluster
```

### Helm Operations
```bash
make helm-validate          # Validate Helm charts
make helm-docs              # Generate Helm documentation
make helm-lint              # Lint Helm charts
make helm-dependency-update # Update chart dependencies
make helmtest               # Run Helm tests
```

## Project Architecture

This is a **Kubernetes Operator** for managing **Slurm HPC clusters** written in **Go** using **Kubebuilder v4**.

### Core Components

1. **Custom Resource Definitions (CRDs)**:
   - `Cluster` (`api/v1alpha1/cluster_types.go`) - Represents a Slurm cluster with authentication and server endpoint
   - `NodeSet` (`api/v1alpha1/nodeset_types.go`) - Represents homogeneous Slurm compute nodes

2. **Controllers**:
   - `internal/controller/cluster/` - Manages Slurm cluster lifecycle
   - `internal/controller/nodeset/` - Manages compute node scaling, upgrades, and lifecycle

3. **Binaries**:
   - `cmd/manager/` - Main operator controller
   - `cmd/webhook/` - Admission webhook server

### Key Features
- **Workload-aware scaling**: Drains nodes before termination to preserve running jobs
- **Scale-to-zero support**: NodeSets can scale down to zero replicas
- **Hybrid deployments**: Integrates with external bare-metal Slurm nodes
- **Multi-architecture support**: linux/amd64, linux/arm64, linux/s390x, linux/ppc64le

### Development Workflow
1. **Pre-commit requirements**: Code must pass `make fmt`, `make vet`, `make tidy`, and `make golangci-lint`
2. **Test coverage**: Minimum 74% test coverage required
3. **Code generation**: Use `make manifests` and `make generate` after API changes
4. **Local development**: Use `make run` to run controller locally against cluster

### File Organization
- `api/v1alpha1/` - API definitions and webhooks
- `internal/controller/` - Controller reconciliation logic
- `internal/resources/` - Kubernetes resource management utilities
- `internal/utils/` - Common utilities and helpers
- `config/` - Kubernetes manifests and deployment configs
- `helm/` - Helm charts for operator and Slurm cluster deployment

### Dependencies
- Built on **controller-runtime v0.20.4** and **Kubernetes v1.33.1**
- Uses **Kubebuilder v4** for scaffolding
- Communicates with **Slurm REST API** (slurmrestd)
- Requires **Go 1.24+** and **Kubernetes >=1.29**

### Installation
```bash
# Install operator
helm install slurm-operator oci://ghcr.io/slinkyproject/charts/slurm-operator \
  --namespace=slinky --create-namespace

# Install Slurm cluster
helm install slurm oci://ghcr.io/slinkyproject/charts/slurm \
  --namespace=slurm --create-namespace
```