# 13-CI-Security-and-Quality-Gates.md

## 1. Purpose

Our three CI systems are now established:

```text
GitHub Actions
GitLab CI
Jenkins
```

All three follow the same application delivery logic:

```text
Source Code
    ↓
Build
    ↓
Test
    ↓
Code Quality
    ↓
Security
    ↓
Docker Image
    ↓
Container Security
    ↓
ECR
```

Now we need to make the pipeline **security-aware**.

A production CI pipeline should not simply answer:

> "Did the application compile?"

It should answer:

> "Can this exact build safely progress to the next stage?"

---

# 2. Security Pipeline Architecture

Our target is:

```text
                         Developer
                             |
                             v
                        Pull Request
                             |
                             v
                    ┌─────────────────┐
                    │      Build      │
                    └────────┬────────┘
                             |
                             v
                       Unit Tests
                             |
             ┌───────────────┼────────────────┐
             |               |                |
             v               v                v
         SonarQube       Dependency        Gitleaks
                          Scanning
             |               |                |
             └───────────────┼────────────────┘
                             |
                             v
                       Docker Build
                             |
                             v
                      Trivy Image Scan
                             |
                             v
                            SBOM
                             |
                             v
                       Security Gate
                             |
                      ┌──────┴──────┐
                      |             |
                    FAIL           PASS
                      |             |
                      X             v
                               ECR Publish
```

---

# 3. Why Multiple Security Tools?

A common beginner mistake is:

```text
"We have Trivy, so security is covered."
```

It isn't.

Different tools inspect different layers.

| Tool               | Primary purpose                              |
| ------------------ | -------------------------------------------- |
| SonarQube          | Code quality + code security analysis        |
| Dependency scanner | Third-party dependency vulnerabilities       |
| Gitleaks           | Secrets accidentally committed to Git        |
| Trivy filesystem   | Filesystem/dependency/configuration scanning |
| Trivy image        | Container image vulnerabilities              |
| SBOM               | Software component inventory                 |
| Image signing      | Artifact authenticity/integrity              |
| AWS IAM            | Cloud authorization                          |
| ECR scanning       | Registry-side image security                 |

Security is **defense in depth**.

---

# 4. Security Layers

Think of the application as layers:

```text
Layer 1
Source Code
    ↓
Layer 2
Dependencies
    ↓
Layer 3
Dockerfile
    ↓
Layer 4
Container Image
    ↓
Layer 5
AWS/ECR
    ↓
Layer 6
Kubernetes
    ↓
Layer 7
Runtime
```

Each layer needs appropriate controls.

---

# 5. Source Code Security

Our Java application contains:

```text
Controller
Service
Repository
Entity
Configuration
```

Potential problems include:

```text
SQL injection
unsafe deserialization
hardcoded credentials
weak cryptography
path traversal
insecure validation
authorization mistakes
```

SonarQube helps identify many classes of source-code issues.

But:

> SonarQube is not a replacement for application security testing or secure code review.

---

# 6. Dependency Security

Our application depends on libraries:

```text
Spring Boot
Spring Web
Spring Data JPA
Hibernate
PostgreSQL Driver
Micrometer
JUnit
Lombok
```

Suppose:

```text
spring-library
     |
     v
Known CVE
     |
     v
Vulnerable version
```

Our own code may be perfectly written.

The application can still be vulnerable.

Therefore:

```text
Source Code Scan
       +
Dependency Scan
```

are separate controls.

---

# 7. Secret Security

A developer might accidentally commit:

```yaml
spring:
  datasource:
    username: admin
    password: MyProductionPassword
```

or:

```text
AWS_SECRET_ACCESS_KEY=...
```

This is a serious incident.

Our CI should detect it before merge.

```text
Git
 |
 v
Gitleaks
 |
 ├── No secret → Continue
 |
 └── Secret → FAIL
```

---

# 8. Container Security

Even if the application source is safe:

