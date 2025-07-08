# Slurm Operator Controller Summaries

This document provides detailed summaries of the two main controllers in the Slurm Kubernetes Operator.

## Cluster Controller

The Cluster controller manages Slurm cluster lifecycle and serves as the infrastructure layer for Slurm integration.

### Core Reconciliation Logic

The controller follows the standard Kubernetes controller pattern with a primary `Reconcile` method that handles all cluster state changes. It maintains a thread-safe map of Slurm clients that can be shared across controllers and implements configurable concurrency with exponential backoff for failed operations.

### Cluster Lifecycle Phases

**Creation:** When a new Cluster CR is created, the controller validates required fields via webhooks, resolves authentication tokens from Kubernetes Secrets, creates a Slurm REST API client, registers it globally, performs health checks, and updates the status.

**Updates:** For cluster modifications, it detects changes in server URLs or token references, compares existing client configurations, replaces clients if needed, re-validates connectivity, and synchronizes status.

**Deletion:** During deletion, it handles cleanup by removing Slurm clients from the global map and cleaning up internal state without using explicit finalizers.

### Slurm Integration

The controller uses the [Slurm client library](https://github.com/SlinkyProject/slurm-client) for REST API communication with JWT token-based authentication stored in Kubernetes Secrets. It implements continuous health monitoring through ping controllers and automatically recreates clients when tokens change.

### Status and Error Management

The controller maintains an `IsReady` status indicating successful Slurm communication and implements sophisticated error handling with different retry strategies for transient errors, configuration issues, and authentication failures. It uses exponential backoff (1s to 15min) for failed operations and periodic requeue for readiness checks.

### Resources Created

The cluster controller **does not create any pods directly**. It manages:

- **Internal Resources Only:**
  - Slurm REST API client connections stored in an internal map
  - Event channels for coordinating with NodeSet controllers  
  - Status updates on the Cluster custom resource

The design emphasizes reliability through defensive programming, graceful degradation, and proper resource cleanup while maintaining thread safety for concurrent operations.

## NodeSet Controller

The NodeSet controller implements a sophisticated control loop that coordinates Kubernetes pod management with Slurm node lifecycle, serving as the workload layer.

### Core Reconciliation Loop

**Main Flow:**
1. **Initialization** - Validates NodeSet, adopts orphaned resources
2. **Revision Management** - Handles ControllerRevisions for rolling updates  
3. **Pod Discovery** - Lists and adopts/orphans pods based on selector
4. **Expectation Checks** - Verifies controller expectations before proceeding
5. **Core Sync** - Three-phase synchronization:
   - **Slurm Sync**: Manages drain/undrain operations
   - **NodeSet Sync**: Handles scaling operations
   - **Update Sync**: Processes rolling updates
6. **Status Update** - Updates NodeSet status with pod and Slurm counts

### Pod Lifecycle Management

**Scale Up (Scale Out):**
- Uncordons existing pods and undrains Slurm nodes
- Creates new pods using lowest available ordinals
- Uses slow-start batch processing (starts small, doubles on success)
- Creates PVCs before pods

**Scale Down (Scale In):**
- Cordons pods and initiates Slurm node drain
- Waits for Slurm to confirm nodes are fully drained
- Only terminates pods after jobs complete
- Manages PVC retention policies

**Rolling Updates:**
- Uses ControllerRevisions for template versioning
- Respects `MaxUnavailable` for controlled rollout
- Coordinates with Slurm drain/undrain during updates

### Slurm Integration

**Drain/Undrain Coordination:**
```go
MakeNodeDrain()   // Sets DRAIN state, allows jobs to complete
MakeNodeUndrain() // Only undrains operator-managed nodes
IsNodeDrained()   // Checks if node is safe for pod termination
```

**Node State Monitoring:**
- Tracks Slurm node states (IDLE, ALLOCATED, DOWN, DRAIN)
- Maps pod names to Slurm node names
- Provides real-time cluster capacity view

### Error Handling & Resilience

**Expectation-Based Control:**
- Tracks expected pod creations/deletions
- Prevents duplicate work and race conditions
- Uses exponential backoff for failed operations

**Slurm Error Tolerance:**
- Tolerates common API errors (404, 204)
- Continues operation when Slurm is temporarily unavailable
- Implements retry mechanisms with appropriate delays

### Status Management

**Comprehensive Status:**
- Pod counts (Total, Ready, Available, Updated)
- Slurm node states (Idle, Allocated, Down, Drain)
- Generation tracking and hash collision handling
- Requeue logic based on MinReadySeconds and drain status

### Resources Created

The NodeSet controller creates all the actual workload resources:

**Pods Created:**
- **Slurm compute node pods** (one per replica)
- **Naming pattern**: `{nodeset-name}-{ordinal}` (e.g., `compute-0`, `compute-1`)
- **Contains**: Slurm daemon containers (`slurmcd` or `sackd`)

**Additional Resources:**
- **PersistentVolumeClaims** for each pod based on volume claim templates
- **ControllerRevisions** for managing rolling updates

## Architecture Separation

```
Cluster Controller → Infrastructure layer (connections, auth, coordination)
NodeSet Controller → Workload layer (pods, storage, compute resources)
```

The cluster controller focuses on establishing and maintaining the connection to Slurm clusters, while the NodeSet controller handles the actual provisioning of compute resources as Kubernetes pods that integrate with Slurm's scheduling system.

The controller's design ensures **workload-aware scaling** - it never terminates pods with running Slurm jobs, making it safe for HPC workloads while maintaining Kubernetes-native pod management patterns.

## Key Design Decisions

1. **No Finalizers**: Cluster controller relies on Kubernetes garbage collection rather than custom finalizers
2. **Shared Client Store**: Single map of Slurm clients shared across controllers
3. **Event-driven Communication**: Uses channels for inter-controller communication
4. **Defensive Programming**: Extensive error handling and graceful degradation
5. **Thread Safety**: Proper synchronization for concurrent operations
6. **Workload Awareness**: NodeSet controller coordinates with Slurm to prevent job disruption during scaling operations