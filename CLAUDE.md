# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **NVIDIA GPU Operator**, a Kubernetes operator written in Go that automates provisioning and management of all NVIDIA software components (drivers, container toolkit, device plugin, DCGM, etc.) in Kubernetes clusters. It uses controller-runtime and manages two primary CRDs: `ClusterPolicy` and `NVIDIADriver`.

## Common Commands

### Build
```bash
make build           # Build all packages
make cmds            # Build all binaries
make build-image     # Build Docker image
```

### Test
```bash
make unit-test                                          # Run unit tests with coverage
go test -v ./controllers/... -count=1                  # Run controller tests verbosely
go test -v -run TestFunctionName ./controllers/...     # Run a single test by name
```

### Lint & Validation
```bash
make lint                      # Run golangci-lint
make fmt                       # Run gofmt
make check                     # Run all checks (lint, license, validate)
make validate-modules          # Verify go.mod and vendor
make validate-generated-assets # Verify generated code is in sync
```

### Code Generation
```bash
make manifests    # Regenerate CRDs and RBAC manifests from API types
make generate     # Regenerate deepcopy and other generated code
make sync-crds    # Sync CRDs to Helm chart and OLM bundle
```

## Architecture

### Reconcilers (entry point: `cmd/gpu-operator/main.go`)

Three reconcilers run concurrently in the manager:

1. **ClusterPolicyReconciler** (`controllers/clusterpolicy_controller.go`) — primary controller; enforces desired state for all GPU components cluster-wide. Only one `ClusterPolicy` object is allowed per cluster.
2. **NVIDIADriverReconciler** (`controllers/nvidiadriver_controller.go`) — manages the alpha `NVIDIADriver` CRD; handles per-node driver deployment.
3. **UpgradeReconciler** (`controllers/upgrade_controller.go`) — orchestrates rolling upgrades of operator-managed components; filters pods by GPU resource presence before deletion.

### State Machine

`controllers/state_manager.go` drives a per-component state machine (NotReady → Ready → Error). Each managed component (Driver, Toolkit, DevicePlugin, DCGM, GFD, MIG Manager, Validator, etc.) goes through this lifecycle during reconciliation.

### Object Reconciliation

`controllers/object_controls.go` (the largest file, ~192KB) contains all transformation logic for creating/updating Kubernetes objects (DaemonSets, Deployments, Services, ConfigMaps, RBAC). Component-specific customizations live here — look here first when a resource isn't rendering correctly.

### API Types

- `api/nvidia/v1/clusterpolicy_types.go` — GA `ClusterPolicy` spec; 20+ nested component configuration structs
- `api/nvidia/v1alpha1/nvidiadriver_types.go` — alpha `NVIDIADriver` spec

Changes to these types require re-running `make manifests && make generate && make sync-crds`.

### Internal Packages

| Package | Purpose |
|---|---|
| `internal/state/` | Component state machine implementations (one per managed component) |
| `internal/nodeinfo/` | Node attribute detection and label-based filtering |
| `internal/render/` | Helm-style template rendering for manifests |
| `internal/conditions/` | Kubernetes condition utilities |
| `internal/config/` | Operator configuration loading |

### Helm Chart

`deployments/gpu-operator/` is the canonical deployment mechanism. CRD manifests are synced here by `make sync-crds` — do not edit CRDs in the chart directly.

## Contribution Requirements

All commits must be signed off per the Developer Certificate of Origin (DCO):
```bash
git commit -s -m "message"
```

The CI enforces this. A missing sign-off will block merge.
