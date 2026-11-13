# Chapter 186 — Security Policy Templates

Phase 20 — Platform Engineering

Project: Enterprise E-commerce Platform — 21 Microservices\
Cloud: AWS (`ap-south-1`)\
Container platform: Amazon EKS\
Container registry: Amazon ECR\
CI/CD: GitHub Actions reusable workflows\
Deployment: Helm, GitOps and Argo CD\
Security tooling: SonarQube, Trivy, secret scanning, dependency scanning and policy-as-code

## 1. Objective

In Chapter 185, we established reusable GitHub Actions workflows to standardize how applications are built, tested, scanned and released.

Now we need to define what those workflows must enforce.

For example, a reusable container scanning workflow can run Trivy, but it needs a policy that answers:

- Which vulnerability severities block a release?
- Can an exception be approved?
- Who can approve the exception?
- How long can an exception remain valid?
- What happens when a critical vulnerability is discovered?
- Can an image be promoted if the scan report is missing?
- Can a developer bypass a required security workflow?

These decisions belong in security policies.

The objective of this chapter is to build a reusable security policy platform for all 21 microservices.

By the end, the platform should have standardized policies for:

1. GitHub repository security.
2. Branch and pull-request protection.
3. Source-code security.
4. Dependency and license compliance.
5. Secret detection.
6. Container image security.
7. Terraform and AWS infrastructure security.
8. Kubernetes workload security.
9. Helm chart security.
10. IAM and GitHub OIDC.
11. GitOps and deployment security.
12. Production release approval.
13. Security exceptions.
14. Audit evidence and compliance.

The important distinction is:

A security scanner finds a problem. A security policy decides what the organization must do about that problem.

## 2. Production security architecture

