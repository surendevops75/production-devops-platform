# 15-Trivy.md

# Trivy — Vulnerability, Misconfiguration, Secret & SBOM Scanning

Trivy is one of the core security tools in our enterprise e-commerce DevSecOps pipeline.

For the `order-service`, Trivy is used at multiple points:

```text
Developer Code
     │
     ▼
Git Repository
     │
     ├── Trivy Filesystem Scan
     │       ├── Dependencies
     │       ├── Secrets
     │       └── Vulnerabilities
     │
     ▼
Dockerfile
     │
     ├── Trivy Misconfiguration Scan
     │
     ▼
Docker Image
     │
     ├── OS packages
     ├── Application dependencies
     ├── Vulnerabilities
     └── SBOM
     │
     ▼
ECR
     │
     └── AWS ECR scanning
     │
     ▼
Production
```

The important point is:

> **We do not use Trivy only once. We scan different artifacts at different stages because each scan answers a different security question.**

---

# 1. Why Trivy?

In a production DevOps environment, simply compiling the application successfully does not mean it is secure.

For example:

```text
Maven build
    ↓
SUCCESS
```

The application may still contain:

* vulnerable Java dependencies
* vulnerable operating-system packages
* insecure Docker configuration
* leaked credentials
* vulnerable base images
* Kubernetes misconfigurations
* infrastructure-as-code problems

Trivy helps identify many of these issues before deployment.

---

# 2. Where Trivy Fits

Our CI/CD architecture is:

```text
GitHub / GitLab / Jenkins
          │
          ▼
       Checkout
          │
          ▼
       Build Code
          │
          ▼
      Unit Tests
          │
          ▼
       SonarQube
          │
          ▼
      Trivy FS Scan
          │
          ▼
    Secret Scanning
          │
          ▼
      Docker Build
          │
          ▼
    Trivy Image Scan
          │
          ▼
        SBOM
          │
          ▼
      Image Signing
          │
          ▼
          ECR
          │
          ▼
       GitOps Repo
          │
          ▼
        ArgoCD
          │
          ▼
         EKS
```

---

# 3. Trivy Scan Types

Trivy can be used for several different security checks.

For our project, the important ones are:

| Scan       | What it checks                            |
| ---------- | ----------------------------------------- |
| Filesystem | Application source/dependencies           |
| Image      | Container image                           |
| Config     | Docker/Kubernetes/Terraform configuration |
| Repository | Repository security                       |
| SBOM       | Software component inventory              |
| Kubernetes | Kubernetes resources                      |
| Secret     | Potential secrets                         |
| IaC        | Infrastructure configuration              |

---

# 4. Trivy Installation

For local development, install Trivy on the engineer's workstation.

Verify:

```bash
trivy --version
```

Example:

```text
Version: 0.x.x
```

The exact version should be standardized by the DevSecOps/platform team.

For production CI pipelines, avoid silently floating to arbitrary versions.

Use an approved version such as:

```text
TRIVY_VERSION=<approved-version>
```

and upgrade it deliberately.

---

# 5. First Scan

From the `order-service` repository:

```bash
trivy fs .
```

This tells Trivy to scan the filesystem.

Example:

```text
order-service/
├── src/
├── pom.xml
├── Dockerfile
├── deploy/
└── ...
```

Trivy examines files and detects supported vulnerabilities and configuration/security issues.

---

# 6. Filesystem Scan

The filesystem scan is particularly useful before building the Docker image.

Run:

```bash
trivy fs .
```

For CI we normally want a severity threshold:

```bash
trivy fs \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  --exit-code 1 \
  .
```

Meaning:

```text
--severity HIGH,CRITICAL
```

Only report high and critical findings.

```text
--ignore-unfixed
```

Ignore vulnerabilities for which no fix is currently available.

```text
--exit-code 1
```

Return exit code `1` when findings matching the policy exist.

This is extremely important in CI.

---

# 7. Why Exit Codes Matter

Suppose Trivy finds:

```text
CRITICAL vulnerability
```

If Trivy only prints the vulnerability but exits successfully:

```text
Pipeline
   ↓
Trivy
   ↓
Warning
   ↓
Pipeline SUCCESS
```

That does not enforce security.

Instead:

