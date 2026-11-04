# Phase 02 — CI/CD Pipeline Foundation

## Overview

This phase focuses on designing and implementing CI/CD pipelines for the microservices platform.

It covers three major CI/CD platforms — GitHub Actions, GitLab CI, and Jenkins — along with pipeline security and quality gates, SonarQube, Trivy, ECR image versioning, and artifact and image promotion.

The objective is to establish a consistent and reliable build, validation, security scanning, and artifact management workflow before deploying applications to Kubernetes.

## Objectives

- Understand CI/CD pipeline architecture and workflow.
- Implement CI pipelines using GitHub Actions.
- Understand GitLab CI pipeline configuration.
- Build and manage Jenkins pipelines.
- Integrate security and code quality gates.
- Perform source code analysis using SonarQube.
- Scan container images using Trivy.
- Implement ECR image versioning.
- Understand artifact and image promotion between environments.

## Chapter Index

| Chapter | Documentation |
|---|---|
| 09 | [CI Pipeline Foundation](09-CI-Pipeline-Foundation.md) |
| 10 | [GitHub Actions CI](10-GitHub-Actions-CI.md) |
| 11 | [GitLab CI](11-GitLab-CI.md) |
| 12 | [Jenkins CI](12-Jenkins-CI.md) |
| 13 | [CI Security and Quality Gates](13-CI-Security-and-Quality-Gates.md) |
| 14 | [SonarQube](14-SonarQube.md) |
| 15 | [Trivy](15-Trivy.md) |
| 16 | [ECR Image Versioning](16-ECR-Image-Versioning.md) |
| 17 | [CI Artifact and Image Promotion](17-CI-Artifact-and-Image-Promotion.md) |

## Topics Covered

### 1. CI Pipeline Foundation
- CI pipeline architecture.
- Build and validation workflow.
- Pipeline execution stages.

### 2. GitHub Actions CI
- GitHub Actions workflows.
- Pipeline jobs and steps.
- Automated build and validation.

### 3. GitLab CI
- GitLab CI configuration.
- Pipeline stages and jobs.
- Automated CI workflows.

### 4. Jenkins CI
- Jenkins pipeline fundamentals.
- Pipeline execution and configuration.
- Jenkins-based CI workflows.

### 5. CI Security and Quality Gates
- Integrating security and quality checks into CI.
- Establishing pipeline quality gates.
- Controlling progression based on validation results.

### 6. SonarQube
- Source code quality analysis.
- Code quality gates.
- Integrating SonarQube into CI pipelines.

### 7. Trivy
- Container image vulnerability scanning.
- Security scanning in CI pipelines.
- Identifying vulnerabilities before image promotion.

### 8. ECR Image Versioning
- Managing container image versions.
- Tagging images.
- Maintaining traceability between builds and images.

### 9. CI Artifact and Image Promotion
- Managing build artifacts and container images.
- Promoting validated images between environments.
- Maintaining consistency across deployment stages.

## Expected Outcome

After completing this phase, the project should have a documented CI/CD foundation covering GitHub Actions, GitLab CI, and Jenkins.

The workflows should establish how application code is built, validated, scanned, packaged, versioned, and prepared for promotion.

This phase prepares the project for Kubernetes deployment, Helm, and GitOps automation.

## Completion Checklist

- [ ] CI pipeline foundation documented.
- [ ] GitHub Actions CI workflow reviewed.
- [ ] GitLab CI workflow reviewed.
- [ ] Jenkins CI workflow reviewed.
- [ ] Security and quality gates documented.
- [ ] SonarQube integration documented.
- [ ] Trivy scanning documented.
- [ ] ECR image versioning documented.
- [ ] Artifact and image promotion workflow documented.
- [ ] All nine chapter files organized in this phase folder.