[Embedding Security in Cloud-Native DevOps Pipelines with DevSecOps | Prakash Jamer posted on the topic | LinkedIn](https://images.openai.com/static-rsc-4/fQbVmsjTHundn0Vy6TaGIPlGSUQYOXSHAb93LAOK4tKZOrzbzGFOFQV6rE9CzhTJ_A-nNF6fT70NKRjja4KyonORmqm_yGFUy08QeXmFnfQFPjHihXcMYUFLczLoDNHNs9LOrzy5Zl5qMMsEg0Nvj7Qkdq13p1LGyAqU6ylR97E?purpose=inline)

[linkedin.com](https://www.linkedin.com/posts/prakash-jamer_devsecops-kubernetes-devops-activity-7435632831529246720-HDzN?utm_source=chatgpt.com)

```
                  Developer
                      |
                      v
               GitHub Repository
                      |
                      v
              Pull Request Security
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
      Secret Scan   SAST       Dependency Scan
          |           |           |
          +-----------+-----------+
                      |
                      v
               Security Policy
                      |
               +------+------+
               |             |
               v             v
            PASS           FAIL
               |             |
               v             v
          Build Image    Block Merge
               |
               v
           Trivy Scan
               |
               v
          Image Policy
               |
          +----+-----+
          |          |
          v          v
         PASS       FAIL
          |          |
          v          v
      ECR Publish   Block Release
          |
          v
       GitOps PR
          |
          v
    Kubernetes Policy
          |
          v
      Argo CD
          |
          v
       Amazon EKS
          |
          v
   Admission Enforcement
          |
          v
    Production Workloads
```

This architecture has multiple enforcement points.

| Enforcement point       | Purpose                                                 |
| ----------------------- | ------------------------------------------------------- |
| GitHub rulesets         | Protect source and workflow changes                     |
| CI security gates       | Detect issues before merge or release                   |
| Terraform policy checks | Block unsafe infrastructure changes                     |
| Helm validation         | Detect unsafe Kubernetes manifests before promotion     |
| ECR verification        | Ensure the approved image is the image being deployed   |
| Kubernetes admission    | Reject noncompliant resources submitted to the cluster  |
| Runtime monitoring      | Detect security and operational issues after deployment |

A successful CI scan does not eliminate the need for Kubernetes admission policies. Someone could submit a manifest through another authorized deployment path. Admission controls provide another enforcement boundary.

## 3. Security ownership and responsibilities

Security policy should not be owned entirely by developers or entirely by the DevOps team.

Each team has a specific responsibility.

| Team                   | Responsibility                                                              |
| ---------------------- | --------------------------------------------------------------------------- |
| Application developers | Secure coding, dependency updates, unit tests and application configuration |
| Platform engineering   | Shared workflows, policy templates, Kubernetes standards and CI enforcement |
| Application security   | Security thresholds, vulnerability assessment and exceptions                |
| Cloud security         | AWS IAM, OIDC, encryption and account-level guardrails                      |
| Infrastructure team    | Terraform security and infrastructure configuration                         |
| Release manager        | Change approval and production release governance                           |
| SRE / operations       | Runtime alerts, incident response and recovery                              |
| DBA / data platform    | Database access, backup and data security controls                          |

### Example: Payment Service vulnerability

Suppose Trivy detects a critical vulnerability in the Payment Service container image.

The responsibilities are:

1. The reusable container scanning workflow identifies the finding.
2. The security policy determines whether the finding blocks release.
3. The application team investigates the affected dependency.
4. The security team reviews a proposed exception if a fix is unavailable and policy allows an exception.
5. The release workflow checks the approved exception and its expiration.
6. The release manager approves production deployment when required.
7. The evidence workflow retains the decision and scan result.

The developer cannot simply add a line to the service workflow saying `continue-on-error: true` to bypass the security gate.

## 4. Security policy repository

Create a dedicated repository for centrally managed security policies.

Repository:

`company-ecommerce/platform-security-policies`

Suggested structure:

```
platform-security-policies/
|
+-- README.md
|
+-- policies/
|   |
|   +-- repository/
|   |   +-- branch-protection.yml
|   |   +-- pull-request-policy.yml
|   |   +-- code-owner-policy.yml
|   |
|   +-- source-code/
|   |   +-- sast-policy.yml
|   |   +-- quality-gate-policy.yml
|   |   +-- secret-scanning-policy.yml
|   |
|   +-- dependency/
|   |   +-- vulnerability-policy.yml
|   |   +-- license-policy.yml
|   |
|   +-- container/
|   |   +-- image-vulnerability-policy.yml
|   |   +-- image-trust-policy.yml
|   |   +-- image-tag-policy.yml
|   |
|   +-- terraform/
|   |   +-- aws-security-policy.yml
|   |   +-- iam-policy.yml
|   |   +-- network-policy.yml
|   |   +-- encryption-policy.yml
|   |
|   +-- kubernetes/
|   |   +-- pod-security-policy.yml
|   |   +-- resource-policy.yml
|   |   +-- serviceaccount-policy.yml
|   |   +-- networkpolicy-policy.yml
|   |   +-- image-policy.yml
|   |
|   +-- release/
|   |   +-- release-eligibility.yml
|   |   +-- production-approval.yml
|   |   +-- exception-policy.yml
|   |
|   +-- exceptions/
|       +-- README.md
|
+-- schemas/
|   +-- security-policy.schema.json
|   +-- exception.schema.json
|
+-- conftest/
|   +-- terraform/
|   +-- kubernetes/
|   +-- helm/
|
+-- kyverno/
|   +-- policies/
|
+-- tests/
|   +-- positive/
|   +-- negative/
|   +-- exceptions/
|
+-- docs/
    +-- policy-ownership.md
    +-- policy-enforcement.md
    +-- exception-process.md
    +-- rollout-guide.md
```

This repository is the central source of policy definitions.

The exact folder structure can evolve, but ownership and version control should remain centralized.

## 5. Policy design principles

Before creating policy files, establish common rules.

### Rule 1 — Default deny for sensitive operations

If a workflow cannot verify a mandatory production security condition, it must not assume the condition passed.

For example:

- Missing scan report: block release.
- Invalid image digest: block promotion.
- Missing production approval: block deployment.
- Expired exception: block release.
- Invalid workflow identity: reject AWS role assumption.

### Rule 2 — Least privilege

Each workflow and workload receives only the permissions it requires.

For example:

- A code-quality job should not have permission to modify EKS.
- A container scanner should not have permission to delete ECR repositories.
- An application pod should not receive administrator IAM permissions.
- A Terraform plan job should not automatically receive production apply permissions.

### Rule 3 — Immutable artifacts

Use immutable image digests for release promotion.

```
order-service:latest
```

is a mutable tag.

```
order-service@sha256:<VERIFIED_DIGEST>
```

identifies a particular image manifest.

The exact digest should be linked to the build, scan, signature, provenance and deployment evidence.

### Rule 4 — Exceptions must be controlled

An exception requires:

- A clear finding.
- An accountable owner.
- A technical justification.
- A mitigation.
- An authorized approver.
- An expiration date.
- A review process.

An exception is not a permanent deletion of a security control.

### Rule 5 — Test policies before enforcement

A new policy can break valid production workloads if it is too broad.

Introduce policies through a controlled rollout:

1. Test against positive and negative fixtures.
2. Run in audit mode.
3. Review real findings.
4. Correct policy false positives.
5. Agree on exceptions.
6. Enable enforcement.
7. Monitor rejected deployments and blocked releases.

## 6. Create the standard security policy contract

All security policies should follow a consistent structure.

This makes policies easier to review, test and consume from reusable workflows.

Example: `policies/container/image-vulnerability-policy.yml`

```
apiVersion: security.company-ecommerce.io/v1
kind: SecurityPolicy

metadata:
  name: container-image-vulnerability
  owner: application-security
  description: Container vulnerability policy for application releases

spec:
  enabled: true

  scope:
    services:
      - "*"
    environments:
      - dev
      - qa
      - staging
      - production

  scanner:
    name: trivy
    required: true

  thresholds:
    critical:
      action: block

    high:
      action: block

    medium:
      action: review

    low:
      action: report

  evidence:
    report-required: true
    image-digest-required: true

  exceptions:
    allowed: true
    require-approval: true
    require-expiration: true
```

This is an organization-defined policy format. GitHub Actions, Trivy and Kubernetes do not automatically understand these custom fields. We must implement a policy evaluator that validates this document and applies its decisions.

### 6.1 Why a common contract matters

Suppose the Order Service and Payment Service both run Trivy.

Without a shared policy, their thresholds might differ:

```
Order Service:
CRITICAL = BLOCK
HIGH     = BLOCK

Payment Service:
CRITICAL = REPORT
HIGH     = REPORT
```

That is an inconsistent security posture.

With a shared policy, both services consume the same approved threshold unless an explicitly reviewed policy exception applies.

### 6.2 Define a JSON Schema

The policy contract should be validated before it is consumed.

Example: `schemas/security-policy.schema.json`

```
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Company Ecommerce Security Policy",
  "type": "object",
  "required": [
    "apiVersion",
    "kind",
    "metadata",
    "spec"
  ],
  "properties": {
    "apiVersion": {
      "type": "string"
    },
    "kind": {
      "const": "SecurityPolicy"
    },
    "metadata": {
      "type": "object",
      "required": [
        "name",
        "owner"
      ],
      "properties": {
        "name": {
          "type": "string",
          "minLength": 1
        },
        "owner": {
          "type": "string",
          "minLength": 1
        },
        "description": {
          "type": "string"
        }
      },
      "additionalProperties": true
    },
    "spec": {
      "type": "object",
      "required": [
        "enabled",
        "thresholds"
      ],
      "properties": {
        "enabled": {
          "type": "boolean"
        },
        "thresholds": {
          "type": "object"
        },
        "exceptions": {
          "type": "object"
        }
      },
      "additionalProperties": true
    }
  },
  "additionalProperties": false
}
```

This schema is a starting point. Extend it to validate all required threshold values, allowed actions, service scopes, exception properties and environment names. Do not rely on a permissive schema as the final production control.

## 7. Repository security policy

The first security boundary is the source repository.

A developer who can modify the workflow or policy files may otherwise be able to weaken the checks protecting the application.

### 7.1 Branch protection

Protect the main branch of every service repository.

Recommended controls:

- Require pull requests.
- Require at least one independent approval.
- Require platform or security review for protected files.
- Require successful CI checks.
- Require security checks.
- Dismiss stale approvals when new commits invalidate the review.
- Require conversation resolution.
- Restrict force pushes.
- Restrict branch deletion.
- Require signed commits where the organization has established signing and verification.
- Prevent direct changes to protected workflow files.

For sensitive repositories, GitHub organization rulesets can centralize these requirements.

### 7.2 Example CODEOWNERS file

File: `order-service/.github/CODEOWNERS`

```
# Application code
/src/ @company-ecommerce/order-team

# Docker configuration
/Dockerfile @company-ecommerce/platform-team

# GitHub workflows
/.github/ @company-ecommerce/platform-team

# Security configuration
/.security/ @company-ecommerce/security-team

# Helm deployment configuration
/deploy/ @company-ecommerce/platform-team

# Database migrations
/db/migrations/ @company-ecommerce/order-team @company-ecommerce/dba-team
```

The GitHub usernames and teams are examples. Replace them with actual organization team identifiers.

CODEOWNERS does not independently force an approval. The corresponding repository ruleset or branch protection setting must require code-owner review.

### 7.3 Protect security-sensitive files

These files should receive additional review:

```
.github/workflows/
.security/
Dockerfile
deploy/
terraform/
```

Why?

A developer could otherwise make changes such as:

```
- name: Security scan
  run: echo "Security scan passed"
```

The job may be green, but no security scan was performed.

The platform must protect the implementation and the policy that defines whether a scan is mandatory.

### 7.4 Pull-request policy

Example: `policies/repository/pull-request-policy.yml`

```
apiVersion: security.company-ecommerce.io/v1
kind: RepositoryPolicy

metadata:
  name: standard-pull-request-policy
  owner: platform-engineering

spec:
  require-pull-request: true

  approvals:
    minimum: 1
    require-code-owner-review: true

  checks:
    required:
      - java-ci
      - secret-scan
      - dependency-scan
      - code-quality

  branch:
    restrict-force-push: true
    restrict-deletion: true

  conversations:
    require-resolution: true
```

This is a policy declaration for the platform's implementation. The actual enforcement is configured through GitHub repository rulesets and required status checks.

## 8. Source-code security policies

Source-code security combines several different controls. They should not be treated as interchangeable.

| Control             | What it detects                                           |
| ------------------- | --------------------------------------------------------- |
| SAST                | Potential vulnerabilities in source code                  |
| SonarQube           | Code quality and configured security analysis             |
| Dependency scanning | Vulnerabilities in third-party libraries                  |
| Secret scanning     | Credentials and tokens exposed in source                  |
| License scanning    | Dependency license risks                                  |
| Code review         | Design and implementation risks that tools may not detect |

### 8.1 SonarQube quality policy

Example:

```
apiVersion: security.company-ecommerce.io/v1
kind: CodeQualityPolicy

metadata:
  name: standard-code-quality
  owner: application-security

spec:
  quality-gate:
    required: true

  new-code:
    coverage:
      minimum: 80

    duplicated-lines:
      maximum-percentage: 3

  security:
    blocker-issues:
      maximum: 0

    critical-issues:
      maximum: 0
```

These are example thresholds, not universal SonarQube defaults. The platform team should verify that each measure maps to an actual supported quality-gate condition in its configured SonarQube edition and project setup.

### 8.2 Important: coverage is not security

A project can have 90% unit test coverage and still contain:

- SQL injection.
- Broken authorization.
- Exposed credentials.
- Insecure deserialization.
- Unsafe file handling.
- Weak cryptographic practices.

Coverage measures which code paths tests exercise. It does not prove that the application is secure.

### 8.3 Secret scanning policy

Example: `policies/source-code/secret-scanning-policy.yml`

```
apiVersion: security.company-ecommerce.io/v1
kind: SecretScanningPolicy

metadata:
  name: secret-detection
  owner: application-security

spec:
  enabled: true

  scan:
    pull-request: true
    default-branch: true
    repository-history: true

  severity:
    verified-secret:
      action: block

    suspected-secret:
      action: review

  remediation:
    revoke-exposed-credential: true
    require-rotation: true
    require-incident-review: true
```

The actual behavior depends on the selected secret scanner and its capabilities. Repository-history scanning, for example, needs a configured scanning mechanism that can inspect the relevant history.

### 8.4 What happens when a secret is found?

Example:

```
Developer commits AWS credentials
            |
            v
Pull Request created
            |
            v
Secret scanning detects credential
            |
            v
Required security check fails
            |
            v
Pull request cannot merge
            |
            v
Security incident assessment
            |
            v
Credential revoked or rotated
            |
            v
Logs and access reviewed
            |
            v
Safe code change submitted
```

Do not merely delete the secret from the file and close the finding. Assume an exposed credential may already have been copied.

## 9. Dependency vulnerability policy

Every service depends on third-party libraries.

For example:

```
Order Service
    |
    +--> Spring Boot
    +--> Spring Security
    +--> MySQL Connector
    +--> Jackson
    +--> Logging libraries
    +--> Other transitive dependencies
```

A vulnerable transitive dependency can affect an application even when developers did not add it directly.

### 9.1 Example policy

File: `policies/dependency/vulnerability-policy.yml`

```
apiVersion: security.company-ecommerce.io/v1
kind: DependencyPolicy

metadata:
  name: dependency-vulnerability-policy
  owner: application-security

spec:
  scan-required: true

  severity:
    critical:
      action: block

    high:
      action: block

    medium:
      action: review

    low:
      action: report

  exceptions:
    require-owner: true
    require-approval: true
    require-expiration: true
```

### 9.2 Dependency remediation process

Suppose the Order Service uses a vulnerable version of Jackson.

The developer should:

1. Identify the affected version.
2. Identify which dependency introduced it.
3. Check the upstream security advisory.
4. Identify a compatible fixed version.
5. Update the dependency declaration or dependency management.
6. Regenerate the lockfile or dependency resolution where applicable.
7. Run unit tests.
8. Run integration tests.
9. Run the dependency scan again.
10. Review application behavior.
11. Submit the pull request.
12. Obtain the required approvals.

For Java, dependency management may be controlled by the Spring Boot parent or BOM. A developer should not blindly override a transitive dependency without verifying compatibility.

### 9.3 Vulnerability severity policy

A practical starting policy:

| Severity | DEV    | QA     | STAGING | Production release     |
| -------- | ------ | ------ | ------- | ---------------------- |
| Critical | Block  | Block  | Block   | Block                  |
| High     | Block  | Block  | Block   | Block                  |
| Medium   | Review | Review | Review  | Approved risk decision |
| Low      | Report | Report | Report  | Report                 |

The organization should also consider exploitability, exposure, compensating controls, available fixes and the business impact of the affected component.

A severity score alone does not always describe the actual risk.

### 9.4 Exception handling

Suppose a high vulnerability has no compatible fix and the service must continue operating.

A security exception could be considered if the organization allows it.

The exception should contain:

```
apiVersion: security.company-ecommerce.io/v1
kind: SecurityException

metadata:
  id: SEC-EX-2026-001
  service: order-service

spec:
  finding:
    type: dependency
    identifier: "EXAMPLE-CVE"
    severity: HIGH

  justification: >
    A compatible fixed release is not currently available.
    The affected functionality is disabled by an approved
    compensating control.

  mitigation:
    - Restrict affected functionality
    - Monitor relevant application activity
    - Reassess upstream fix availability

  owner: order-team
  approver: application-security

  expires-on: "2026-11-30"

  status: approved
```

The identifier is illustrative, not a real vulnerability finding.

The policy evaluator must verify that the approval is genuine, the exception matches the actual finding, the service is in scope and the expiration date has not passed.

An exception for one service must not silently authorize the same finding across every microservice.

## 10. Container image security policy

The container image is the artifact that eventually runs inside EKS. It therefore needs its own security policy.

### 10.1 Approved image requirements

Every production image should satisfy the organization's approved baseline.

| Control            | Requirement                             |
| ------------------ | --------------------------------------- |
| Base image         | Approved and maintained base image      |
| Image tag          | Unique build identifier                 |
| Image digest       | Required for deployment                 |
| Vulnerability scan | Required                                |
| SBOM               | Generated and associated with the image |
| Signature          | Verified where required                 |
| Provenance         | Verified where required                 |
| Root user          | Avoid running as root                   |
| Secrets            | No embedded credentials                 |
| Package contents   | Minimize unnecessary packages           |
| Registry           | Approved ECR repository                 |
| Release evidence   | Retained                                |

### 10.2 Base image selection

Consider these two Dockerfile patterns.

Less desirable:

```
FROM ubuntu:latest

RUN apt-get update
RUN apt-get install -y curl wget vim net-tools
```

Problems:

- `latest` can resolve to different images over time.
- The image includes utilities that may not be needed at runtime.
- The package installation is not pinned.
- The resulting image may contain a larger attack surface.

A better approach uses a reviewed, maintained base image and pins the base image digest.

Illustrative Java Dockerfile:

```
FROM eclipse-temurin:21-jre@sha256:<APPROVED_BASE_IMAGE_DIGEST>

WORKDIR /app

COPY target/order-service.jar app.jar

RUN useradd --system --uid 10001 appuser

USER 10001

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

The digest is a placeholder. Verify the exact image and digest from the approved registry before use.

### 10.3 Why use a non-root user?

By default, some container images run the application as root.

If an application is compromised, root inside the container can increase the impact of the incident. Container isolation limits what that user can access, but running as non-root reduces unnecessary privilege.

The Kubernetes security policy should independently verify the runtime user rather than trusting only the Dockerfile.

### 10.4 Image tag policy

Example: `policies/container/image-tag-policy.yml`

```
apiVersion: security.company-ecommerce.io/v1
kind: ImagePolicy

metadata:
  name: image-identity-policy
  owner: platform-engineering

spec:
  allowed-registries:
    - "123456789012.dkr.ecr.ap-south-1.amazonaws.com"

  deployment:
    require-digest: true
    allow-latest-tag: false
    require-approved-registry: true

  metadata:
    require-source-commit: true
    require-build-evidence: true
```

The registry hostname is illustrative. Replace it with the actual approved registry.

### 10.5 ECR scanning

Configure ECR image scanning according to the registry's supported scanning capabilities and organizational requirements.

The security process should account for:

- Scan-on-push behavior.
- Continuous vulnerability reassessment where supported.
- Scan findings and severity.
- Image digest.
- Finding age.
- Remediation status.
- Release eligibility.

A successful scan at build time does not guarantee that an image remains free of known vulnerabilities. New vulnerabilities can be disclosed after publication.

The platform should therefore define how existing production images are reassessed and how newly discovered critical vulnerabilities are escalated.

### 10.6 ECR repository controls

Recommended controls:

- Private repositories by default.
- Repository access restricted through IAM.
- Image immutability enabled for release repositories where appropriate.
- Lifecycle policies that preserve supported release and rollback images.
- Encryption enabled.
- Image scanning configured.
- Cross-account access reviewed.
- Repository deletion restricted.
- Production image promotion controlled.

Do not create a lifecycle rule that deletes the previous known-good production image before the rollback retention period ends.

## 11. AWS IAM security policy

AWS IAM controls what CI workflows, platform services and workloads can do in the AWS account.

A policy error can have a large impact.

For example, giving a CI publishing role full administrator permissions is unnecessary when the role only needs to publish images to a small number of ECR repositories.

### 11.1 IAM policy principles

1. Grant only required actions.
2. Scope resources wherever the AWS service supports it.
3. Avoid wildcard permissions for sensitive operations.
4. Separate read-only, publishing and infrastructure roles.
5. Restrict trust policies.
6. Review role assumption paths.
7. Use temporary credentials.
8. Rotate or revoke compromised identities.
9. Monitor privileged role usage.
10. Review unused permissions.

### 11.2 Separate AWS roles

Example role design:

| IAM role                      | Purpose                                              |
| ----------------------------- | ---------------------------------------------------- |
| `github-order-ci-role`        | Read-only access required for approved CI operations |
| `github-ecr-publish-role`     | Publish images to approved ECR repositories          |
| `github-terraform-plan-role`  | Read infrastructure and state for a plan             |
| `github-terraform-apply-role` | Apply approved infrastructure changes                |
| `eks-order-service-role`      | Runtime AWS permissions for Order Service            |
| `argocd-platform-role`        | Required GitOps reconciliation permissions           |

These names are illustrative.

Do not reuse the ECR publishing role for Terraform or application runtime access.

### 11.3 Example ECR publishing IAM policy

A simplified policy example:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GetAuthorizationToken",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Sid": "PublishOrderServiceImage",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage"
      ],
      "Resource": "arn:aws:ecr:ap-south-1:123456789012:repository/order-service"
    }
  ]
}
```

This is an illustrative starting policy, not a complete AWS permission set for every ECR publishing implementation. The actual workflow may require additional read operations for manifest or image verification. Validate the required actions with the AWS API calls used by the pipeline.

### 11.4 Why `Resource: "*"` appears for ECR authorization

The ECR authorization-token action is not scoped to a specific repository resource in the same way as repository image operations.

The other actions can be scoped to the required repository.

This distinction is important: do not blindly replace every wildcard with an ARN, but do not leave all permissions unrestricted either.

## 12. GitHub OIDC security policy

GitHub OIDC lets a workflow obtain temporary AWS credentials without storing permanent access keys.

The security policy must restrict which GitHub identities are allowed to assume the AWS role.

### 12.1 Identity flow

```
GitHub Actions Job
       |
       v
Request OIDC Token
       |
       v
GitHub OIDC Provider
       |
       v
AWS STS AssumeRoleWithWebIdentity
       |
       v
IAM Trust Policy Evaluation
       |
       +---- Rejected
       |
       +---- Approved
                |
                v
        Temporary AWS Credentials
                |
                v
           ECR Operations
```

### 12.2 Trust policy example

An illustrative trust policy for a protected environment:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:company-ecommerce/order-service:environment:ecr-publish"
        }
      }
    }
  ]
}
```

The account ID is a placeholder.

This trust condition allows the intended repository and GitHub environment identity. The actual environment name and subject claim must match the identity token generated by the workflow.

### 12.3 Important security point

An AWS trust policy that accepts every repository in an organization creates a wider trust boundary than one that accepts only a specific repository and environment.

For high-risk operations, use the narrowest practical identity conditions.

### 12.4 OIDC failure response

If an unapproved branch tries to assume the publishing role:

1. AWS evaluates the token claims.
2. The trust policy rejects the identity.
3. The credentials action fails.
4. The workflow stops.
5. No image is published.
6. The failure is recorded for investigation.

Do not fix this by making the trust policy accept every branch.

## 13. Terraform security policy

Terraform can create or modify almost every major component in the AWS environment.

A weak Terraform policy can introduce serious risks.

### 13.1 High-risk Terraform configurations

| Risk                 | Example                                             |
| -------------------- | --------------------------------------------------- |
| Public storage       | Publicly accessible S3 bucket                       |
| Public database      | RDS security group allows internet access           |
| Broad IAM            | Administrator access granted to an application role |
| Unrestricted network | Ingress from `0.0.0.0/0` on sensitive ports         |
| Missing encryption   | Sensitive storage without encryption                |
| Weak state security  | Terraform state exposed to unauthorized users       |
| Unsafe EKS           | Public endpoint or excessive cluster access         |
| Unprotected secrets  | Sensitive values committed to source                |
| Destructive changes  | Unexpected resource replacement                     |

### 13.2 Terraform policy file

Example: `policies/terraform/aws-security-policy.yml`

```
apiVersion: security.company-ecommerce.io/v1
kind: TerraformSecurityPolicy

