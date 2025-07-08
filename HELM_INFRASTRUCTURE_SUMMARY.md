# Slurm Operator Helm Infrastructure Summary

This document provides a comprehensive overview of the static infrastructure resources created by the Slurm Operator Helm charts.

## Overview

The slurm-operator repository contains two main Helm charts:

1. **slurm-operator** - The Kubernetes operator that manages Slurm clusters
2. **slurm** - The complete Slurm HPC cluster deployment

## Kubernetes Resources Created

### slurm-operator Chart

| Resource Type | Count | Purpose |
|---------------|-------|---------|
| Deployments | 2 | Operator and Webhook |
| Services | 2 | Operator and Webhook services |
| ServiceAccounts | 2 | For operator and webhook |
| ClusterRoles | 2 | RBAC permissions |
| ClusterRoleBindings | 2 | RBAC bindings |
| Secrets | 1 | TLS certificates for webhook |
| ValidatingWebhookConfiguration | 1 | Admission controller |
| MutatingWebhookConfiguration | 1 | Admission controller |
| CustomResourceDefinitions | 2 | Cluster and NodeSet CRDs |
| Certificates | 1 | cert-manager certificate (optional) |

### slurm Chart

| Resource Type | Count | Purpose |
|---------------|-------|---------|
| StatefulSets | 2 | Controller (slurmctld) and Accounting (slurmdbd) |
| Deployments | 3 | Login, REST API, and MariaDB (via subchart) |
| Services | 4 | Controller, Accounting, Login, REST API |
| ConfigMaps | 4 | Slurm config, Accounting config, Login config |
| Secrets | 4 | JWT, Slurm auth, Accounting, Login SSH keys |
| PersistentVolumeClaims | 2 | Controller state-save, MariaDB data |
| Jobs | 1 | Token initialization job |
| ServiceAccounts | 1 | For token job |
| Roles/RoleBindings | 2 | Token job RBAC |
| Custom Resources | 2 | Slurm Cluster and NodeSet instances |

## Main Components and Architecture

### Operator Infrastructure (slurm-operator)

**Core Components:**
- **Operator Deployment**: Manages Slurm Cluster and NodeSet custom resources
- **Webhook Deployment**: Validates and mutates Slurm custom resources
- **RBAC**: Permissions to manage pods, secrets, PVCs, and custom resources
- **Admission Controllers**: Ensure valid Slurm configurations

### Slurm Cluster Infrastructure (slurm)

**Control Plane:**
- **Controller (slurmctld)**: Central Slurm daemon managing jobs and resources
- **Accounting (slurmdbd)**: Database daemon for job accounting and reporting
- **REST API (slurmrestd)**: HTTP API for external Slurm interactions

**Data Plane:**
- **Login Nodes**: SSH access points for users (optional)
- **Compute NodeSets**: Managed by operator, not directly created by chart
- **MariaDB**: Database backend for accounting (via bitnami subchart)

## Resource Relationships

```
┌─────────────────────┐    ┌─────────────────────┐
│   slurm-operator    │    │       slurm         │
│                     │    │                     │
│ ┌─────────────────┐ │    │ ┌─────────────────┐ │
│ │    Operator     │ │    │ │   Controller    │ │
│ │   Deployment    │ │    │ │  StatefulSet    │ │
│ └─────────────────┘ │    │ └─────────────────┘ │
│                     │    │                     │
│ ┌─────────────────┐ │    │ ┌─────────────────┐ │
│ │    Webhook      │ │    │ │   Accounting    │ │
│ │   Deployment    │ │    │ │  StatefulSet    │ │
│ └─────────────────┘ │    │ └─────────────────┘ │
│                     │    │                     │
│ ┌─────────────────┐ │    │ ┌─────────────────┐ │
│ │      CRDs       │ │    │ │    REST API     │ │
│ │ Cluster/NodeSet │ │    │ │   Deployment    │ │
│ └─────────────────┘ │    │ └─────────────────┘ │
└─────────────────────┘    │                     │
                           │ ┌─────────────────┐ │
                           │ │ Login Nodes     │ │
                           │ │   Deployment    │ │
                           │ └─────────────────┘ │
                           │                     │
                           │ ┌─────────────────┐ │
                           │ │    MariaDB      │ │
                           │ │   Deployment    │ │
                           │ └─────────────────┘ │
                           └─────────────────────┘
```

## Optional/Conditional Resources

### Conditionally Created Resources

| Component | Condition | Default | Purpose |
|-----------|-----------|---------|---------|
| Login Deployment | `login.enabled=true` | false | User SSH access |
| Accounting StatefulSet | `accounting.enabled=true` | true | Job accounting |
| REST API Deployment | `restapi.enabled=true` | true | HTTP API access |
| MariaDB | `mariadb.enabled=true` | true | Accounting database |
| Slurm Exporter | `slurm-exporter.enabled=true` | true | Metrics collection |
| Webhook Components | `webhook.enabled=true` | true | Admission control |
| Cert-Manager Certificate | `certManager.enabled=true` | true | TLS certificates |

### External Dependencies

- **External Database**: `accounting.external.enabled=true` (alternative to MariaDB)
- **Existing PVCs**: Can use existing storage claims
- **Custom Images**: All container images are configurable

## Overall Architecture

The static infrastructure creates a **two-tier architecture**:

### Tier 1: Operator Layer
- Kubernetes-native operator managing Slurm resources
- Admission webhooks for validation and mutation
- Custom resource definitions for declarative management

### Tier 2: Slurm Cluster Layer
- Complete HPC cluster with all Slurm components
- Persistent storage for state and data
- Network services for internal and external communication
- Authentication and authorization infrastructure

## Key Features

- **High Availability**: StatefulSets ensure persistent identity
- **Scalability**: NodeSets managed dynamically by operator
- **Security**: RBAC, network policies, and secret management
- **Monitoring**: Built-in exporter for metrics collection
- **Flexibility**: Extensive configuration options and overrides

## Resource Naming Patterns

- **Operator resources**: `{release-name}-slurm-operator-{component}`
- **Slurm resources**: `{release-name}-{component}` (e.g., `myslurm-slurmctld`)
- **NodeSets**: `{release-name}-compute-{nodeset-name}`
- **Services**: Match their corresponding deployment/statefulset names

## Component Storage

### Persistent Storage
- **Controller state-save**: Persistent storage for cluster state recovery
- **MariaDB data**: Accounting database persistence
- **Configuration**: Projected volumes with configs and secrets

### Configuration Management
- **ConfigMaps**: Slurm configuration files, accounting setup, login configs
- **Secrets**: JWT tokens, authentication credentials, SSH keys
- **Template Helpers**: Shared configuration templates across components

This architecture provides a complete, production-ready Slurm HPC cluster deployment with Kubernetes-native management capabilities through the operator pattern.