```text
Trivy
   ↓
CRITICAL vulnerability
   ↓
exit code 1
   ↓
CI job FAILED
   ↓
Deployment blocked
```

This turns scanning into an actual security gate.

---

# 8. Understanding Exit Code

Linux:

```bash
echo $?
```

Example:

```bash
trivy fs --severity HIGH,CRITICAL --exit-code 1 .

echo $?
```

If there are no policy-breaking findings:

```text
0
```

If there are findings:

```text
1
```

CI systems use this exit code to determine whether the job passed.

---

# 9. Dependency Scanning

Our Spring Boot application contains Maven dependencies.

Example:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

Dependencies themselves can contain vulnerabilities.

For example:

```text
order-service
     │
     ├── Spring Boot
     ├── Jackson
     ├── Hibernate
     ├── PostgreSQL driver
     └── other libraries
```

A vulnerable transitive dependency can exist even if developers never directly added it.

---

# 10. Transitive Dependencies

Suppose:

```text
order-service
    │
    └── library-A
          │
          └── library-B
                │
                └── vulnerable-library
```

The developer may never have explicitly declared:

```text
vulnerable-library
```

but it is still packaged into the application.

Dependency scanning helps detect this.

---

# 11. Scan Maven Project

From the project root:

```bash
trivy fs .
```

Trivy can inspect supported dependency manifests such as:

```text
pom.xml
package-lock.json
requirements.txt
go.mod
Gemfile.lock
```

depending on the application technology.

For our project:

```text
pom.xml
```

is especially important.

---

# 12. Scan Only Vulnerabilities

You can focus on vulnerabilities:

```bash
trivy fs \
  --scanners vuln \
  .
```

This is useful when configuration and secret checks are handled separately.

---

# 13. Vulnerability Severity

Trivy reports vulnerability severity.

Typical levels:

```text
UNKNOWN
LOW
MEDIUM
HIGH
CRITICAL
```

Our release policy should not simply be:

```text
CRITICAL = bad
HIGH = okay
```

Instead, we need to consider:

* exploitability
* whether a fix exists
* internet exposure
* application exposure
* runtime reachability
* compensating controls
* business impact
* vendor advisory
* whether the vulnerable component is actually used

---

# 14. Example Vulnerability

A scan may produce something conceptually like:

```text
Library       Version    Severity    Fixed Version
----------------------------------------------------
some-library  1.2.3      HIGH        1.2.5
```

The developer should:

```text
Identify dependency
       ↓
Check dependency tree
       ↓
Upgrade dependency
       ↓
Run tests
       ↓
Run Trivy again
       ↓
Merge PR
```

---

# 15. Fixing Maven Dependency Vulnerabilities

First inspect the dependency tree:

```bash
./mvnw dependency:tree
```

Search for a dependency:

```bash
./mvnw dependency:tree | grep jackson
```

Then determine which direct dependency brings it into the application.

Upgrade the appropriate dependency.

Then:

```bash
./mvnw clean verify
```

and:

```bash
trivy fs \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  .
```

---

# 16. Don't Blindly Upgrade Everything

A common beginner mistake is:

```text
Trivy finds vulnerability
        ↓
Upgrade everything
        ↓
Production breaks
```

Instead:

```text
Vulnerability
     ↓
Identify dependency
     ↓
Check fixed version
     ↓
Check compatibility
     ↓
Run unit tests
     ↓
Run integration tests
     ↓
Build image
     ↓
Scan image
     ↓
Deploy to DEV
     ↓
Validate
```

Security remediation must not bypass engineering discipline.

---

# 17. Dockerfile Misconfiguration Scanning

Our Dockerfile:

```dockerfile
FROM eclipse-temurin:21-jdk AS builder

WORKDIR /build

COPY .mvn/ .mvn/
COPY mvnw pom.xml ./

RUN chmod +x mvnw

RUN ./mvnw dependency:go-offline

COPY src ./src

RUN ./mvnw clean package -DskipTests

FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=builder /build/target/*.jar app.jar

EXPOSE 8080

USER 10001

ENTRYPOINT ["java", "-jar", "app.jar"]
```

Trivy can inspect configuration for security problems.

Run:

```bash
trivy config .
```

---