```text
Application
     ↓
Base Image
     ↓
OS Packages
     ↓
Libraries
```

can contain vulnerabilities.

Example:

```text
eclipse-temurin
      |
      ├── OS package A
      ├── OS package B
      └── library C
                 |
                 v
              CVE
```

Therefore we scan the final image.

---

# 9. Security Gate

All these controls should eventually produce a decision:

```text
             Security Results
                    |
       ┌────────────┼────────────┐
       ↓            ↓            ↓
   SonarQube     Gitleaks      Trivy
       |            |            |
       └────────────┼────────────┘
                    ↓
               Policy Engine
                    |
             ┌──────┴──────┐
             ↓             ↓
           PASS           FAIL
             |             |
             v             X
            ECR          STOP
```

---

# 10. Security Severity

Not every finding has the same impact.

Typical vulnerability classifications include:

```text
LOW
MEDIUM
HIGH
CRITICAL
```

Our initial learning policy:

```text
HIGH + CRITICAL
        ↓
Pipeline failure
```

But this policy needs refinement.

---

# 11. Why `ignore-unfixed`?

We previously used:

```text
--ignore-unfixed
```

This means vulnerabilities without an available fix can be excluded from the blocking result.

This can reduce false blocking, but it has a tradeoff.

Example:

```text
CRITICAL CVE
    |
    ├── Fixed version available
    │       ↓
    │     BLOCK
    │
    └── No fixed version
            ↓
       Policy decision
```

In a real company, the security team should define this policy.

Don't blindly copy:

```text
ignore-unfixed = true
```

and assume the system is secure.

---

# 12. Vulnerability Exceptions

Sometimes a vulnerability cannot immediately be fixed.

Example:

```text
Application
    |
    v
Library X
    |
    v
HIGH CVE
```

But:

```text
No compatible fixed release
```

The team may need an exception.

The wrong approach:

```text
Just add the CVE to ignore list
```

The better approach:

```text
CVE
 ↓
Risk assessment
 ↓
Business justification
 ↓
Owner
 ↓
Expiration date
 ↓
Compensating controls
 ↓
Approved exception
```

Exceptions should expire.

---

# 13. Never Create Permanent Security Blind Spots

Avoid:

```text
trivy ignore CVE-XXXX
forever
```

Instead:

```text
CVE-XXXX
 ↓
Exception ID
 ↓
Owner
 ↓
Reason
 ↓
Expiry
 ↓
Review
```

This is important in production environments.

---

# 14. SonarQube Quality Gate

The SonarQube gate should evaluate conditions such as:

```text
New bugs
New vulnerabilities
Code smells
Coverage
Duplications
Security hotspots
```

Conceptually:

```text
                SonarQube
                    |
        ┌───────────┼───────────┐
        ↓           ↓           ↓
      Bugs      Vulnerabilities Coverage
        |           |           |
        └───────────┼───────────┘
                    ↓
              Quality Gate
                    |
             ┌──────┴──────┐
             ↓             ↓
           PASS           FAIL
```

---

# 15. New Code vs Existing Code

This is an important SonarQube concept.

Suppose the existing project has:

```text
100 legacy issues
```

and we require:

```text
0 issues
```

for every build.

The team may never be able to modernize the application.

A better strategy is often:

```text
Existing technical debt
        +
New code quality gate
```

For example:

```text
Existing code
   |
   └── Managed separately

New code
   |
   ├── No new critical issues
   ├── Minimum coverage
   └── No new vulnerabilities
```

This lets teams improve legacy systems incrementally.

---

# 16. Dependency Scanning

For Maven:

```text
pom.xml
   |
   v
Dependency Tree
   |
   v
Security Database
   |
   v
Known vulnerabilities
```

Example dependency chain:

```text
order-service
   |
   v
Spring Boot
   |
   v
Spring Framework
   |
   v
transitive dependency
   |
   v
CVE
```

The important word is:

> **transitive dependency**

You may not explicitly declare a vulnerable library.

Another library may pull it in.