metadata:
  name: aws-infrastructure-baseline
  owner: cloud-security

spec:
  storage:
    s3:
      require-encryption: true
      require-public-access-block: true

  database:
    rds:
      require-encryption: true
      require-private-subnet: true
      prohibit-public-access: true

  iam:
    prohibit-administrator-access-for-workloads: true
    require-specific-resource-scope: true

  network:
    prohibit-public-database-ingress: true
    require-reviewed-public-ingress: true

  state:
    require-remote-backend: true
    require-encryption: true
    restrict-access: true
```

Again, this is a custom policy definition. It must be translated into working checks, for example using Checkov, Terraform plan inspection or a policy engine.

### 13.3 Terraform validation pipeline

```
Pull Request
     |
     v
terraform fmt
     |
     v
terraform validate
     |
     v
Terraform security scan
     |
     v
Generate plan
     |
     v
Policy evaluation
     |
     +------ Fail ------> Block approval
     |
     v
Review plan
     |
     v
Infrastructure approval
     |
     v
Controlled apply
```

### 13.4 Example: public RDS exposure

A developer changes a database security group:

```
ingress {
  from_port   = 3306
  to_port     = 3306
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
```

This exposes the MySQL port to the internet, subject to other network and service controls.

The policy should reject this configuration.

The expected secure design is to allow only the required application or database administration network paths, using appropriately scoped security group references and private networking.

### 13.5 Infrastructure plan review

Not every infrastructure change has the same risk.

Examples:

- Adding a nonproduction S3 tag: low risk.
- Changing a security group: medium or high risk depending on exposure.
- Replacing an EKS node group: high operational risk.
- Deleting an RDS instance: critical risk.
- Changing IAM trust policies: high security risk.

The platform should use risk-based approval rules and retain the reviewed plan. Do not assume that `terraform apply` is safe merely because `terraform plan` returned exit code zero.

## 14. Kubernetes workload security policy

This is one of the most important parts of the platform because all 21 microservices eventually run inside EKS.

The objective is to ensure that every workload follows a common security baseline.

### 14.1 Pod Security Standards

Kubernetes defines three Pod Security Standards:

| Standard   | Description                                                                        |
| ---------- | ---------------------------------------------------------------------------------- |
| Privileged | Unrestricted; intended for workloads that genuinely need elevated privileges       |
| Baseline   | Prevents known privilege escalations while allowing common workload configurations |
| Restricted | Stronger security requirements for workloads                                       |

For ordinary e-commerce application services, the target should generally be the `restricted` profile, with documented exceptions for workloads that genuinely require additional capabilities.

### 14.2 Restricted workload requirements

The application workload should generally:

- Run as a non-root user.
- Avoid privileged containers.
- Drop unnecessary Linux capabilities.
- Use an appropriate seccomp profile.
- Avoid host PID, network and IPC namespaces.
- Avoid hostPath volumes unless specifically approved.
- Avoid privilege escalation.
- Use read-only root filesystems where the application supports them.
- Define CPU and memory requests and limits.
- Use an approved ServiceAccount.
- Pull images from approved registries.
- Use health probes.
- Avoid embedding credentials in environment values or images.

### 14.3 Example Kubernetes deployment

File: `order-service/deploy/templates/deployment.yaml`

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: order-service
  namespace: ecommerce

spec:
  replicas: 3

  selector:
    matchLabels:
      app: order-service

  template:
    metadata:
      labels:
        app: order-service

    spec:
      serviceAccountName: order-service

      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: order-service

          image: example.invalid/order-service@sha256:<VERIFIED_DIGEST>

          ports:
            - containerPort: 8080

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL

          resources:
            requests:
              cpu: 250m
              memory: 512Mi

            limits:
              cpu: "1"
              memory: 1Gi

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080

          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
```

The registry and digest are placeholders. Replace them with the verified ECR image reference.

This manifest also assumes the application is compatible with a read-only root filesystem. Some applications need explicitly configured writable temporary directories or volumes.

### 14.4 Why `allowPrivilegeEscalation: false`?

Without this control, a process may be able to gain additional privileges through certain executable configurations.

Setting it to false reduces that risk.

### 14.5 Why drop capabilities?

Linux capabilities divide some traditionally root-only privileges into smaller permissions.

Most ordinary web application containers do not need all capabilities.

```
capabilities:
  drop:
    - ALL
```

The workload should receive only specifically approved capabilities if an application genuinely needs them.

### 14.6 Why use seccomp?

Seccomp limits the system calls a process can make.

For standard workloads:

```
seccompProfile:
  type: RuntimeDefault
```

This uses the container runtime's default seccomp profile.

The restricted Pod Security Standard requires an approved seccomp configuration. The organization should test its EKS runtime and workload compatibility before enforcement.

## 15. Pod Security Admission policy

The Kubernetes Pod Security Admission controller can enforce Pod Security Standards at namespace level.

For the `ecommerce` namespace, an example is:

```
apiVersion: v1
kind: Namespace

metadata:
  name: ecommerce

  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 15.1 What do these three labels mean?

| Mode      | Behavior                                                  |
| --------- | --------------------------------------------------------- |
| `enforce` | Rejects noncompliant Pod creation                         |
| `audit`   | Records policy violations in audit information            |
| `warn`    | Sends warnings to users submitting noncompliant resources |

You can use the same profile in all three modes, as shown above.

For a safe rollout, start with audit and warning modes, examine the violations, fix workloads, and then enable enforcement according to the approved rollout plan.

### 15.2 What happens when a developer deploys a privileged container?

Suppose a Deployment contains:

```
securityContext:
  privileged: true
```

If the namespace enforces the restricted profile, Kubernetes rejects the resulting Pod request because the configuration violates the standard.

The deployment may show a failure, and the developer must correct the manifest or obtain an approved exception through the organization's policy process.

### 15.3 Important: Deployment versus Pod validation

A Deployment object is not itself a Pod.

The admission mechanism evaluates the resources involved in the request. A Deployment may be accepted while its controller-created Pod is later rejected.

Therefore, validate rendered manifests in CI and monitor admission failures in the cluster. Do not assume that accepting the Deployment object proves that its Pods can start.

## 16. Kubernetes policy-as-code with Kyverno

For custom cluster-level policy enforcement, Kyverno is one possible option.

Kyverno can validate, mutate and generate Kubernetes resources according to policy definitions.

It can enforce controls such as:

- Approved image registries.
- Required security contexts.
- Required resource limits.
- Required labels.
- Restrictions on privileged workloads.
- Required ServiceAccount configuration.

This project can use Kyverno as an additional admission control layer if the platform team chooses it.

Do not deploy Kyverno just because a policy example is present. The platform team must first evaluate its compatibility, operational requirements, version support and ownership.

### 16.1 Example: approved image registry

Illustrative Kyverno policy:

```
apiVersion: kyverno.io/v1
kind: ClusterPolicy

metadata:
  name: require-approved-image-registry

spec:
  validationFailureAction: Enforce

  background: true

  rules:
    - name: validate-image-registry

      match:
        any:
          - resources:
              kinds:
                - Pod

      validate:
        message: "Images must come from the approved ECR registry."

        foreach:
          - list: "request.object.spec.containers"

            deny:
              conditions:
                any:
                  - key: "{{ element.image }}"
                    operator: NotEquals
                    value: "123456789012.dkr.ecr.ap-south-1.amazonaws.com/{{ element.image | split(@, '/') | [1] }}"
```

This is a conceptual example, not a verified production Kyverno expression. The registry matching expression should be replaced with a tested policy using the syntax supported by the deployed Kyverno version. Test it against both approved and unapproved image references before enabling enforcement.

### 16.2 Example: require resource limits

A separate policy can require resource limits:

```
apiVersion: kyverno.io/v1
kind: ClusterPolicy

metadata:
  name: require-container-limits

spec:
  validationFailureAction: Enforce

  background: true

  rules:
    - name: require-memory-and-cpu-limits

      match:
        any:
          - resources:
              kinds:
                - Pod

      validate:
        message: "Every application container must define CPU and memory limits."

        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    cpu: "?*"
                    memory: "?*"
```

This example checks for resource limits on containers. Extend and test the policy for init containers, ephemeral containers, supported exceptions and the organization's resource standards.

### 16.3 Audit before enforce

For a new custom policy:

1. Deploy the policy in audit mode.
2. Collect existing violations.
3. Identify legitimate exceptions.
4. Fix application templates.
5. Test the policy with fixtures.
6. Review operational impact.
7. Enable enforcement.
8. Monitor policy rejections.

A policy that unexpectedly blocks system components can disrupt cluster operations. System namespaces and platform workloads need explicit policy planning.

## 17. Kubernetes ServiceAccount security policy

A Kubernetes ServiceAccount is the workload's Kubernetes identity.

It is not the same thing as an AWS IAM role, although EKS can associate the identities through supported workload identity mechanisms.

### 17.1 Avoid default ServiceAccount usage

Application workloads should normally use dedicated ServiceAccounts.

For example:

```
apiVersion: v1
kind: ServiceAccount

metadata:
  name: order-service
  namespace: ecommerce

automountServiceAccountToken: false
```

This disables automatic mounting of a Kubernetes API token into the Pod.

If an application does not need Kubernetes API access, this reduces unnecessary credentials inside the container.

### 17.2 When does a service need Kubernetes API access?

Most ordinary Spring Boot services do not need to create or modify Kubernetes resources.

If the Order Service only communicates with other services, MySQL and Kafka, it usually does not need Kubernetes API permissions.

A platform controller or automation component may need Kubernetes API access. That should be granted through a separately designed identity and RBAC policy.

### 17.3 AWS workload identity

If the application needs AWS access, configure a supported EKS workload identity mechanism, such as EKS Pod Identity or IAM Roles for Service Accounts (IRSA), based on the platform's chosen design.

The architecture should be:

```
Order Service Pod
       |
       v
Kubernetes ServiceAccount
       |
       v
EKS Workload Identity Integration
       |
       v
Dedicated AWS IAM Role
       |
       v
Approved AWS API Permissions
```

For example, if the Order Service must read a specific secret from AWS Secrets Manager, its IAM role should permit only the required secret operations and resources.

It should not receive broad access to all secrets in the account.

### 17.4 Example ServiceAccount policy

```
apiVersion: security.company-ecommerce.io/v1
kind: ServiceAccountPolicy

metadata:
  name: workload-identity-policy
  owner: cloud-security

spec:
  require-dedicated-serviceaccount: true

  require-default-token-disabled: true

  aws-identity:
    require-dedicated-role: true
    prohibit-administrator-role: true

  permissions:
    require-least-privilege: true
    prohibit-unapproved-wildcards: true
```

This is a custom policy declaration. Its implementation will be expanded in Chapter 190 — ServiceAccount Templates.

## 18. Kubernetes NetworkPolicy security policy

Network security is important because microservices should not automatically communicate with every other workload.

For example:

```
Order Service
    |
    +--> Payment Service
    +--> Inventory Service
    +--> Customer Service
    +--> Notification Service
    |
    X--> Unrelated internal services
```

The application should have only the required network paths.

### 18.1 Network policy principles

- Default-deny ingress where appropriate.
- Default-deny egress where operationally feasible.
- Explicitly allow required service-to-service communication.
- Permit DNS resolution to approved cluster DNS endpoints.
- Permit required database and messaging access.
- Restrict access to metadata endpoints and sensitive network ranges.
- Test policies against actual CNI and cluster behavior.
- Document exceptions.

### 18.2 Example default-deny policy

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: default-deny
  namespace: ecommerce

spec:
  podSelector: {}

  policyTypes:
    - Ingress
    - Egress
```

This blocks ingress and egress for selected Pods by default, subject to the behavior of the cluster's networking implementation.

Do not apply it blindly. Without additional allow policies, it can prevent DNS, database, broker and service communication.

### 18.3 Example Order Service ingress policy

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: order-service-ingress
  namespace: ecommerce

spec:
  podSelector:
    matchLabels:
      app: order-service

  policyTypes:
    - Ingress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway

      ports:
        - protocol: TCP
          port: 8080
```

This allows traffic from selected API Gateway Pods in the same namespace to the Order Service on TCP port 8080.

If the API Gateway is in another namespace, the policy needs an appropriate namespace selector as well.

### 18.4 Example DNS egress policy

A DNS policy must match the actual CoreDNS or cluster DNS deployment and its network path.

An illustrative policy:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-cluster-dns
  namespace: ecommerce

spec:
  podSelector: {}

  policyTypes:
    - Egress

  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system

      ports:
        - protocol: UDP
          port: 53

        - protocol: TCP
          port: 53
```

This example assumes the relevant DNS endpoints are selected by the namespace selector and port. Verify the actual CoreDNS deployment, service routing and CNI enforcement behavior before using it.

A default-deny policy without the required DNS and dependency rules can make every service appear unavailable.

## 19. Helm security policy

Helm is the standard packaging mechanism for the microservices. Security policies must validate the manifests produced by Helm, not only the source templates.

### 19.1 Required Helm controls

Every application chart should be checked for:

- Approved image repository.
- Immutable image digest for production.
- Non-root execution.
- Container privilege restrictions.
- Resource requests and limits.
- Health probes.
- Approved ServiceAccount.
- No embedded secrets.
- Appropriate ingress exposure.
- NetworkPolicy configuration.
- PodDisruptionBudget where required.
- HorizontalPodAutoscaler where appropriate.
- Environment-specific configuration.
- Valid Kubernetes API versions.

### 19.2 Example policy declaration

File: `policies/kubernetes/pod-security-policy.yml`

```
apiVersion: security.company-ecommerce.io/v1
kind: KubernetesWorkloadPolicy

metadata:
  name: ecommerce-workload-baseline
  owner: kubernetes-platform

spec:
  containers:
    require-non-root: true
    require-resource-limits: true
    require-resource-requests: true
    require-read-only-root-filesystem: true
    prohibit-privileged: true
    prohibit-privilege-escalation: true
    require-capability-drop-all: true
    require-seccomp-runtime-default: true

  images:
    approved-registry-only: true
    production-require-digest: true

  service-account:
    require-dedicated-identity: true
```

This policy should be implemented as checks in CI and admission enforcement where appropriate.

### 19.3 Validate every environment

The same chart may have different values in DEV, QA, STAGING and production.

For example:

```
deploy/
|
+-- chart/
|   +-- Chart.yaml
|   +-- templates/
|       +-- deployment.yaml
|       +-- service.yaml
|       +-- networkpolicy.yaml
|
+-- values/
    +-- dev.yaml
    +-- qa.yaml
    +-- staging.yaml
    +-- production.yaml
```

Run validation for each relevant values file:

```
helm lint deploy/chart \
  -f deploy/values/dev.yaml

helm lint deploy/chart \
  -f deploy/values/qa.yaml

helm lint deploy/chart \
  -f deploy/values/staging.yaml

helm lint deploy/chart \
  -f deploy/values/production.yaml
```

Then render the manifests and run the Kubernetes policy suite against the rendered results.

A chart passing validation with `dev.yaml` does not prove that `production.yaml` is safe.

## 20. Secrets management policy

Applications should not store production secrets in Git repositories, Dockerfiles, Helm values files or container images.

For this project, AWS Secrets Manager can be used as the central secrets store, with an approved EKS integration.

### 20.1 Example secret flow

```
AWS Secrets Manager
        |
        v
Approved EKS Secret Integration
        |
        v
Order Service Pod
        |
        v
Application reads required credential
```

The exact integration may use the AWS Secrets and Configuration Provider for the Secrets Store CSI Driver, an external secrets controller, or another approved approach.

Choose one supported pattern and document its ownership and lifecycle. Do not assume every EKS deployment automatically reads Secrets Manager.

### 20.2 Secrets policy

Example:

```
apiVersion: security.company-ecommerce.io/v1
kind: SecretsPolicy

metadata:
  name: application-secrets-baseline
  owner: cloud-security

spec:
  source:
    approved:
      - aws-secrets-manager

  prohibit:
    - plaintext-git-secrets
    - secrets-in-dockerfile
    - secrets-in-container-image
    - hardcoded-credentials
    - production-secrets-in-pull-request-logs

  access:
    require-workload-identity: true
    require-resource-scope: true

  lifecycle:
    require-rotation-policy: true
    require-revocation-procedure: true
```

This is a custom policy format. The enforcement mechanisms need to inspect source, workflow configuration, IAM policies and deployment resources.

### 20.3 Secret rotation

A production secret lifecycle should include:

1. Creation in the approved secrets store.
2. Access through the approved workload identity.
3. Rotation according to the credential type and business requirements.
4. Application refresh or restart behavior.
5. Validation after rotation.
6. Revocation of the old credential.
7. Audit evidence.

A rotation policy must account for application connection pools and dependent services. Changing a database password without coordinating application credential refresh can cause widespread connection failures.

## 21. Database security policy

The project has multiple services and databases. Database security must protect both infrastructure and data access.

### 21.1 RDS controls

Recommended baseline:

- Private database subnets.
- Encryption at rest.
- Encryption in transit where supported and configured.
- Restricted security groups.
- No public database exposure.
- Managed backups and retention.
- Appropriate database authentication.
- Auditing according to business and security requirements.
- Least-privilege database users.
- Separate credentials by service.
- Controlled database migrations.

### 21.2 Service-specific database identity

For example:

```
Order Service
    |
    v
order-service-db-user
    |
    v
Order Database
```

The Order Service should not use the master database account.

Its database identity should have only the required permissions on the relevant schema and tables.

### 21.3 Database policy declaration

```
apiVersion: security.company-ecommerce.io/v1
kind: DatabasePolicy

metadata:
  name: rds-security-baseline
  owner: database-platform

spec:
  network:
    private-subnets-required: true
    public-access-prohibited: true

  encryption:
    at-rest-required: true
    in-transit-required: true

  identity:
    prohibit-master-account-for-applications: true
    require-service-specific-users: true
    least-privilege-required: true

  backup:
    require-backup-policy: true
    require-recovery-testing: true
```

The actual policy must map these requirements to AWS RDS configuration, security groups, database grants and recovery procedures.

### 21.4 Database migration security

A database migration should:

- Be reviewed by the application or database owner.
- Be tested against representative data.
- Avoid unnecessary destructive operations.
- Preserve compatibility with the currently running application version.
- Have a recovery strategy.
- Be included in change evidence.

For a production application deployment, a successful image build does not authorize a destructive schema change.

## 22. Production release security policy

Production is the most sensitive environment. Its policies must enforce stronger controls than a developer's local environment.

### 22.1 Release eligibility

A release should not qualify for production if any mandatory requirement is missing.

Example: `policies/release/release-eligibility.yml`

```
apiVersion: security.company-ecommerce.io/v1
kind: ReleasePolicy

metadata:
  name: production-release-policy
  owner: release-governance

spec:
  environment: production

  required-evidence:
    - source-commit
    - build-result
    - unit-test-result
    - security-scan-result
    - image-digest
    - sbom-reference
    - gitops-change

  image:
    require-approved-registry: true
    require-verified-digest: true

  security:
    prohibit-unapproved-critical-findings: true
    prohibit-expired-exceptions: true

  approval:
    require-protected-environment: true
    require-authorized-approver: true
```

This declaration needs a release evaluator that verifies the actual evidence and approval identities. It is not enforced merely by saving the YAML file.

### 22.2 Production approval

Configure a protected GitHub Environment for production.

Example:

```
jobs:
  production-promotion:
    environment: production

    permissions:
      contents: read
```

Configure required reviewers and other environment protections in GitHub.

The workflow must also restrict which branches and tags can deploy to the environment.

A job referencing `environment: production` is not sufficient by itself unless the environment has been configured with the required protection rules.

### 22.3 Separation of duties

Where organizational policy requires separation of duties:

- The developer proposes the change.
- The CI system validates the change.
- Security controls evaluate the release.
- An authorized approver approves production promotion.
- GitOps reconciles the approved configuration.
- Deployment verification confirms the outcome.

A person who creates a change should not automatically be able to approve their own production release when the applicable policy prohibits it.

## 23. Security exception management

There will be cases where a finding cannot be fixed immediately.

The solution is a controlled exception process, not a permanent bypass.

### 23.1 Exception lifecycle

```
Finding Detected
       |
       v
Application Team Investigation
       |
       v
Remediation Available?
       |
    +--+--+
    |     |
   Yes    No
    |     |
    v     v
  Fix   Exception Request
    |     |
    |     v
    |  Security Review
    |     |
    |  +--+---+
    |  |      |
    | Approve Reject
    |  |      |
    |  v      v
    | Time-   Block
    | Limited Release
    | Exception
    |  |
    +--+
       |
       v
   Re-scan
       |
       v
 Release Eligibility
```

### 23.2 Exception requirements

Every exception must contain:

| Field         | Meaning                                        |
| ------------- | ---------------------------------------------- |
| Exception ID  | Unique reference                               |
| Finding ID    | Specific security finding                      |
| Service       | Affected service                               |
| Environment   | Where the exception applies                    |
| Severity      | Finding severity                               |
| Owner         | Accountable engineering owner                  |
| Approver      | Authorized security approver                   |
| Justification | Reason for temporary acceptance                |
| Mitigation    | Compensating control                           |
| Created date  | When the exception was approved                |
| Expiry date   | When the exception stops being valid           |
| Status        | Pending, approved, rejected, expired or closed |

### 23.3 Exception policy

Example: `policies/release/exception-policy.yml`

```
apiVersion: security.company-ecommerce.io/v1
kind: ExceptionPolicy

metadata:
  name: security-exception-baseline
  owner: application-security

spec:
  require:
    - unique-id
    - finding-id
    - service-owner
    - security-approver
    - justification
    - mitigation
    - expiry-date

  maximum-duration-days: 30

  prohibit:
    - permanent-exceptions
    - wildcard-service-exceptions
    - unapproved-exceptions
    - expired-exceptions
```

The 30-day limit is an example policy value. The security organization must determine the approved maximum duration based on severity and risk.

### 23.4 Expired exception behavior

Suppose an exception expires on 30 November.

On 1 December:

1. The policy evaluator identifies the expired exception.
2. It no longer authorizes the finding.
3. The next release evaluation treats the finding under the standard policy.
4. If the finding violates a mandatory release rule, promotion is blocked.
5. The owner must remediate or request a newly reviewed exception.

The pipeline must not treat the existence of an old exception file as perpetual approval.

## 24. Policy enforcement with OPA and Conftest

Now we need to turn policy definitions into actual checks.

One useful approach is Open Policy Agent (OPA) with Conftest.

- OPA is a policy engine.
- Rego is its policy language.
- Conftest can evaluate configuration files against Rego policies.
- CI workflows can fail when a policy violation is detected.

This is particularly useful for Terraform, Kubernetes manifests and Helm-rendered YAML.

### 24.1 Install Conftest in a controlled CI runner

For a local learning environment:

```
conftest --version
```

Use an approved installation method and pin the Conftest version in CI. Do not download an unverified binary during a privileged production workflow.

### 24.2 Directory structure

```
platform-security-policies/
|
+-- conftest/
|   |
|   +-- kubernetes/
|   |   +-- policy/
|   |   |   +-- main.rego
|   |   |
|   |   +-- fixtures/
|   |       +-- valid-deployment.yaml
|   |       +-- invalid-deployment.yaml
|   |
|   +-- terraform/
|       +-- policy/
|       +-- fixtures/
|
+-- tests/
```

### 24.3 Example Rego policy

File: `conftest/kubernetes/policy/main.rego`

```
package main

deny contains message if {
    input.kind == "Deployment"

    container := input.spec.template.spec.containers[_]

    not container.resources.limits

    message := sprintf(
        "Container %s must define resource limits",
        [container.name]
    )
}
```

This policy rejects a Deployment if any container has no `resources.limits`.

### 24.4 Valid Deployment fixture

File: `valid-deployment.yaml`

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: order-service

spec:
  replicas: 3

  selector:
    matchLabels:
      app: order-service

  template:
    metadata:
      labels:
        app: order-service

    spec:
      containers:
        - name: order-service
          image: example.invalid/order-service@sha256:<DIGEST>

          resources:
            requests:
              cpu: 250m
              memory: 512Mi

            limits:
              cpu: "1"
              memory: 1Gi
```

The digest is a placeholder for testing the policy structure.

### 24.5 Invalid Deployment fixture

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: order-service

spec:
  replicas: 3

  selector:
    matchLabels:
      app: order-service

  template:
    metadata:
      labels:
        app: order-service

    spec:
      containers:
        - name: order-service
          image: example.invalid/order-service:latest
```

This fixture should fail because it lacks resource limits.

### 24.6 Run the policy

```
conftest test \
  valid-deployment.yaml \
  -p conftest/kubernetes/policy/
```

Expected result: pass.

Now test the invalid fixture:

```
conftest test \
  invalid-deployment.yaml \
  -p conftest/kubernetes/policy/
```

Expected result: failure with a resource-limit violation.

The exact output depends on the installed Conftest version.

### 24.7 Why test both valid and invalid manifests?

If we only test invalid manifests, we may create a policy that rejects everything.

If we only test valid manifests, we may create a policy that allows unsafe configurations.

Both positive and negative tests are necessary.

## 25. Additional Kubernetes Rego policies

A single resource-limit check is not enough for a production workload.

The policy suite should be built from independently testable controls.

### 25.1 Prohibit privileged containers

```
package main

deny contains message if {
    input.kind == "Deployment"

    container := input.spec.template.spec.containers[_]

    container.securityContext.privileged == true

    message := sprintf(
        "Container %s must not run in privileged mode",
        [container.name]
    )
}
```

### 25.2 Require non-root execution

```
package main

deny contains message if {
    input.kind == "Deployment"

    container := input.spec.template.spec.containers[_]

    container.securityContext.runAsNonRoot != true

    message := sprintf(
        "Container %s must run as non-root",
        [container.name]
    )
}
```

A production policy should also consider Pod-level and container-level security context inheritance. The policy must correctly evaluate effective settings, not just one field.

### 25.3 Require read-only root filesystem

```
package main

deny contains message if {
    input.kind == "Deployment"

    container := input.spec.template.spec.containers[_]

    container.securityContext.readOnlyRootFilesystem != true

    message := sprintf(
        "Container %s must use a read-only root filesystem",
        [container.name]
    )
}
```

This policy needs a documented exception mechanism for applications that genuinely require writable paths. Prefer explicit writable volumes over making the entire root filesystem writable.

### 25.4 Require dropped capabilities

```
package main

deny contains message if {
    input.kind == "Deployment"

    container := input.spec.template.spec.containers[_]

    not contains(container.securityContext.capabilities.drop, "ALL")

    message := sprintf(
        "Container %s must drop all Linux capabilities",
        [container.name]
    )
}
```

This is an illustrative policy expression. Ensure the policy handles missing or null fields safely and is tested against your selected OPA version.

### 25.5 Prohibit latest image tag

```
package main

deny contains message if {
    input.kind == "Deployment"

    container := input.spec.template.spec.containers[_]

    endswith(container.image, ":latest")

    message := sprintf(
        "Container %s must not use the latest image tag",
        [container.name]
    )
}
```

This is a basic check. A complete production policy should require the correct approved registry and a digest reference, not simply reject the `latest` suffix.

## 26. Terraform policy-as-code

Terraform policy checks should inspect actual resource configuration and, where appropriate, the Terraform plan.

A source-code check may miss the final effect of variable interpolation, module composition or provider defaults.

### 26.1 Example: public S3 bucket check

A simplified Rego policy for Terraform plan JSON:

```
package main

deny contains message if {
    resource := input.resource_changes[_]

    resource.type == "aws_s3_bucket"

    resource.change.after.acl == "public-read"

    message := sprintf(
        "S3 bucket %s must not use public-read ACL",
        [resource.address]
    )
}
```

This example assumes the input is a Terraform plan JSON representation. A production policy must account for unknown values, resource types, provider versions, public-access-block resources and the organization's approved exceptions.

### 26.2 Generate Terraform plan JSON

After generating a reviewed plan:

```
terraform show -json tfplan > tfplan.json
```

Then evaluate it:

```
conftest test \
  tfplan.json \
  -p conftest/terraform/policy/
```

Do not upload a Terraform plan JSON to a public location. Plan files and their JSON representations can contain sensitive values.

### 26.3 Why plan-based checks are useful

Suppose a module creates a security group through several variable values.

The source file might look harmless:

```
module "database_security" {
  source = "../../modules/security-group"

  database_port = 3306
  allowed_cidrs = var.database_allowed_cidrs
}
```

But the environment variable might contain:

```
database_allowed_cidrs = ["0.0.0.0/0"]
```

A plan-based policy can evaluate the resulting infrastructure configuration, making it easier to detect the actual proposed exposure.

### 26.4 Policy checks do not replace review

A policy engine cannot understand every business requirement.

For example, a public endpoint may be intentionally required for an internet-facing application gateway, but not for an internal database.

The policy should distinguish approved exposure types, resource scopes and environment requirements.

## 27. Integrate Conftest into the reusable workflow

We now have the policy repository and testable policies. The next step is to connect them to Chapter 185's reusable workflows.

Example: `reusable-kubernetes-policy.yml`

```
name: Reusable Kubernetes Policy

on:
  workflow_call:
    inputs:
      manifest-directory:
        type: string
        required: true

permissions:
  contents: read

jobs:
  policy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout service repository
        uses: actions/checkout@<REVIEWED_FULL_COMMIT_SHA>

      - name: Checkout security policies
        uses: actions/checkout@<REVIEWED_FULL_COMMIT_SHA>
        with:
          repository: company-ecommerce/platform-security-policies
          ref: <REVIEWED_POLICY_COMMIT_SHA>
          path: security-policies

      - name: Validate manifests
        run: |
          set -euo pipefail

          test -d "${{ inputs.manifest-directory }}"

          find "${{ inputs.manifest-directory }}" \
            -type f \
            -name '*.yaml' \
            -o -name '*.yml'
```

This first portion checks out both repositories and validates that the expected directory exists. It does not yet execute Conftest.

Add an approved Conftest installation step, then run:

```
      - name: Run Kubernetes policy checks
        run: |
          set -euo pipefail

          conftest test \
            "${{ inputs.manifest-directory }}" \
            -p security-policies/conftest/kubernetes/policy/
```

The installation should be pinned and verified. The policy checkout should also be pinned to a reviewed commit, not a floating branch.

### 27.1 Calling from Order Service

```
jobs:
  kubernetes-policy:
    uses: company-ecommerce/platform-github-actions/.github/workflows/reusable-kubernetes-policy.yml@<REVIEWED_WORKFLOW_COMMIT_SHA>

    with:
      manifest-directory: rendered-manifests
```

In the actual pipeline, `rendered-manifests` must be produced by an earlier job and transferred using an artifact mechanism if the jobs run on separate runners. Jobs do not share a filesystem automatically.

### 27.2 Important workflow dependency

The correct sequence is:

```
Helm Validation
      |
      v
Render Helm Manifests
      |
      v
Upload Rendered Artifact
      |
      v
Kubernetes Policy Job
      |
      v
Release Eligibility
```

The policy job must evaluate the exact rendered configuration that will be promoted, not an unrelated directory or stale artifact.

## 28. Security policy rollout strategy

Introducing security controls across 21 microservices at once can create a large number of failures.

A policy might reject a workload because it lacks resource limits, uses a writable root filesystem or relies on an older Kubernetes configuration.

The correct approach is gradual enforcement.

### 28.1 Rollout stages

| Stage | Mode              | Action                                          |
| ----- | ----------------- | ----------------------------------------------- |
| 1     | Inventory         | Discover current violations                     |
| 2     | Audit             | Record violations without blocking              |
| 3     | Remediation       | Fix service templates and configurations        |
| 4     | Pilot enforcement | Enforce on selected services                    |
| 5     | Broad enforcement | Enforce across approved services                |
| 6     | Continuous review | Monitor findings, exceptions and policy changes |

### 28.2 Example rollout plan

```
Week 1
  |
  +--> Inventory 21 services
  +--> Identify current security posture
  +--> Select policy owners

Week 2
  |
  +--> Introduce audit policies
  +--> Collect Kubernetes violations
  +--> Collect Terraform violations

Week 3
  |
  +--> Fix Dockerfiles
  +--> Fix Helm templates
  +--> Fix IAM configurations
  +--> Fix workflow permissions

Week 4
  |
  +--> Enforce on Order Service
  +--> Enforce on Payment Service
  +--> Validate failure handling

Week 5
  |
  +--> Expand enforcement
  +--> Review exceptions
  +--> Verify production release gates
```

This is an example schedule, not a guaranteed implementation duration. The actual timeline depends on the number of findings and the remediation effort.

### 28.3 Why start with Payment and Order Services?

These are useful pilot candidates because they are central to the e-commerce transaction flow.

A policy failure in either service can be evaluated against realistic application requirements.

However, a pilot must not create an unsafe exception for payment processing simply to get a green pipeline. Any exception needs the same approval and evidence standards as other services.

## 29. Security policy testing framework

The policy repository needs its own automated CI pipeline.

Policies are code. A change to a policy can affect every service, so it must be tested before release.

### 29.1 Policy test structure

```
platform-security-policies/
|
+-- tests/
|   |
|   +-- repository/
|   |   +-- valid-branch-policy.yml
|   |   +-- invalid-branch-policy.yml
|   |
|   +-- kubernetes/
|   |   +-- valid-deployment.yaml
|   |   +-- invalid-privileged.yaml
|   |   +-- missing-resource-limits.yaml
|   |   +-- latest-image.yaml
|   |
|   +-- terraform/
|   |   +-- valid-s3-plan.json
|   |   +-- public-s3-plan.json
|   |   +-- public-rds-plan.json
|   |
|   +-- exceptions/
|       +-- valid-exception.yml
|       +-- expired-exception.yml
|       +-- missing-owner.yml
```

### 29.2 Positive tests

A positive test verifies that a compliant configuration is allowed.

For example:

```
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
```

The policy should accept a compliant workload, assuming all other required fields are present.

### 29.3 Negative tests

A negative test verifies that a known violation is rejected.

Example:

```
securityContext:
  privileged: true
```

Expected outcome:

```
POLICY FAILED

Rule: prohibit-privileged-container

Resource: order-service

Violation:
Container must not run in privileged mode.
```

### 29.4 Exception tests

The test suite should also prove that:

- An approved, unexpired exception works only within its scope.
- An expired exception is rejected.
- A pending exception is not treated as approved.
- An exception without an owner is invalid.
- An exception for one service does not apply to another.
- An exception cannot suppress a finding with a different identifier.

### 29.5 Regression testing

When the platform team changes a policy, rerun the test suite and a representative set of existing application manifests.

This prevents a new policy from unexpectedly blocking previously compliant services.

## 30. Policy change management

Security policies can affect the entire application platform. They need controlled releases.

### 30.1 Policy versioning

Example:

```
platform-security-policies/
|
+-- main
|
+-- release/v1
|
+-- release/v2
```

Service workflows should consume a reviewed policy revision.

Avoid automatically evaluating production releases against an unreviewed change to the default branch.

### 30.2 Policy change request

Every policy change should document:

```
Policy Change Request

Policy:
Owner:
Reason for change:

Affected services:
Affected environments:

Current behavior:
New behavior:

Security impact:
Operational impact:

Positive test results:
Negative test results:

Migration required:
Exception impact:

Approver:
Release version:
```

### 30.3 Breaking policy changes

For example, changing from audit to enforce may cause previously accepted workloads to be rejected.

Before doing this:

1. Identify all affected workloads.
2. Run the new policy in audit mode.
3. Review violations.
4. Notify service owners.
5. Provide remediation guidance.
6. Test representative workloads.
7. Define the enforcement date.
8. Apply the new policy.
9. Monitor rejected requests.

Do not introduce a breaking policy change directly during a production release without a reviewed rollout.

## 31. GitHub Actions security policy enforcement

We have already created reusable workflows in Chapter 185. Now we need to protect the workflow execution itself.

### 31.1 Required workflow policy

The organization should require approved workflows for production-bound branches.

A recommended baseline includes:

- Required CI workflow.
- Required secret scanning.
- Required dependency scanning.
- Required code-quality check.
- Required container scan for release artifacts.
- Required artifact verification.
- Required release eligibility check.
- Protected production environment.
- Restricted deployment branches.
- Minimal job permissions.

### 31.2 Example protected workflow

```
name: Protected Production Release

on:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  validate:
    uses: company-ecommerce/platform-github-actions/.github/workflows/reusable-java-ci.yml@<REVIEWED_WORKFLOW_SHA>

  security:
    needs: validate
    uses: company-ecommerce/platform-github-actions/.github/workflows/reusable-dependency-scan.yml@<REVIEWED_WORKFLOW_SHA>

  release:
    needs:
      - validate
      - security

    environment: production

    permissions:
      contents: read
      id-token: write

    uses: company-ecommerce/platform-github-actions/.github/workflows/reusable-release.yml@<REVIEWED_WORKFLOW_SHA>
```

This is a structural example. A reusable workflow called through `jobs.<job_id>.uses` has its own workflow contract. Inputs, secrets and outputs must be declared and passed correctly. The `release` job must also be implemented to check all mandatory evidence rather than relying solely on job dependencies.

### 31.3 Required status checks

GitHub branch protection should require the correct status checks from trusted workflows.

Avoid depending only on a generic check name that an untrusted workflow could imitate.

Use organization rulesets and workflow governance to ensure that the required checks originate from the intended workflow and repository context.

### 31.4 Prevent workflow bypass

The platform should review:

- Who can modify required workflows.
- Who can change repository rulesets.
- Who can bypass pull-request protections.
- Who can approve production environments.
- Who can publish new reusable workflow versions.
- Who can modify security policy versions.

A workflow is not a security boundary if developers can freely disable it.

## 32. Production incident: Critical vulnerability discovered after release

Let's simulate a realistic production scenario.

The Order Service is running in EKS.

A new vulnerability is disclosed in a dependency included in the current image.

The image passed its security scan two weeks earlier.

### 32.1 Incident flow

```
New Vulnerability Advisory
          |
          v
Security Intelligence / Scanner Update
          |
          v
Affected ECR Images Identified
          |
          v
Order Service Digest Matches
          |
          v
Security Risk Assessment
          |
          +--------------------+
          |                    |
          v                    v
Fix Available           No Fix Available
          |                    |
          v                    v
Patch Dependency       Temporary Risk Controls
          |                    |
          v                    v
Build New Image        Security Exception Review
          |                    |
          v                    v
Scan and Verify        Monitor and Reassess
          |                    |
          +---------+----------+
                    |
                    v
            Approved Release
                    |
                    v
             GitOps Promotion
                    |
                    v
             EKS Verification
```

### 32.2 Immediate investigation

The team should determine:

- Which services use the affected dependency?
- Which image digests contain it?
- Is the vulnerable component reachable?
- Is there a known exploit?
- Is the affected functionality exposed externally?
- Is a fixed version available?
- Is an immediate mitigation required?
- Are any security exceptions already active?

### 32.3 Response actions

If the vulnerability is exploitable and the service is exposed, the security team may require immediate mitigation.

Possible actions include:

- Block new releases containing the vulnerable dependency.
- Restrict affected functionality.
- Disable a vulnerable endpoint if safe.
- Apply network or access restrictions.
- Build and deploy a fixed image.
- Review logs for possible exploitation.
- Rotate affected credentials if compromise is suspected.
- Open an incident and document the response.

The correct response depends on the vulnerability, exploitability and application architecture.

### 32.4 Why continuous image reassessment matters

The original CI scan was correct for the information available at the time.

However, security knowledge changes.

Therefore, the platform must reassess deployed image digests when new vulnerability intelligence becomes available, not only scan new builds.

## 33. Production incident: Developer bypasses security workflow

Scenario:

A developer modifies the Order Service workflow.

The original workflow calls:

```
jobs:
  security:
    uses: company-ecommerce/platform-github-actions/.github/workflows/reusable-container-scan.yml@<REVIEWED_WORKFLOW_SHA>
```

The developer replaces it with:

```
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Security scan passed"
```

If repository protection is weak, the check could appear green without a real scan.

### 33.1 Expected security controls

- Workflow files are protected through CODEOWNERS and branch rules.
- Required checks are configured centrally where possible.
- Production release eligibility independently verifies required scan evidence.
- The release workflow checks the exact image digest.
- Production environment permissions are restricted.
- Workflow changes receive platform/security review.

### 33.2 Incident response

1. Block the unauthorized merge or deployment if still possible.
2. Identify the affected workflow revision.
3. Determine whether a release was produced.
4. Verify whether a real security scan was performed.
5. Identify any image already deployed.
6. Run the missing checks.
7. Review the actor and change history.
8. Restore the approved workflow.
9. Record the incident.
10. Add preventive controls if a bypass was possible.

A green status message is not evidence that a security control actually ran.

## 34. Audit and compliance evidence

The security platform must retain evidence that controls operated as intended.

For each production release, collect:

| Evidence              | Purpose                                   |
| --------------------- | ----------------------------------------- |
| Source commit         | Identifies the source revision            |
| Workflow revision     | Identifies the pipeline implementation    |
| Policy revision       | Identifies the rules applied              |
| Test report           | Shows application validation              |
| SAST report           | Shows source security analysis            |
| Dependency report     | Shows dependency findings                 |
| Secret scan report    | Shows secret detection results            |
| Container scan report | Shows image findings                      |
| Image digest          | Identifies the deployed artifact          |
| SBOM                  | Lists included software components        |
| Signature/provenance  | Establishes artifact integrity and origin |
| Exception records     | Shows accepted risks and approvals        |
| GitOps commit         | Identifies desired deployment state       |
| Production approval   | Shows authorization                       |
| Deployment result     | Shows rollout outcome                     |

### 34.1 Example evidence structure

```
release-evidence/
|
+-- order-service/
    |
    +-- release-2026-001/
        |
        +-- source.json
        +-- workflow.json
        +-- policy.json
        +-- unit-tests.xml
        +-- sonarqube-report.json
        +-- dependency-report.json
        +-- secret-scan.json
        +-- trivy-report.json
        +-- sbom.cdx.json
        +-- image-digest.txt
        +-- provenance.json
        +-- exception-records.json
        +-- gitops-commit.json
        +-- approval.json
        +-- deployment-verification.json
```

These filenames are examples. The storage location should be an approved artifact or evidence repository with suitable retention, encryption and access control.

### 34.2 Evidence must be traceable

An auditor or incident responder should be able to answer:

1. Which source commit produced this image?
2. Which workflow revision built it?
3. Which security policy revision was evaluated?
4. Which findings were present?
5. Were any exceptions active?
6. Who approved the release?
7. Which digest was deployed?
8. Which GitOps commit requested the deployment?
9. Did the deployment pass verification?

If those questions cannot be answered, the release evidence process needs improvement.

## 35. Security policy mapping across all 21 microservices

The same baseline policies should apply to all services, but some service-specific requirements will differ.

| #  | Microservice           | Security considerations                                            |
| -- | ---------------------- | ------------------------------------------------------------------ |
| 1  | Auth Service           | Authentication, token handling, password security, identity access |
| 2  | User Service           | Personal information, authorization and data access                |
| 3  | Customer Service       | Customer data privacy and database permissions                     |
| 4  | Catalog Service        | Input validation, API security and image handling                  |
| 5  | Product Service        | Product data integrity and API authorization                       |
| 6  | Inventory Service      | Inventory integrity and restricted write access                    |
| 7  | Pricing Service        | Pricing integrity and unauthorized modification prevention         |
| 8  | Promotion Service      | Coupon abuse, authorization and business-rule integrity            |
| 9  | Cart Service           | Session security and customer data isolation                       |
| 10 | Order Service          | Transaction integrity and inter-service authorization              |
| 11 | Payment Service        | Payment data protection, secrets and restricted network access     |
| 12 | Fraud Service          | Sensitive transaction data and access controls                     |
| 13 | Shipping Service       | Customer address protection and provider credentials               |
| 14 | Delivery Service       | Location and delivery information protection                       |
| 15 | Notification Service   | Email/SMS provider credentials and message abuse prevention        |
| 16 | Search Service         | Input validation, access control and resource protection           |
| 17 | Recommendation Service | Customer data access and model/data protection                     |
| 18 | Review Service         | Input validation, content abuse and authorization                  |
| 19 | Tax Service            | Financial data integrity and restricted updates                    |
| 20 | Invoice Service        | Invoice integrity and sensitive financial information              |
| 21 | Reporting Service      | Data access, export permissions and information leakage            |

This is a security planning map, not a claim that every service has the same data classification or runtime implementation. The application owners must confirm the actual data flows and requirements.

### 35.1 Central baseline and service-specific extensions

The recommended policy structure is:

```
Security Baseline
      |
      +--> Repository Policy
      |
      +--> Source Code Policy
      |
      +--> Dependency Policy
      |
      +--> Container Policy
      |
      +--> Kubernetes Policy
      |
      +--> AWS Policy
      |
      +--> Release Policy
      |
      v
Service-Specific Extensions
      |
      +--> Payment Service
      +--> Auth Service
      +--> Customer Service
      +--> Other Services
```

For example, Payment Service may require additional payment-data protections. Those requirements extend the common baseline; they should not weaken the common baseline.

## 36. Security policy troubleshooting runbooks

### 36.1 Kubernetes Pod rejected by admission policy

Symptoms

```
Error creating: pods "order-service" is forbidden
```

Investigation

```
kubectl describe deployment order-service \
  -n ecommerce
```

Inspect the ReplicaSet and Pod events:

```
kubectl get replicasets -n ecommerce

kubectl get events \
  -n ecommerce \
  --sort-by=.lastTimestamp
```

Common causes

- Privileged container.
- Running as root.
- Missing seccomp profile.
- Missing resource requirements.
- Unapproved image registry.
- Missing required security context.

Resolution

1. Identify the exact policy violation.
2. Check the Helm template.
3. Check the rendered manifest.
4. Update the appropriate platform template or service configuration.
5. Rerun CI policy checks.
6. Submit a reviewed GitOps change.
7. Verify the Pod is admitted.

Do not disable the cluster policy simply to get the deployment running.

### 36.2 Conftest policy fails

Symptoms

A policy check returns a non-zero exit status.

Investigation

```
conftest test rendered.yaml \
  -p conftest/kubernetes/policy/
```

Check:

- The input is valid YAML.
- The correct policy directory was loaded.
- The expected resource kind is being evaluated.
- The Rego rule matches the actual rendered structure.
- The test fixture reproduces the problem.

Resolution

Fix the manifest if it is noncompliant. If the policy is incorrect, update the policy through a reviewed change and test both positive and negative cases.

### 36.3 ECR image rejected by policy

Symptoms

A workload cannot deploy because the image does not meet the approved registry or digest requirements.

Investigation

```
kubectl get deployment order-service \
  -n ecommerce \
  -o yaml
```

Inspect the configured image:

```
kubectl get deployment order-service \
  -n ecommerce \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
```

Check the GitOps values and compare the image digest with the release evidence.

Resolution

Promote the verified image digest through GitOps. Do not substitute a mutable tag or bypass image verification.

### 36.4 AWS IAM policy blocks the workflow

Symptoms

```
AccessDenied
```

Investigation

- Identify the IAM role assumed by the job.
- Identify the denied AWS API action.
- Check the resource ARN.
- Review the trust policy.
- Review the permissions policy.
- Check the GitHub OIDC identity claims.

Resolution

Grant only the missing, justified permission to the appropriate role and resource scope. Do not attach `AdministratorAccess` as a troubleshooting shortcut.

### 36.5 Expired exception blocks production

Symptoms

A release that previously passed security review is now blocked.

Investigation

- Identify the exception ID.
- Confirm the expiration date.
- Verify the finding is still present.
- Check whether a fixed dependency or mitigation is available.

Resolution

Remediate the vulnerability or submit a new exception for approval if permitted. Do not change the system clock or modify the old approval record to bypass expiry.

### 36.6 Policy update blocks multiple services

Symptoms

Multiple microservices fail after a central policy update.

Investigation

1. Identify the new policy revision.
2. Compare the policy with the previous version.
3. Determine which rule introduced the failures.
4. Run positive and negative test fixtures.
5. Test representative service manifests.
6. Check whether the issue is an intended enforcement change or a policy defect.

Resolution

If the new policy is defective, revert or release a corrected policy through the governed process. If the violations are genuine, coordinate remediation with service owners and follow the rollout plan.

## 37. Security operations and periodic review

Security policy is not a one-time implementation.

The platform team should maintain a regular review process.

| Frequency                                | Activity                                                  |
| ---------------------------------------- | --------------------------------------------------------- |
| Every pull request                       | Run required source and configuration security checks     |
| Every image release                      | Scan and verify the exact artifact                        |
| Every production promotion               | Evaluate release eligibility and approvals                |
| Daily or according to scanner capability | Reassess known vulnerabilities                            |
| Weekly                                   | Review open critical and high findings                    |
| Monthly                                  | Review exceptions, policy violations and workflow changes |
| Quarterly                                | Review IAM roles, access paths and policy effectiveness   |
| After an incident                        | Update policies and test cases based on lessons learned   |

These are suggested cadences. Critical findings and security incidents require immediate handling rather than waiting for the next scheduled review.

### 37.1 Security dashboard

A useful platform dashboard should show:

- Critical and high vulnerabilities by service.
- Open security exceptions.
- Expired exceptions.
- Images without verified digests.
- Images missing required SBOMs.
- Workloads violating Kubernetes policies.
- Terraform policy failures.
- Repositories missing required checks.
- Services with outdated workflow versions.
- Production releases missing evidence.

This dashboard should be driven by actual scan and release records, not manually maintained status labels.

## 38. Practical implementation sequence

Implement Chapter 186 in this order.

### Stage 1 — Repository and governance

- Create `platform-security-policies`.
- Add README and policy ownership.
- Configure repository protection.
- Add CODEOWNERS.
- Define policy versioning.
- Define the exception approval process.

### Stage 2 — Source security

- Create repository security policy.
- Create secret scanning policy.
- Create dependency vulnerability policy.
- Create code-quality policy.
- Integrate the policies with Chapter 185 workflows.
- Test positive and negative cases.

### Stage 3 — Container security

- Create approved base-image requirements.
- Create image vulnerability policy.
- Create immutable image requirements.
- Configure ECR scanning.
- Define SBOM requirements.
- Define signature and provenance requirements.
- Verify the exact scanned and published digest.

### Stage 4 — Infrastructure security

- Create Terraform security policies.
- Add Checkov or another approved scanner.
- Add plan-based policy evaluation.
- Restrict public database exposure.
- Enforce S3 encryption and public-access controls.
- Review IAM wildcard permissions.
- Protect Terraform state.

### Stage 5 — Kubernetes security

- Create Pod Security baseline.
- Create ServiceAccount policy.
- Create NetworkPolicy requirements.
- Add Conftest policies.
- Evaluate Kyverno or another admission controller.
- Test namespace enforcement.
- Test application workload compatibility.

### Stage 6 — Production release security

- Create release eligibility policy.
- Create production approval policy.
- Configure protected environments.
- Implement exception validation.
- Verify GitOps promotion controls.
- Collect release evidence.
- Simulate policy failures.

### Stage 7 — Organization-wide rollout

- Pilot with Order Service.
- Pilot with Payment Service.
- Remediate findings.
- Expand to remaining microservices.
- Monitor policy rejection rates.
- Review and improve the policies.

## 39. Definition of Done

Chapter 186 is complete when the platform can demonstrate all of the following:

Repository security

- Protected branches and required reviews are configured.
- Workflow and security policy files have appropriate code owners.
- Developers cannot silently remove required security checks.

Application security

- Source, dependency and secret scanning run through approved workflows.
- Quality gates are evaluated correctly.
- Required findings block releases.
- Exceptions are approved, scoped and time-limited.

Container security

- The approved image is scanned.
- The exact digest is verified.
- The SBOM is associated with the image.
- Required signing and provenance verification are performed.
- Unapproved registries and mutable production image references are rejected.

Infrastructure security

- Terraform configurations are validated and scanned.
- Risky public access is rejected.
- IAM policies follow least privilege.
- Infrastructure changes have appropriate review and approval.

Kubernetes security

- Workloads follow the approved Pod Security baseline.
- ServiceAccounts are dedicated and appropriately scoped.
- NetworkPolicies restrict unnecessary communication.
- Admission policies reject noncompliant workloads.
- Helm-rendered manifests pass security validation.

Release security

- Missing security evidence blocks promotion.
- Expired exceptions do not authorize releases.
- Production approvals are protected.
- GitOps deploys the verified image digest.
- Release evidence is retained and traceable.

## 40. Connection to the next platform chapters

| Chapter                                 | How it connects to security policies                                    |
| --------------------------------------- | ----------------------------------------------------------------------- |
| 182 — Reusable Helm Templates           | Provides standardized secure Kubernetes workload templates              |
| 183 — Reusable GitHub Actions           | Provides common CI execution                                            |
| 184 — Terraform Modules                 | Provides reusable infrastructure components                             |
| 185 — GitHub Actions Reusable Workflows | Executes shared build, test, scan and release workflows                 |
| 186 — Security Policy Templates         | Defines required security controls                                      |
| 187 — Monitoring Templates              | Standardizes service and infrastructure monitoring                      |
| 188 — Alert Rule Templates              | Defines reusable alerting rules                                         |
| 189 — NetworkPolicy Templates           | Provides reusable network isolation                                     |
| 190 — ServiceAccount Templates          | Provides reusable workload identity configuration                       |
| 191 — GitOps Generators                 | Automates deployment configuration generation                           |
| 192 — Microservice Golden Path          | Combines the platform capabilities into a standard developer experience |

Next chapter — Chapter 187: Monitoring Templates

We will build reusable Prometheus and Grafana monitoring templates for all 21 microservices, including metrics endpoints, scrape configuration, dashboards, service-level indicators, Kubernetes infrastructure metrics, alert integration and production troubleshooting.

The key principle of Chapter 186:

Security must be an enforced platform capability, not a checklist that developers can choose to ignore.