# 18. Why Dockerfile Scanning?

Consider an insecure Dockerfile:

```dockerfile
FROM ubuntu

USER root

RUN curl something | bash
```

Potential problems include:

* running as root
* untrusted image
* insecure installation practices
* missing security configuration

The application might compile successfully, but the container can still be insecure.

---

# 19. Dockerfile Best Practices

Our production Dockerfile should generally:

* use a trusted base image
* use a supported runtime
* use multi-stage builds
* minimize installed packages
* avoid secrets
* run as non-root
* avoid unnecessary tools
* use explicit versions/digests where appropriate
* minimize image size

For example:

```dockerfile
USER 10001
```

is preferable to:

```dockerfile
USER root
```

for the application runtime.

---

# 20. Scan Kubernetes Configuration

Our GitOps repository will eventually contain:

```text
gitops/
├── environments/
│   ├── dev/
│   ├── qa/
│   └── prod/
└── applications/
```

We can scan Kubernetes YAML:

```bash
trivy config gitops/
```

Example:

```text
Deployment
Service
Ingress
ConfigMap
ServiceAccount
HPA
PDB
NetworkPolicy
```

---

# 21. Kubernetes Misconfiguration Examples

Potential problems:

```yaml
securityContext:
  privileged: true
```

or:

```yaml
hostNetwork: true
```

or:

```yaml
containers:
  - name: order-service
    securityContext:
      runAsUser: 0
```

or:

```yaml
resources: {}
```

These configurations may create operational or security risks.

---

# 22. Kubernetes SecurityContext

Our application should eventually have something similar to:

```yaml
securityContext:
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
```

Container-level:

```yaml
containers:
  - name: order-service
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
          - ALL
```

Exact settings must be validated against the application/runtime requirements.

---

# 23. Secret Scanning

Trivy can also identify secrets in supported scans.

However, in our pipeline we are already using:

```text
Gitleaks
```

for dedicated secret scanning.

Therefore:

```text
Gitleaks
   +
Trivy
```

provides defense in depth.

---

# 24. Example of a Secret We Must Never Commit

Never commit:

```yaml
DB_PASSWORD: my-secret-password
```

or:

```bash
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
```

or:

```yaml
password: SuperSecretPassword
```

Production secrets belong in:

```text
AWS Secrets Manager
```

and are injected through an appropriate workload identity/secrets mechanism.

---

# 25. Repository Scan

You can scan the entire repository:

```bash
trivy repo .
```

Depending on Trivy version and supported scanners, repository scanning can inspect source/dependencies/configuration.

For CI, explicitly selecting scanners and policies is usually easier to reason about.

---

# 26. Container Image Scanning

After building:

```bash
docker build -t order-service:${GIT_SHA} .
```

scan the image:

```bash
trivy image order-service:${GIT_SHA}
```

Production CI:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  --exit-code 1 \
  order-service:${GIT_SHA}
```

---

# 27. What Image Scanning Finds

Suppose our image contains:

```text
Java application
       │
       ├── JRE
       ├── Linux packages
       ├── application JAR
       └── application dependencies
```

Trivy can identify vulnerabilities in supported components.

For example:

```text
Operating system package
Java library
Application dependency
Base image component
```

---

# 28. Why Scan Both Filesystem and Image?

They answer different questions.

### Filesystem

```text
Is the source/dependency tree vulnerable?
```

### Image

```text
What actually ended up inside the container?
```

Example:

```text
Source
  ↓
pom.xml
  ↓
Maven dependencies
  ↓
Docker build
  ↓
Final image
  ↓
OS packages + runtime + application
```

Therefore:

```text
FS scan
+
Image scan
```

is stronger than relying on only one.

---

# 29. Build Once, Scan Once

A major improvement over our initial CI foundation is:

> **Build the image once and scan that exact image.**

Avoid:

```text
Build image
   ↓
Scan image

Build image again
   ↓
Push image
```

because the second build could theoretically differ.

Preferred:

```text
Build image
     ↓
Scan exact image
     ↓
Generate SBOM
     ↓
Sign/attest
     ↓
Push exact image
```

This is an important production principle.

---

# 30. Image Digest

Docker tags are convenient:

```text
order-service:abc123
```

But the immutable identity of the image is its digest:

```text
order-service@sha256:<digest>
```

Conceptually:

```text
Tag
 │
 └── human-friendly reference