---

# 17. Dependency Tree

Use:

```bash
./mvnw dependency:tree
```

Example:

```text
com.company:order-service
 |
 ├── spring-boot-starter-web
 │      |
 │      ├── spring-web
 │      └── spring-core
 |
 └── spring-boot-starter-data-jpa
        |
        ├── hibernate
        └── spring-data-jpa
```

This helps understand where dependencies originate.

---

# 18. Secret Scanning Strategy

We should scan:

```text
Current commit
+
Repository history
```

because a developer could commit a secret and remove it in the next commit.

Example:

```text
Commit A
  |
  └── AWS secret
       |
Commit B
  |
  └── secret deleted
```

The secret still exists in Git history.

Therefore:

```text
Secret deleted
   ≠
Credential safe
```

The credential must be rotated/revoked.

---

# 19. Trivy Filesystem Scan

The filesystem scan happens before or around the container build.

Conceptually:

```text
Repository
    |
    ├── Dockerfile
    ├── pom.xml
    ├── application.yaml
    └── source
         |
         v
       Trivy
```

This can identify vulnerabilities/misconfigurations depending on enabled scanners.

---

# 20. Trivy Image Scan

After Docker build:

```text
Dockerfile
    |
    v
Docker Image
    |
    v
Trivy
    |
    v
OS + Libraries + Packages
```

Example:

```text
order-service:4f81e6d9
       |
       ├── OS packages
       ├── Java runtime
       ├── application dependencies
       └── application
```

---

# 21. ECR Scanning

We also enabled:

```hcl
image_scanning_configuration {
  scan_on_push = true
}
```

in our ECR Terraform configuration.

Therefore:

```text
CI
 |
 v
Trivy
 |
 v
PASS
 |
 v
ECR Push
 |
 v
ECR Scan
```

This gives us **two layers of container scanning**.

---

# 22. Why Scan Twice?

Because they serve different purposes.

CI scan:

```text
Developer feedback
before publication
```

ECR scan:

```text
Registry-side visibility
after publication
```

Architecture:

```text
             Docker Image
                  |
          ┌───────┴───────┐
          ↓               ↓
       Trivy            ECR Scan
       in CI            in registry
          |               |
          └───────┬───────┘
                  ↓
             Vulnerability
               visibility
```

They should not be treated as identical controls.

---

# 23. SBOM

Next important concept:

> **Software Bill of Materials**

An SBOM tells us what is inside our artifact.

Example:

```text
order-service
 |
 ├── Java 21
 ├── Spring Boot
 ├── Spring Web
 ├── Hibernate
 ├── PostgreSQL Driver
 ├── Jackson
 └── Other dependencies
```

This becomes extremely useful when a new vulnerability is announced.

---

# 24. Why SBOM Matters

Imagine tomorrow a critical vulnerability is announced in:

```text
library-X version 2.4.1
```

Without an SBOM:

```text
21 microservices
     |
     v
Search manually
```

With SBOMs:

```text
CVE
 ↓
Affected component
 ↓
SBOM inventory
 ↓
Affected services
```

We can quickly determine:

```text
auth-service      affected
order-service     affected
payment-service   not affected
catalog-service   affected
```

This is especially valuable for our 21-microservice architecture.

---

# 25. SBOM Formats

Common formats include:

```text
CycloneDX
SPDX
```

For our project we'll eventually generate an SBOM during CI and associate it with the image/artifact.

---

# 26. Image Signing

Another important supply-chain control is:

> **How do we know the image came from our CI system?**

Without signing:

```text
ECR
 |
 └── order-service:abc123
```

We know the tag.

But stronger supply-chain verification can use:

```text
Image
  |
  v
Digital Signature
  |
  v
Verified publisher
```

Later we can implement image signing using a modern signing/attestation approach such as Sigstore tooling.

---

# 27. Image Tag vs Digest vs Signature

These three are different.

### Tag

```text
order-service:1.5.0
```

Human-friendly.

