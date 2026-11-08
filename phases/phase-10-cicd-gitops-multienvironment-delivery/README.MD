# Phase 10 — CI/CD, GitOps Automation & Multi-Environment Delivery

## Overview

This phase focuses on integrating CI/CD pipelines with GitOps workflows to automate application builds, container image management, Kubernetes deployments, and multi-environment delivery.

It covers GitHub Actions CI, Order Service CI and GitOps automation, multi-environment GitOps promotion, Argo CD ApplicationSets, Helm chart versioning, immutable image digests, automatic image promotion, environment promotion workflows, production release gates, and Jira change management.

The objective is to establish an automated and controlled delivery process in which application changes move from source code through CI validation and image publishing to Kubernetes deployment across multiple environments.

## Objectives

- Implement CI workflows using GitHub Actions.
- Automate Order Service CI and GitOps integration.
- Understand multi-environment GitOps promotion.
- Manage multiple applications using Argo CD ApplicationSets.
- Expand ApplicationSets from a single service to multiple services.
- Manage Helm chart versioning and releases.
- Understand immutable container image digests.
- Automate image promotion from CI to GitOps.
- Implement DEV, QA, STAGING, and production promotion workflows.
- Establish production release gates.
- Integrate Jira change management into production delivery workflows.

## Chapter Index

| Chapter | Documentation |
|---|---|
| 62 | [GitHub Actions CI](62-GitHub-Actions-CI.md) |
| 67 | Order Service CI & GitOps Automation |
| 68 | Multi-Environment GitOps Promotion |
| 69 | Argo CD ApplicationSet and 21-Microservice Configuration |
| 70 | Hands-On: Create Your First ApplicationSet |
| 71 | Expand ApplicationSet from 1 Service |
| 72 | Multi-Environment ApplicationSets |
| 73 | Helm Chart Versioning and Release Management |
| 74 | Immutable Image Digest and Production Image Management |
| 75 | CI-to-GitOps Automatic Image Promotion |
| 76 | [GitOps Promotion: DEV to QA](76-GitOps-Promotion-DEV-to-QA.md) |
| 77 | [GitOps Promotion: QA to STAGING](77-GitOps-Promotion-QA-to-STAGING.md) |
| 78 | Production Release Gate and Promotion |
| 79 | Jira Change Management and Production Release |

## Topics Covered

### 1. GitHub Actions CI
- GitHub Actions workflow fundamentals.
- CI pipeline execution.
- Automated application validation and builds.

### 2. Order Service CI and GitOps Automation
- Order Service CI workflow.
- Connecting CI pipelines with GitOps.
- Automating application delivery workflows.

### 3. Multi-Environment GitOps Promotion
- Managing deployment configuration across environments.
- Environment-specific GitOps configuration.
- Promoting application changes between environments.

### 4. Argo CD ApplicationSets
- ApplicationSet fundamentals.
- Managing multiple Kubernetes applications.
- ApplicationSet configuration for microservices.
- Scaling application management across services.

### 5. ApplicationSet Hands-On Implementation
- Creating the first ApplicationSet.
- Expanding ApplicationSet from one service.
- Managing multi-environment ApplicationSets.

### 6. Helm Chart Versioning and Release Management
- Helm chart version management.
- Application release versioning.
- Managing chart updates across environments.

### 7. Immutable Image Digest and Production Image Management
- Container image digest fundamentals.
- Immutable image references.
- Image traceability and consistency.
- Managing production image versions.

### 8. CI-to-GitOps Automatic Image Promotion
- Connecting CI image publishing with GitOps updates.
- Automating image version updates.
- Managing deployment configuration changes.

### 9. GitOps Environment Promotion
- DEV-to-QA promotion.
- QA-to-STAGING promotion.
- Environment-based deployment workflows.
- Controlled application version progression.

### 10. Production Release Gate and Promotion
- Production release controls.
- Release validation and approval considerations.
- Production promotion workflow.
- Controlled delivery of application changes.

### 11. Jira Change Management
- Jira change management.
- Production change tracking.
- Release documentation and traceability.
- Connecting change management with production releases.

## Expected Outcome

After completing this phase, you should understand how to connect CI pipelines with GitOps-based application delivery and automate the movement of application versions across multiple environments.

You should be able to explain how GitHub Actions supports CI, how Argo CD ApplicationSets manage multiple microservices, how Helm chart versions and immutable image digests support release consistency, and how automated GitOps promotion moves changes through DEV, QA, STAGING, and production.

You should also understand the role of production release gates and Jira change management in maintaining controlled and traceable production releases.

## Completion Checklist

- [ ] GitHub Actions CI workflow documented.
- [ ] Order Service CI and GitOps automation documented.
- [ ] Multi-environment GitOps promotion documented.
- [ ] Argo CD ApplicationSet configuration documented.
- [ ] First ApplicationSet creation documented.
- [ ] ApplicationSet expansion documented.
- [ ] Multi-environment ApplicationSets documented.
- [ ] Helm chart versioning and release management documented.
- [ ] Immutable image digest management documented.
- [ ] CI-to-GitOps automatic image promotion documented.
- [ ] DEV-to-QA promotion documented.
- [ ] QA-to-STAGING promotion documented.
- [ ] Production release gate and promotion documented.
- [ ] Jira change management documented.
- [ ] All fourteen chapter files organized in this phase folder.