Digest
 │
 └── immutable content identity
```

Our GitOps deployment should eventually prefer digest-based promotion where practical.

---

# 31. Scan ECR Image

Once the image exists in ECR:

```bash
trivy image \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:${IMAGE_TAG}
```

Authentication is required for private ECR repositories.

Example:

```bash
aws ecr get-login-password \
  --region ap-south-1 |
docker login \
  --username AWS \
  --password-stdin \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

Then:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:${IMAGE_TAG}
```

---

# 32. ECR Scanning vs Trivy

Our architecture uses both.

```text
CI
 │
 └── Trivy
       ↓
      ECR
       ↓
AWS ECR scanning
```

They should not be considered exact substitutes.

Trivy gives us security checks integrated directly into CI.

ECR provides registry-side scanning and ongoing visibility.

---

# 33. Why Scan After Push?

A production registry can continuously contain images.

For example:

```text
Image pushed today
       ↓
No vulnerability detected
       ↓
New CVE published next month
       ↓
Existing image becomes vulnerable
```

Therefore image security is not just a one-time CI event.

This is why registry-side scanning and continuous vulnerability management are useful.

---

# 34. Severity Policy

A practical initial policy:

| Finding                  | CI                    |
| ------------------------ | --------------------- |
| Critical + fix available | ❌ Block               |
| High + fix available     | ❌ Block               |
| Critical + no fix        | ⚠️ Exception required |
| High + no fix            | ⚠️ Review             |
| Medium                   | Report/review         |
| Low                      | Informational         |

This is an example policy, not a universal standard.

The final policy belongs to:

```text
Security Team
+
Platform Team
+
Architecture
+
SRE
```

---

# 35. Why `--ignore-unfixed` Needs Care

This command:

```bash
--ignore-unfixed
```

means:

> Don't fail/report on vulnerabilities where no fixed version is available.

This can reduce noise.

But it does **not** mean:

> Unfixed vulnerabilities are safe.

For a critical vulnerability with no patch:

```text
No fix available
       ↓
Security review
       ↓
Determine exposure
       ↓
Compensating controls
       ↓
Document exception
       ↓
Revisit periodically
```

---

# 36. Vulnerability Exceptions

Sometimes a vulnerability cannot immediately be fixed.

Example:

```text
CVE
 ↓
No compatible vendor fix
 ↓
Application requires current dependency
 ↓
Risk accepted temporarily
```

Do not simply add:

```text
.trivyignore
```

and forget about it.

---

# 37. Good Exception Process

An exception should contain:

```text
CVE:
Affected component:
Reason:
Business impact:
Exploitability:
Compensating controls:
Owner:
Approved by:
Created date:
Expiration date:
```

Example:

```text
CVE: CVE-XXXX-YYYY
Component: example-library
Reason: No compatible fixed version
Compensating control: Service is private and network-restricted
Owner: Security Team
Expiration: 2026-12-31
```

Exceptions should expire.

---

# 38. `.trivyignore`

A repository can maintain approved exceptions using a Trivy ignore file.

Example:

```text
CVE-XXXX-YYYY
```

But this file must be protected through:

```text
Pull Request
   ↓
Security review
   ↓
Approval
   ↓
Merge
```

Do not allow developers to silently suppress vulnerabilities.

---

# 39. Dangerous Exception Pattern

Bad:

```text
Trivy failed
     ↓
Developer adds CVE to ignore
     ↓
Pipeline passes
```

Better:

```text
Trivy failed
     ↓
Investigate
     ↓
Determine exploitability
     ↓
Try remediation
     ↓
If impossible → documented exception
     ↓
Security approval
     ↓
Temporary suppression
```

---

# 40. Trivy Cache

Trivy downloads vulnerability databases.

Repeated CI jobs can become expensive and slow if every job starts from scratch.

Use caching where appropriate.

For example:

```text
Runner
  │
  └── Trivy DB cache