### Digest

```text
sha256:abc123...
```

Immutable content identity.

### Signature

```text
"This image was signed by our trusted build identity."
```

Stronger provenance/authenticity.

Our eventual production deployment should rely heavily on:

```text
Digest
+
Provenance/attestation
+
Signature
```

rather than trusting only a mutable human-readable tag.

---

# 28. Supply Chain

Our CI supply chain becomes:

```text
Developer
   |
   v
Git Commit
   |
   v
CI
   |
   ├── Source Scan
   ├── Dependency Scan
   ├── Secret Scan
   ├── Build
   ├── Container Scan
   ├── SBOM
   └── Signing
   |
   v
ECR
```

Later:

```text
ECR
 |
 v
ArgoCD
 |
 v
EKS
```

This gives us an end-to-end software supply chain.

---

# 29. CI Security Policy

For our learning project, define:

```text
CRITICAL vulnerability
    → BLOCK

HIGH vulnerability with available fix
    → BLOCK

Medium
    → Report / policy-dependent

Low
    → Report

Secret detected
    → BLOCK

SonarQube quality gate failed
    → BLOCK

Unit test failed
    → BLOCK
```

This is a starting policy.

A real enterprise security policy should be based on risk, exploitability, exposure, compensating controls, and approved exceptions.

---

# 30. Security Decision Matrix

| Finding                | Initial policy    |
| ---------------------- | ----------------- |
| Unit test failure      | Block             |
| Build failure          | Block             |
| SonarQube quality gate | Block             |
| Secret detected        | Block             |
| Critical CVE           | Block             |
| High CVE with fix      | Block             |
| High CVE without fix   | Exception/review  |
| Medium CVE             | Report/policy     |
| Low CVE                | Report            |
| Missing SBOM           | Block for release |
| Failed image signing   | Block for release |

---

# 31. Pull Request Security

For:

```text
feature/order-discount
       |
       v
Pull Request
```

run:

```text
Build
Test
SonarQube
Dependency Review
Gitleaks
Trivy
Docker Build
```

Then:

```text
Security Gate
```

Only if it passes:

```text
PR
 ↓
Approval
 ↓
Merge
```

---

# 32. Main Branch Security

After merge:

```text
main
 |
 v
CI
 |
 ├── Build
 ├── Test
 ├── Quality
 ├── Security
 ├── Docker
 ├── Trivy
 ├── SBOM
 └── Sign
       |
       v
     ECR
```

This produces the trusted deployment artifact.

---

# 33. Don't Make Security Checks Optional by Accident

Avoid:

```yaml
continue-on-error: true
```

on critical security stages.

For example:

```yaml
- name: Trivy
  continue-on-error: true
```

would mean:

```text
Trivy FAIL
   |
   v
Pipeline continues
   |
   v
ECR
```

That defeats the purpose of the gate.

Use exceptions through a controlled process instead.

---

# 34. Security Findings Need Ownership

Suppose Trivy finds:

```text
CVE-2026-XXXX
```

The pipeline shouldn't simply become:

```text
FAILED
```

and leave everyone guessing.

A mature process identifies:

```text
Finding
   |
   v
Service owner
   |
   v
Severity
   |
   v
Fix version
   |
   v
Due date
   |
   v
Verification
```

For our project:

```text
order-service
     |
     └── backend-team
```

owns application-level fixes.

Platform/security teams help with platform-wide issues.

---

# 35. Security Responsibilities

Our organization model:

```text
backend-team
    |
    └── Application vulnerabilities

devops-team
    |
    └── CI/CD + AWS infrastructure

platform-team
    |
    └── Kubernetes/platform

security-team
    |
    └── Security policy + review

sre-team
    |
    └── Reliability + incident response
```

This creates clear ownership.

---

# 36. Common Real-World Failure

Imagine a developer says:

> "Trivy is failing because of the base image."

The wrong response:

```text
Disable Trivy.
```

The correct investigation:

```text
Which CVE?
     ↓
Which package?
     ↓
Which image layer?
     ↓
Is a fixed base image available?
     ↓
Can we upgrade?
     ↓
Does the application require the old version?
     ↓
Can we mitigate?
     ↓
If not, approved exception?
```

This is how production troubleshooting should work.

---

# 37. CI Security Architecture for 21 Services

We have:

```text
21 microservices
```

We don't want 21 completely different security policies.

Instead:

```text
                 Organization CI Policy
                         |
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
       GitHub          GitLab         Jenkins
          |              |              |
          └──────────────┼──────────────┘
                         |
                  Common Standards
                         |
        ┌────────────────┼────────────────┐
        ↓                ↓                ↓
    SonarQube          Trivy          Gitleaks
        |                |                |
        └────────────────┼────────────────┘
                         ↓
                        ECR
```

The implementation can differ.

The security policy should remain consistent.

---

# 38. Reusable CI Standards

Eventually we should avoid copying:

```text
50 lines
```

across:

```text
21 repositories
```

Instead, create reusable components.

### GitHub

```text
Reusable Workflow
```

### GitLab

```text
CI Template / include
```

### Jenkins

```text
Shared Library
```

Conceptually:

```text
             Common CI Standards
                    |
        ┌───────────┼───────────┐
        ↓           ↓           ↓
     GitHub       GitLab      Jenkins
    Workflow     Template     Library
        |           |           |
        └───────────┼───────────┘
                    |
                    v
              21 Microservices
```

We'll implement this later.

---

# 39. Security Pipeline Versioning

Security rules change.

Today:

```text
HIGH → block
```

Tomorrow:

```text
HIGH + exploitable → block
```

Therefore the security policy should be version controlled.

For example:

```text
platform/
└── ci-standards/
    ├── security-policy.yaml
    ├── trivy-policy.yaml
    ├── sonar-policy.yaml
    └── README.md
```

This is better than hidden configuration.

---

# 40. Production Security Gate

Our target is:

```text
                      CI
                       |
                       v
                 Build + Test
                       |
                       v
              ┌────────────────┐
              │ Security Checks│
              └───────┬────────┘
                      |
       ┌──────────────┼───────────────┐
       ↓              ↓               ↓
   SonarQube       Gitleaks        Dependency
                                    Scan
       |              |               |
       └──────────────┼───────────────┘
                      ↓
                 Docker Build
                      |
                      v
                   Trivy
                      |
                      v
                    SBOM
                      |
                      v
                   Signing
                      |
                      v
               Release Decision
                  /        \
                FAIL       PASS
                 |           |
                 X           v
                            ECR
```

---

# 41. What We Have Accomplished

Our CI architecture now understands:

```text
GitHub Actions
       |
GitLab CI
       |
Jenkins
       |
       v
Common CI policy
       |
       ├── Code Quality
       ├── Dependency Security
       ├── Secret Security
       ├── Container Security
       ├── SBOM
       └── Image Integrity
```

This is the correct foundation before moving deeper into individual tools.

---

# 42. Next Chapter: SonarQube

## `14-SonarQube.md`

We'll now configure SonarQube properly rather than just adding:

```bash
mvn sonar:sonar
```

We'll cover:

```text
SonarQube Architecture
        ↓
SonarQube Server
        ↓
Projects
        ↓
Project Key
        ↓
Quality Profiles
        ↓
Quality Gates
        ↓
Java Analysis
        ↓
JaCoCo Coverage
        ↓
Pull Request Analysis
        ↓
GitHub Integration
        ↓
GitLab Integration
        ↓
Jenkins Integration
        ↓
Authentication
        ↓
Webhook
        ↓
CI Quality Gate
        ↓
PASS / FAIL
```

We'll also connect the same SonarQube server to all three CI systems:

```text
GitHub Actions ──┐
GitLab CI ───────┼──> SonarQube
Jenkins ─────────┘
```

and use the `order-service` as our complete working example.
