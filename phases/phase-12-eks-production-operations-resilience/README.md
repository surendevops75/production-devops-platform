# Phase 12 — EKS Production Operations & Resilience

## Overview

This phase focuses on advanced Amazon EKS production operations and platform resilience.

It covers EKS cluster operations, node groups and Karpenter, cluster upgrades and lifecycle management, backup and recovery architecture, DevOps connectivity and automation, multi-region disaster recovery, multi-AZ workload placement, Kubernetes resource management, production autoscaling using HPA, VPA, and KEDA, and cluster autoscaling.

The objective is to understand how to operate, scale, maintain, and recover an EKS platform while supporting highly available and resilient microservices workloads.

## Objectives

- Understand production EKS cluster operations.
- Manage EKS node groups and Karpenter.
- Understand EKS upgrades and cluster lifecycle management.
- Design EKS backup and recovery architecture.
- Understand DevOps connectivity and automation.
- Plan multi-region disaster recovery.
- Configure multi-AZ workload placement.
- Manage Kubernetes resource requests and limits.
- Understand HPA, VPA, and KEDA implementation.
- Implement cluster autoscaling.
- Improve production platform availability and resilience.

## Chapter Index

| Chapter | Documentation |
|---|---|
| 94 | [EKS Cluster Operations](94-EKS-Cluster-Operations.md) |
| 95 | EKS Node Groups and Karpenter |
| 96 | EKS Upgrades and Cluster Lifecycle |
| 97 | EKS Backup and Recovery Architecture |
| 98 | Complete DevOps Connectivity and Automation |
| 99 | EKS Multi-Region Disaster Recovery |
| 101 | [Multi-AZ Workload Placement](101-Multi-AZ-Workload-Placement.md) |
| 102 | Kubernetes Resource Management |
| 103 | HPA, VPA and KEDA Production Implementation |
| 104 | [Cluster Autoscaling](104-Cluster-Autoscaling.md) |

## Topics Covered

### 1. EKS Cluster Operations
- Production EKS cluster management.
- Cluster operational procedures.
- Kubernetes cluster health and troubleshooting.
- Managing production workloads.

### 2. EKS Node Groups and Karpenter
- EKS node group management.
- Worker node lifecycle.
- Karpenter fundamentals.
- Dynamic node provisioning and capacity management.

### 3. EKS Upgrades and Cluster Lifecycle
- EKS cluster upgrade planning.
- Kubernetes version compatibility.
- Node group upgrades.
- Cluster lifecycle management.

### 4. EKS Backup and Recovery Architecture
- EKS backup architecture.
- Cluster and application recovery considerations.
- Backup and restore planning.
- Recovery validation.

### 5. Complete DevOps Connectivity and Automation
- DevOps platform connectivity.
- Infrastructure and application integration.
- Operational automation.
- Connectivity across platform components.

### 6. EKS Multi-Region Disaster Recovery
- Multi-region disaster recovery architecture.
- Regional failure scenarios.
- Recovery planning.
- Application and infrastructure recovery considerations.

### 7. Multi-AZ Workload Placement
- Availability Zone architecture.
- Kubernetes workload distribution.
- High-availability placement.
- Improving resilience against zone-level failures.

### 8. Kubernetes Resource Management
- CPU and memory requests.
- Resource limits.
- Workload resource allocation.
- Kubernetes resource utilization and optimization.

### 9. HPA, VPA and KEDA Production Implementation
- Horizontal Pod Autoscaler (HPA).
- Vertical Pod Autoscaler (VPA).
- Kubernetes Event-Driven Autoscaling (KEDA).
- Workload scaling based on resource utilization and events.

### 10. Cluster Autoscaling
- Cluster autoscaling fundamentals.
- Node capacity management.
- Scaling worker nodes based on workload demand.
- Integrating workload scaling with cluster capacity.

## Expected Outcome

After completing this phase, you should understand how to manage an EKS cluster throughout its production lifecycle.

You should be able to explain cluster operations, node group management, Karpenter, cluster upgrades, backup and recovery, multi-region disaster recovery, and multi-AZ workload placement.

You should also understand Kubernetes resource management and the differences between HPA, VPA, KEDA, and cluster autoscaling.

These capabilities help establish a scalable, highly available, and resilient EKS platform for production microservices.

## Completion Checklist

- [ ] EKS cluster operations documented.
- [ ] EKS node groups and Karpenter documented.
- [ ] EKS upgrades and cluster lifecycle documented.
- [ ] EKS backup and recovery architecture documented.
- [ ] DevOps connectivity and automation documented.
- [ ] Multi-region disaster recovery documented.
- [ ] Multi-AZ workload placement documented.
- [ ] Kubernetes resource management documented.
- [ ] HPA, VPA, and KEDA implementation documented.
- [ ] Cluster autoscaling documented.
- [ ] All visible chapter files organized in this phase folder.