```

CI caching strategy should be designed according to the runner platform and security requirements.

---

# 41. GitHub Actions Integration

Our GitHub Actions workflow includes:

```yaml
trivy-fs:
  name: Trivy Filesystem Scan
  runs-on: ubuntu-latest

  steps:

    - name: Checkout source
      uses: actions/checkout@v4

    - name: Trivy filesystem scan
      uses: aquasecurity/trivy-action@<approved-version>
      with:
        scan-type: fs
        scan-ref: .
        severity: HIGH,CRITICAL
        ignore-unfixed: true
        exit-code: '1'
```

The exact action version should be pinned according to the organization's approved dependency policy.

---

# 42. GitHub Image Scan

After building the image:

```yaml
- name: Trivy image scan
  uses: aquasecurity/trivy-action@<approved-version>
  with:
    scan-type: image
    image-ref: order-service:${{ github.sha }}
    severity: HIGH,CRITICAL
    ignore-unfixed: true
    exit-code: '1'
```

Pipeline behavior:

```text
Trivy finds acceptable result
        ↓
exit 0
        ↓
continue
```

or:

```text
High/Critical violation
        ↓
exit 1
        ↓
job fails
        ↓
publish blocked
```

---

# 43. GitLab CI Integration

Our GitLab pipeline can use a Trivy container:

```yaml
trivy-fs:
  stage: security

  image:
    name: aquasec/trivy:<approved-version>
    entrypoint: [""]

  script:
    - trivy fs
      --severity HIGH,CRITICAL
      --ignore-unfixed
      --exit-code 1
      .
```

The pipeline fails if the configured policy is violated.

---

# 44. GitLab Image Scan

For an image scan:

```yaml
trivy-image:
  stage: security

  image:
    name: aquasec/trivy:<approved-version>
    entrypoint: [""]

  script:
    - trivy image
      --severity HIGH,CRITICAL
      --ignore-unfixed
      --exit-code 1
      ${APP_NAME}:${CI_COMMIT_SHA}
```

Remember the GitLab runner architecture must make the Docker image available to the scanning job.

A Docker image built in one isolated job does **not** automatically exist in another job.

---

# 45. Better GitLab Design

Instead of:

```text
docker-build job
      ↓
trivy-image job
      ↓
publish job
```

with an ephemeral local Docker daemon, use a controlled artifact strategy.

Possible approaches:

```text
Option A
Build + scan in same job
```

or:

```text
Option B

Build image
    ↓
Push temporary immutable image
    ↓
Scan registry image
    ↓
Promote same digest
```

The second model becomes particularly useful for large production pipelines.

---

# 46. Jenkins Integration

Our Jenkins pipeline contains:

```groovy
stage('Trivy Filesystem Scan') {
    steps {
        sh '''
            trivy fs \
              --severity HIGH,CRITICAL \
              --ignore-unfixed \
              --exit-code 1 \
              .
        '''
    }
}
```

Then:

```groovy
stage('Docker Build') {
    steps {
        sh '''
            docker build \
              -t ${APP_NAME}:${IMAGE_TAG} \
              .
        '''
    }
}
```

Then:

```groovy
stage('Trivy Image Scan') {
    steps {
        sh '''
            trivy image \
              --severity HIGH,CRITICAL \
              --ignore-unfixed \
              --exit-code 1 \
              ${APP_NAME}:${IMAGE_TAG}
        '''
    }
}
```

---

# 47. Jenkins Agent Requirements

If Jenkins runs Trivy and Docker scanning:

```text
Jenkins Controller
       │
       ▼
Jenkins Agent
       │
       ├── Docker
       ├── Trivy
       ├── Java
       └── Maven
```

The agent must have the required tools or use a controlled containerized build environment.

In larger organizations, we avoid manually modifying random Jenkins agents.

Instead:

```text
Approved Jenkins Agent Image
          │
          ├── Java
          ├── Maven
          ├── Docker tooling
          ├── Trivy
          └── security tooling
```

---

# 48. Trivy + SonarQube

These tools solve different problems.

```text
SonarQube
    │
    ├── Code quality
    ├── Bugs
    ├── Code smells
    ├── Maintainability
    └── Security rules

Trivy
    │
    ├── Dependencies
    ├── Container vulnerabilities
    ├── OS packages
    ├── Misconfiguration
    └── SBOM/security scanning
