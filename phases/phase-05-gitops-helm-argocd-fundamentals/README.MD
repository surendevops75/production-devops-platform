# Phase 05 — GitOps, Helm & Argo CD Fundamentals

## Overview

This phase focuses on GitOps principles and Kubernetes application delivery using Helm and Argo CD.

It covers GitOps repository management, Helm chart fundamentals, Argo CD concepts, and their practical implementation.

The objective is to establish a declarative deployment approach in which Git repositories maintain the desired application configuration and Argo CD synchronizes that configuration with Kubernetes environments.

## Objectives

- Understand GitOps principles and workflows.
- Design a GitOps repository structure.
- Understand Helm charts and their components.
- Learn how Helm manages Kubernetes application deployments.
- Understand Argo CD architecture and functionality.
- Implement GitOps workflows using Argo CD.
- Manage application deployments through Git-based configuration.
- Understand synchronization and deployment management.

## Chapter Index

### GitOps and Tool Fundamentals

| Chapter | Documentation |
|---|---|
| 27 | [GitOps Repository](27-GitOps-Repository.md) |
| 28 | [Helm](28-Helm.md) |
| 29 | [Argo CD](29-ArgoCD.md) |

### Practical Implementation

| Chapter | Documentation |
|---|---|
| 63 | [Helm Implementation](63-Helm-Implementation.md) |
| 64 | [GitOps Implementation](64-GitOps-Implementation.md) |
| 65 | [Argo CD Implementation](65-ArgoCD-Implementation.md) |

## Topics Covered

### 1. GitOps Repository
- GitOps repository organization.
- Managing Kubernetes deployment manifests.
- Separating application source code and deployment configuration.
- Maintaining Git as the desired-state source.

### 2. Helm
- Helm architecture and fundamentals.
- Helm charts and templates.
- Values files and configuration management.
- Managing Kubernetes application deployments.

### 3. Argo CD
- Argo CD architecture.
- GitOps-based continuous delivery.
- Application and repository configuration.
- Synchronization and deployment health.

### 4. Helm Implementation
- Practical Helm chart implementation.
- Managing deployment templates and values.
- Deploying applications using Helm.

### 5. GitOps Implementation
- Organizing GitOps configuration.
- Managing deployment changes through Git.
- Maintaining desired application state.
- Applying GitOps workflows to Kubernetes environments.

### 6. Argo CD Implementation
- Practical Argo CD configuration.
- Connecting Git repositories.
- Managing Kubernetes applications.
- Synchronization and deployment validation.

## Expected Outcome

After completing this phase, you should understand how GitOps, Helm, and Argo CD work together to manage Kubernetes application deployments.

You should be able to explain the difference between foundational GitOps concepts and their practical implementation, organize deployment configuration in Git, manage Helm charts, and understand how Argo CD synchronizes application deployments with the desired state.

These capabilities prepare the project for multi-environment deployment automation and production-grade application delivery.

## Completion Checklist

- [ ] GitOps repository structure documented.
- [ ] Helm fundamentals reviewed.
- [ ] Argo CD fundamentals reviewed.
- [ ] Helm implementation documented.
- [ ] GitOps implementation documented.
- [ ] Argo CD implementation documented.
- [ ] Git-based deployment workflow understood.
- [ ] All six chapter files organized in this phase folder.