```

Therefore:

```text
SonarQube + Trivy
```

is stronger than using either alone.

---

# 49. Trivy + Gitleaks

Likewise:

```text
Gitleaks
    ↓
Dedicated Git secret detection

Trivy
    ↓
Broader vulnerability/configuration/security scanning
```

Example pipeline:

```text
Checkout
   │
   ├── Gitleaks
   │
   ├── SonarQube
   │
   └── Trivy
```

All are security gates.

---

# 50. SBOM

SBOM means:

> Software Bill of Materials.

It answers:

```text
What software components are inside this artifact?
```

Example:

```text
order-service
   │
   ├── Java runtime
   ├── Spring Boot
   ├── Hibernate
   ├── Jackson
   ├── PostgreSQL driver
   └── other libraries
```

---

# 51. Why SBOM Matters

Suppose a new critical CVE is announced:

```text
CVE
 ↓
Security team
```

The first question becomes:

> Which of our applications contain this component?

Without an SBOM:

```text
Search hundreds of repositories
```

With SBOM inventory:

```text
CVE
 ↓
Component
 ↓
Affected images
 ↓
Affected services
```

Much faster.

---

# 52. Generate SBOM with Trivy

For a filesystem:

```bash
trivy fs \
  --format cyclonedx \
  --output sbom.json \
  .
```

For an image:

```bash
trivy image \
  --format cyclonedx \
  --output sbom.json \
  order-service:${IMAGE_TAG}
```

The exact SBOM format should be standardized across the organization.

Common formats include:

```text
CycloneDX
SPDX
```

---

# 53. Store SBOM as a CI Artifact

Example GitHub Actions:

```yaml
- name: Generate SBOM
  run: |
    trivy image \
      --format cyclonedx \
      --output sbom.json \
      order-service:${{ github.sha }}

- name: Upload SBOM
  uses: actions/upload-artifact@v4
  with:
    name: order-service-sbom-${{ github.sha }}
    path: sbom.json
    retention-days: 30
```

For production, we may also associate SBOM information with the image in the registry/artifact system.

---

# 54. SBOM Flow

Our target architecture:

```text
Docker Build
     │
     ▼
Image
     │
     ├───────────────┐
     ▼               ▼
Trivy Scan         SBOM
     │               │
     ▼               ▼
Security Gate      Artifact
     │               │
     └───────┬───────┘
             ▼
          Signing
             │
             ▼
            ECR
```

---

# 55. Image Signing

A mature supply-chain pipeline should eventually establish:

```text
Who built this image?
```

and:

```text
Was this image modified?
```

This is where image signing and attestations come into the architecture.

Conceptually:

```text
Source
  ↓
Build
  ↓
Image
  ↓
Scan
  ↓
SBOM
  ↓
Sign
  ↓
ECR
```

Later we will implement modern signing/attestation tooling.

---

# 56. Complete Trivy Security Gate

For our `order-service`, the target CI flow is:

```text
Checkout
   │
   ▼
Build
   │
   ▼
Unit Tests
   │
   ▼
SonarQube
   │
   ▼
Gitleaks
   │
   ▼
Trivy FS
   │
   ▼
Docker Build
   │
   ▼
Trivy Image
   │
   ▼
Generate SBOM
   │
   ▼
Sign / Attest
   │
   ▼
Push ECR
```

---

# 57. Recommended Initial Commands

### Filesystem

```bash
trivy fs \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  --exit-code 1 \
  .
```

### Configuration

```bash
trivy config .
```

### Docker image

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  --exit-code 1 \
  order-service:${IMAGE_TAG}
```

### SBOM

```bash
trivy image \
  --format cyclonedx \
  --output sbom.json \
  order-service:${IMAGE_TAG}
```

---

# 58. Local Developer Workflow

Before creating a PR:

```bash
git checkout -b feature/order-validation
```

Then:

```bash
./mvnw clean verify
```

Run:

```bash
trivy fs \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  --exit-code 1 \
  .
```

Build:

```bash
docker build \
  -t order-service:local \
  .
```

Scan:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  --exit-code 1 \
  order-service:local
```

Then:

```bash
git add .
git commit -m "feat: add order validation"
git push origin feature/order-validation
```

Create PR.

---

# 59. Pull Request Flow

```text
Developer
    │
    ▼
Feature branch
    │
    ▼
Pull Request
    │
    ├── Build
    ├── Unit tests
    ├── SonarQube
    ├── Gitleaks
    ├── Trivy FS
    ├── Docker build
    └── Trivy image
          │
          ▼
     Security gates
          │
          ▼
       Review
          │
          ▼
        Merge
```

---

# 60. Production Branch Protection

`main` should not allow:

```text
Developer
   ↓
git push
   ↓
main
```

Instead:

```text
Developer
   ↓
Feature branch
   ↓
PR
   ↓
CI
   ↓
Security gates
   ↓
CODEOWNER review
   ↓
Approval
   ↓
Merge
```

A failed Trivy security gate should prevent the PR from being merged when the policy says the finding is blocking.

---

# 61. False Positives

Security scanners can sometimes produce findings that require investigation.

Never immediately assume:

```text
Trivy finding = exploitable production vulnerability
```

Instead investigate:

```text
CVE
 ↓
Package
 ↓
Version
 ↓
Dependency path
 ↓
Affected functionality
 ↓
Runtime exposure
 ↓
Exploitability
```

This is why DevSecOps is not simply:

> "Run scanner and fix everything."

---

# 62. Vulnerability Management Lifecycle

Production process:

```text
Detect
  ↓
Classify
  ↓
Prioritize
  ↓
Remediate
  ↓
Test
  ↓
Deploy
  ↓
Verify
```

If remediation is impossible:

```text
Detect
  ↓
Risk assessment
  ↓
Exception
  ↓
Compensating control
  ↓
Expiry
  ↓
Reassessment
```

---

# 63. Example Enterprise Security Policy

For our learning project:

```yaml
securityPolicy:
  critical:
    fixedAvailable: block

  high:
    fixedAvailable: block

  criticalUnfixed:
    action: security-review

  highUnfixed:
    action: risk-review

  medium:
    action: report

  low:
    action: informational
```

This is conceptual policy configuration rather than a native Trivy configuration file.

---

# 64. What Happens When Trivy Fails?

Example:

```text
Developer opens PR
        │
        ▼
CI starts
        │
        ▼
Build SUCCESS
        │
        ▼
Tests SUCCESS
        │
        ▼
SonarQube SUCCESS
        │
        ▼
Gitleaks SUCCESS
        │
        ▼
Trivy FS FAILED
        │
        ▼
PR cannot merge
```

Developer receives:

```text
Trivy detected HIGH vulnerability
```

Then:

```text
Investigate
   ↓
Upgrade dependency
   ↓
Commit fix
   ↓
Push
   ↓
CI reruns
```

---

# 65. What Happens When Image Scan Fails?

```text
Docker build
     ↓
SUCCESS
     ↓
Trivy image
     ↓
CRITICAL vulnerability
     ↓
FAIL
     ↓
No ECR publish
```

This is critical.

We don't want:

```text
Vulnerable image
      ↓
ECR
      ↓
GitOps
      ↓
ArgoCD
      ↓
EKS
```

The security gate must stop the artifact before promotion.

---

# 66. Important Production Principle

Never rely on:

```text
Developer remembers to run Trivy
```

Instead:

```text
Developer
    ↓
CI automatically
    ↓
Trivy
    ↓
Policy
    ↓
Pass / Fail
```

Security should be automated.

---

# 67. Complete Order-Service Security Pipeline

Our final conceptual pipeline becomes:

```text
                    ┌─────────────────┐
                    │   Developer     │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ GitHub/GitLab   │
                    │     /Jenkins    │
                    └────────┬────────┘
                             │
                             ▼
                       Checkout
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         Build/Test      Gitleaks       SonarQube
              │                             │
              └──────────────┬──────────────┘
                             ▼
                         Trivy FS
                             │
                             ▼
                        Docker Build
                             │
                             ▼
                       Trivy Image
                             │
                             ▼
                           SBOM
                             │
                             ▼
                     Sign / Attest
                             │
                             ▼
                            ECR
                             │
                             ▼
                         GitOps Repo
                             │
                             ▼
                           ArgoCD
                             │
                             ▼
                            EKS
```

---

# 68. Key DevOps Understanding

You should be able to explain this chain without memorizing commands:

```text
Why FS scan?
      ↓
Catch dependency/source issues early

Why config scan?
      ↓
Catch insecure Docker/Kubernetes/IaC configuration

Why image scan?
      ↓
Validate what actually goes into the container

Why SBOM?
      ↓
Know exactly what software exists in the artifact

Why ECR scanning?
      ↓
Continue vulnerability visibility after publication

Why exit code?
      ↓
Turn security findings into CI gates

Why exceptions?
      ↓
Handle vulnerabilities that cannot immediately be remediated

Why scan again later?
      ↓
New vulnerabilities can appear after the image was built
```

---

# 69. Interview/Production Scenario

### Scenario

A developer says:

> "Trivy says my image has a critical CVE, but the application doesn't use that functionality. Can we ignore it?"

Your response should be:

```text
First identify the affected component and dependency path.

Then determine whether the vulnerable functionality is actually reachable,
whether the component is exposed, whether a fixed version exists,
and whether compensating controls exist.

If the vulnerability is still considered acceptable,
we should create a documented, time-bound security exception
rather than simply suppressing the scanner result.
```

That is a production DevSecOps mindset.

---

# 70. Another Scenario

### Trivy passes today, but the same image is vulnerable next month.

Why?

Because:

```text
Today:
Image
 ↓
CVE database
 ↓
No known vulnerability
```

Later:

```text
New CVE published
       ↓
Same image
       ↓
Now vulnerable
```

Therefore:

```text
CI scan
+
Registry scanning
+
Continuous vulnerability management
```

are all valuable.

---

# 71. Common Mistakes

### Mistake 1

Only scanning source:

```bash
trivy fs .
```

and never scanning the final image.

### Mistake 2

Running:

```bash
trivy image ...
```

without:

```bash
--exit-code 1
```

and assuming the pipeline is protected.

### Mistake 3

Automatically ignoring every unfixed CVE.

### Mistake 4

Allowing developers to add arbitrary entries to `.trivyignore`.

### Mistake 5

Using only ECR scanning.

### Mistake 6

Using only Trivy and removing SonarQube/Gitleaks.

### Mistake 7

Rebuilding the image after scanning instead of promoting the exact scanned artifact.

### Mistake 8

Using mutable:

```text
latest
```

tags.

### Mistake 9

Ignoring SBOM generation.

### Mistake 10

Treating scanner output as automatically equal to exploitability.

---

# 72. Final Trivy Checklist

Before an image reaches production:

```text
[ ] Source checked
[ ] Maven build successful
[ ] Unit tests successful
[ ] SonarQube quality gate passed
[ ] Gitleaks passed
[ ] Trivy filesystem scan passed
[ ] Dockerfile/configuration scanned
[ ] Docker image built
[ ] Trivy image scan passed
[ ] SBOM generated
[ ] Vulnerability exceptions documented
[ ] Image signed/attested
[ ] Immutable image reference recorded
[ ] Image pushed to ECR
[ ] ECR scanning enabled
```

---

# 73. Final Architecture

For our enterprise e-commerce platform:

```text
                    SOURCE
                       │
                       ▼
              ┌─────────────────┐
              │     CI/CD       │
              └────────┬────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
      SonarQube     Gitleaks      Trivy
          │                         │
          │              ┌──────────┼───────────┐
          │              │          │           │
          │              ▼          ▼           ▼
          │             FS       Config       Image
          │              │          │           │
          └──────────────┴──────────┴───────────┘
                               │
                               ▼
                              SBOM
                               │
                               ▼
                         Sign / Attest
                               │
                               ▼
                              ECR
                               │
                               ▼
                           GitOps Repo
                               │
                               ▼
                             ArgoCD
                               │
                               ▼
                              EKS
```

The key production principle is:

> **Security scanning is not a separate activity after deployment. It is a set of automated gates throughout the software supply chain.**

---

## Next

The next chapter is:

**`16-ECR-Image-Versioning.md`**

There we'll go deeper into **ECR repository strategy, immutable tags, Git SHA vs semantic versions, image digests, lifecycle policies, build-once/push-once, image promotion across DEV → QA → PROD, and how the exact same image gets deployed to EKS without rebuilding it.**
