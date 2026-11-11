# 10-GitHub-Actions-CI.md

## 1. Purpose

In the previous chapter, we created the CI foundation.

Now we will build the **complete GitHub Actions CI pipeline** for our `order-service`.

The final pipeline will perform:

```text
Developer
   |
   v
Pull Request / Push
   |
   v
Checkout
   |
   v
Java + Maven
   |
   v
Compile + Unit Test
   |
   v
JaCoCo Coverage
   |
   v
SonarQube
   |
   v
Dependency Scan
   |
   v
Secret Scan
   |
   v
Docker Build
   |
   v
Trivy Filesystem Scan
   |
   v
Trivy Image Scan
   |
   v
Security / Quality Gate
   |
   v
AWS OIDC
   |
   v
ECR
   |
   v
Immutable Image
```

For pull requests:

```text
PR
 ↓
Validate
 ↓
Scan
 ↓
PASS/FAIL

No ECR push
```

For `main`:

```text
main
 ↓
Validate
 ↓
Scan
 ↓
Build Image
 ↓
ECR Push
```

The later CD pipeline will take the ECR image and deploy it through GitOps/Argo CD.

---

# 2. Final Repository Structure

Our application repository becomes:

```text
order-service/
│
├── src/
│   ├── main/
│   └── test/
│
├── .mvn/
│
├── pom.xml
├── mvnw
├── mvnw.cmd
│
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
│
├── README.md
├── .gitignore
│
├── .github/
│   ├── CODEOWNERS
│   │
│   └── workflows/
│       ├── ci.yaml
│       ├── security.yaml
│       └── dependency-review.yaml
│
├── .gitlab-ci.yml
│
└── Jenkinsfile
```

We are concentrating on:

```text
.github/workflows/ci.yaml
```

in this chapter.

---

# 3. CI Design

We don't want one giant sequential pipeline.

Instead:

```text
                    Checkout
                       |
                       v
                 Build + Test
                       |
           ┌───────────┼───────────┐
           |           |           |
           v           v           v
       SonarQube    Gitleaks   Dependency
                                Scan
           |           |           |
           └───────────┼───────────┘
                       |
                       v
                 Docker Build
                       |
                       v
                Trivy Image Scan
                       |
                       v
                  Security Gate
                       |
                       v
                   AWS OIDC
                       |
                       v
                      ECR
```

The exact parallelization can be optimized later.

---

# 4. PR vs Main

This is one of the most important parts.

## Pull Request

```text
feature/order-validation
          |
          v
       PR → main
          |
          v
       CI validation
          |
          ├── Build
          ├── Test
          ├── SonarQube
          ├── Dependency Scan
          ├── Gitleaks
          ├── Docker Build
          └── Trivy
```

If everything passes:

```text
PR → Approved → Merge
```

No image push.

---

## Main

After merge:

```text
main
 |
 v
CI
 |
 ├── Build
 ├── Test
 ├── Security
 ├── Docker
 ├── Trivy
 |
 v
AWS OIDC
 |
 v
ECR
```

This creates our deployable artifact.

---

# 5. GitHub Secrets and Variables

Before using SonarQube and AWS, configure GitHub repository settings.

Go to:

```text
Repository
   ↓
Settings
   ↓
Secrets and variables
   ↓
Actions
```

Create variables:

```text
AWS_REGION
SONAR_HOST_URL
AWS_ROLE_ARN
ECR_REPOSITORY
```

Example:

```text
AWS_REGION       = ap-south-1

SONAR_HOST_URL   = https://sonarqube.example.com

AWS_ROLE_ARN     = arn:aws:iam::123456789012:role/GitHubActionsOrderServiceECR

ECR_REPOSITORY   = order-service
```

Create secret:

```text
SONAR_TOKEN
```

### Important

Do not put:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

into GitHub secrets for this architecture.

We will use:

```text
GitHub OIDC
      ↓
AWS STS
      ↓
Temporary credentials
```

instead.

---

# 6. GitHub Token

GitHub automatically provides:

```text
GITHUB_TOKEN
```

for Actions.

We don't manually create it.

For a basic CI workflow:

```yaml
permissions:
  contents: read
```

is enough.

For AWS OIDC:

```yaml
permissions:
  id-token: write
  contents: read
```

will be required for the publishing job.

---

# 7. Workflow Skeleton

Create:

```text
.github/workflows/ci.yaml
```

Start with:

```yaml
name: Order Service CI

on:
  pull_request:
    branches:
      - main

  push:
    branches:
      - main

permissions:
  contents: read

concurrency:
  group: order-service-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

---

# 8. Build and Test Job

Add:

```yaml
jobs:

  build-test:

    name: Build and Test

    runs-on: ubuntu-latest

    timeout-minutes: 20

    steps:

      - name: Checkout source
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Verify Maven
        run: |
          chmod +x mvnw
          ./mvnw --version

      - name: Build and test
        run: |
          ./mvnw -B clean verify

      - name: Upload JAR
        uses: actions/upload-artifact@v4
        with:
          name: order-service-${{ github.sha }}
          path: target/*.jar
          retention-days: 7
```

---

# 9. Why `verify`?

Instead of:

```bash
./mvnw clean compile
./mvnw test
./mvnw package
```

we use:

```bash
./mvnw clean verify
```

Maven executes the lifecycle phases required to reach `verify`.

This gives us:

```text
clean
  ↓
compile
  ↓
test
  ↓
package
  ↓
verify
```

---

# 10. Test Reports

We should not only know:

```text
BUILD SUCCESS
```

We should collect test reports.

Add:

```yaml
      - name: Upload test reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-reports-${{ github.sha }}
          path: |
            target/surefire-reports/
            target/failsafe-reports/
          if-no-files-found: ignore
          retention-days: 7
```

Now if a test fails, the reports are still uploaded.

This is why:

```yaml
if: always()
```

is useful.

---

# 11. Test Flow

Suppose:

```text
OrderControllerTest
OrderServiceTest
OrderRepositoryTest
```

run successfully.

GitHub shows:

```text
Build and Test
     |
     ├── Compile       PASS
     ├── Unit Tests    PASS
     ├── Package       PASS
     └── Verify        PASS
```

If one test fails:

```text
Build and Test
     |
     ├── Compile       PASS
     ├── Unit Tests    FAIL
     └── Pipeline      FAIL
```

The later publishing stages must not execute.

---

# 12. JaCoCo Coverage

SonarQube becomes more useful when test coverage is available.

Add the JaCoCo Maven plugin to `pom.xml`.

Example:

```xml
<build>
    <plugins>

        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>

            <executions>

                <execution>
                    <goals>
                        <goal>prepare-agent</goal>
                    </goals>
                </execution>

                <execution>
                    <id>report</id>
                    <phase>verify</phase>
                    <goals>
                        <goal>report</goal>
                    </goals>
                </execution>

            </executions>
        </plugin>

    </plugins>
</build>
```

Use the organization-approved current plugin version through your dependency/version-management policy rather than blindly copying an old version.

After:

```bash
./mvnw clean verify
```

we should get:

```text
target/site/jacoco/
```

with:

```text
jacoco.xml
```

and an HTML report.

---

# 13. Upload Coverage

Add:

```yaml
      - name: Upload JaCoCo report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: jacoco-${{ github.sha }}
          path: target/site/jacoco/
          if-no-files-found: ignore
          retention-days: 7
```

Now we have:

```text
CI
 |
 ├── JAR
 ├── Test Reports
 └── JaCoCo Coverage
```

---

# 14. SonarQube

The next stage is code quality.

Architecture:

```text
GitHub Actions
      |
      v
Maven
      |
      v
SonarQube
      |
      ├── Bugs
      ├── Vulnerabilities
      ├── Code Smells
      ├── Duplications
      └── Coverage
```

Run:

```bash
./mvnw verify sonar:sonar
```

with:

```yaml
      - name: SonarQube analysis
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
        run: |
          ./mvnw -B verify sonar:sonar \
            -Dsonar.host.url="${SONAR_HOST_URL}" \
            -Dsonar.token="${SONAR_TOKEN}" \
            -Dsonar.qualitygate.wait=true
```

---

# 15. SonarQube Quality Gate

The important option is:

```text
-Dsonar.qualitygate.wait=true
```

The pipeline waits for the SonarQube quality gate.

Example:

```text
Code
 |
 v
SonarQube
 |
 ├── Bugs          0
 ├── Vulnerabilities 0
 ├── Coverage      82%
 ├── Duplications  2%
 |
 v
Quality Gate
 |
 ├── PASS → Continue
 └── FAIL → Stop
```

This prevents low-quality code from progressing.

We will build a proper SonarQube project and quality-gate policy in the dedicated SonarQube chapter.

---

# 16. Important SonarQube Networking Issue

If your SonarQube server is:

```text
https://sonarqube.internal.company
```

a GitHub-hosted runner may not be able to reach it.

Then:

```text
GitHub Runner
      |
      X
      |
Internal SonarQube
```

won't work.

You would need something such as:

```text
GitHub
   |
   v
Self-hosted GitHub Runner
   |
   v
Corporate Network
   |
   v
SonarQube
```

This is a very common enterprise consideration.

---

# 17. Secret Scanning with Gitleaks

We don't want developers accidentally committing:

```text
AWS_SECRET_ACCESS_KEY
DB_PASSWORD
API_KEY
PRIVATE_KEY
```

Use Gitleaks as an additional CI control.

Example:

```yaml
  secret-scan:

    name: Secret Scan

    runs-on: ubuntu-latest

    steps:

      - name: Checkout source
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

For production, pin third-party Actions to approved commit SHAs.

---

# 18. Dependency Review

For pull requests, we also want to detect dependency changes that introduce known vulnerabilities.

Create:

```text
.github/workflows/dependency-review.yaml
```

```yaml
name: Dependency Review

on:
  pull_request:
    branches:
      - main

permissions:
  contents: read

jobs:

  dependency-review:

    name: Dependency Review

    runs-on: ubuntu-latest

    steps:

      - name: Dependency review
        uses: actions/dependency-review-action@v4
```

This is specifically useful when a PR introduces or upgrades dependencies.

---

# 19. Why Dependency Review Is Different From SonarQube

They solve different problems.

### SonarQube

Looks mainly at:

```text
Source Code
```

Examples:

```text
bugs
code smells
security issues
duplication
coverage
```

### Dependency scanning

Looks at:

```text
Third-party libraries
```

Example:

```text
spring dependency
     ↓
known CVE
     ↓
vulnerable version
```

You need both.

---

# 20. Docker Build

After application validation:

```yaml
  docker:

    name: Build Container

    needs:
      - build-test

    runs-on: ubuntu-latest

    steps:

      - name: Checkout source
        uses: actions/checkout@v4

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: false
          load: true
          tags: |
            order-service:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

# 21. Why BuildKit?

Docker BuildKit provides improved:

* build performance
* caching
* parallelism
* build features

Our pipeline uses:

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

so GitHub Actions can reuse build layers.

For example:

```text
First build:

Dependencies
   ↓
Download
   ↓
Build
   ↓
20 minutes


Second build:

Cached dependencies
       ↓
Only changed layers
       ↓
Much faster
```

---

# 22. Trivy Filesystem Scan

Before scanning the final image, scan the repository itself.

Example:

```yaml
  trivy-fs:

    name: Trivy Filesystem Scan

    needs:
      - build-test

    runs-on: ubuntu-latest

    steps:

      - name: Checkout source
        uses: actions/checkout@v4

      - name: Run Trivy filesystem scan
        uses: aquasecurity/trivy-action@<approved-version>
        with:
          scan-type: fs
          scan-ref: .
          severity: HIGH,CRITICAL
          ignore-unfixed: true
          exit-code: '1'
```

The exact Trivy Action version should be pinned to the version approved by your organization.

---

# 23. Trivy Image Scan

Now scan:

```text
order-service:<git-sha>
```

Example:

```yaml
  trivy-image:

    name: Trivy Container Scan

    needs:
      - docker

    runs-on: ubuntu-latest

    steps:

      - name: Checkout source
        uses: actions/checkout@v4

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build image for scanning
        uses: docker/build-push-action@v6
        with:
          context: .
          load: true
          push: false
          tags: |
            order-service:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Trivy image scan
        uses: aquasecurity/trivy-action@<approved-version>
        with:
          scan-type: image
          image-ref: order-service:${{ github.sha }}
          severity: HIGH,CRITICAL
          ignore-unfixed: true
          exit-code: '1'
```

---

# 24. Why `exit-code: '1'`?

This is critical.

Without a failure condition:

```text
Trivy
  ↓
HIGH vulnerability
  ↓
Pipeline
  ↓
SUCCESS
```

That isn't a security gate.

With:

```yaml
exit-code: '1'
```

we get:

```text
Trivy
  ↓
HIGH/CRITICAL vulnerability
  ↓
Exit code 1
  ↓
GitHub job FAIL
  ↓
ECR push doesn't happen
```

---

# 25. Example

Suppose our image contains:

```text
openssl
glibc
curl
java
spring
```

Trivy finds:

```text
CRITICAL   CVE-XXXX
HIGH       CVE-YYYY
```

Pipeline:

```text
Docker Build
     |
     v
Trivy
     |
     X
FAIL
```

Therefore:

```text
No ECR push
No deployment
```

This is the desired behavior.

---

# 26. Security Gate

Our pipeline now becomes:

```text
Build
  |
  v
Test
  |
  v
SonarQube
  |
  v
Gitleaks
  |
  v
Dependency Review
  |
  v
Trivy FS
  |
  v
Docker Build
  |
  v
Trivy Image
  |
  v
SECURITY GATE
```

Only after these succeed can we publish.

---

# 27. AWS OIDC

Now we reach one of the most important parts.

We do **not** want:

```text
GitHub
   |
   ├── AWS_ACCESS_KEY_ID
   └── AWS_SECRET_ACCESS_KEY
```

Instead:

```text
GitHub Actions
      |
      | OIDC token
      v
AWS STS
      |
      | AssumeRoleWithWebIdentity
      v
IAM Role
      |
      v
Temporary AWS credentials
```

This is much safer than storing long-lived AWS access keys.

---

# 28. Publish Job

The ECR publishing job should depend on successful security checks.

```yaml
  publish:

    name: Publish Image

    needs:
      - build-test
      - secret-scan
      - trivy-fs
      - trivy-image

    if: >
      github.event_name == 'push' &&
      github.ref == 'refs/heads/main'

    runs-on: ubuntu-latest

    permissions:
      contents: read
      id-token: write

    timeout-minutes: 20

    steps:

      - name: Checkout source
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ steps.login-ecr.outputs.registry }}/${{ vars.ECR_REPOSITORY }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

There is one improvement we will make next: **build once and promote the exact same image**, rather than rebuilding after scanning.

---

# 29. Why AWS OIDC Works

The IAM trust policy we created earlier contains:

```text
GitHub repository
        |
        v
company-ecommerce/order-service
        |
        v
main branch
```

The role only trusts the expected GitHub identity.

Conceptually:

```text
Any GitHub repository
       X
       |
       |
company-ecommerce/order-service
       |
       v
      main
       |
       v
    IAM Role
```

This prevents an unrelated repository from assuming the role.

---

# 30. ECR Image

After publishing:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:<SHA>
```

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:
4f81e6d9c8a1...
```

Now we have:

```text
Git commit
     |
     v
Docker image
     |
     v
ECR
```

This creates traceability.

---

# 31. Image Traceability

Suppose production is running:

```text
order-service:4f81e6d9
```

We can determine:

```text
Image
  ↓
Git SHA
  ↓
Commit
  ↓
PR
  ↓
Developer
  ↓
Code change
```

This is extremely useful during production incidents.

For example:

> "Which source code is running in production?"

Answer:

```text
ECR image digest
       ↓
Git SHA
       ↓
GitHub commit
```

---

# 32. Build Once Improvement

Our current educational pipeline builds the image once for scanning and again for publishing.

That works, but there's a better production design.

```text
BADER
Build
 ↓
Scan
 ↓
Rebuild
 ↓
Push
```

Prefer:

```text
Build
 ↓
Scan exact artifact
 ↓
Publish exact artifact
```

One possible learning implementation is:

```text
Build Image
     |
     v
Docker Save
     |
     v
GitHub Artifact
     |
     v
Security Scan
     |
     v
Download Artifact
     |
     v
Docker Load
     |
     v
ECR Push
```

However, transferring large Docker tarballs through GitHub artifacts isn't ideal for a large enterprise.

A production implementation will generally move toward:

```text
Build
  ↓
Registry
  ↓
Scan
  ↓
Promote by digest
```

or use a controlled internal artifact/registry promotion model.

We'll cover this properly in:

```text
16-ECR-Image-Versioning.md
17-CI-Artifact-and-Image-Promotion.md
```

---

# 33. Complete Learning Pipeline

At this point the conceptual GitHub pipeline is:

```text
                         PR
                         |
                         v
                  ┌──────────────┐
                  │   Checkout   │
                  └──────┬───────┘
                         |
                         v
                   Java + Maven
                         |
                         v
                  Build + Tests
                         |
             ┌───────────┼───────────┐
             |           |           |
             v           v           v
         SonarQube    Gitleaks   Dependency
                                  Review
             |           |           |
             └───────────┼───────────┘
                         |
                         v
                   Trivy FS Scan
                         |
                         v
                    Docker Build
                         |
                         v
                   Trivy Image
                         |
                         v
                  Security Gate
                         |
                ┌────────┴────────┐
                |                 |
               PR               main
                |                 |
             Stop             AWS OIDC
                                  |
                                  v
                                 ECR
```

---

# 34. Complete `ci.yaml` Foundation

Putting the main pieces together:

```yaml
name: Order Service CI

on:
  pull_request:
    branches:
      - main

  push:
    branches:
      - main

permissions:
  contents: read

concurrency:
  group: order-service-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true


jobs:

  build-test:

    name: Build and Test

    runs-on: ubuntu-latest

    timeout-minutes: 20

    steps:

      - name: Checkout source
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Verify Maven
        run: |
          chmod +x mvnw
          ./mvnw --version

      - name: Build and test
        run: |
          ./mvnw -B clean verify

      - name: Upload JAR
        uses: actions/upload-artifact@v4
        with:
          name: order-service-${{ github.sha }}
          path: target/*.jar
          retention-days: 7

      - name: Upload test reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-reports-${{ github.sha }}
          path: |
            target/surefire-reports/
            target/failsafe-reports/
          if-no-files-found: ignore
          retention-days: 7

      - name: Upload JaCoCo report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: jacoco-${{ github.sha }}
          path: target/site/jacoco/
          if-no-files-found: ignore
          retention-days: 7


  secret-scan:

    name: Secret Scan

    runs-on: ubuntu-latest

    steps:

      - name: Checkout source
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}


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


  docker:

    name: Docker Build

    needs:
      - build-test

    runs-on: ubuntu-latest

    steps:

      - name: Checkout source
        uses: actions/checkout@v4

      - name: Setup Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: false
          load: true
          tags: |
            order-service:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max


  trivy-image:

    name: Trivy Container Scan

    needs:
      - docker

    runs-on: ubuntu-latest

    steps:

      - name: Checkout source
        uses: actions/checkout@v4

      - name: Setup Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: false
          load: true
          tags: |
            order-service:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Trivy image scan
        uses: aquasecurity/trivy-action@<approved-version>
        with:
          scan-type: image
          image-ref: order-service:${{ github.sha }}
          severity: HIGH,CRITICAL
          ignore-unfixed: true
          exit-code: '1'


  publish:

    name: Publish to ECR

    needs:
      - build-test
      - secret-scan
      - trivy-fs
      - trivy-image

    if: >
      github.event_name == 'push' &&
      github.ref == 'refs/heads/main'

    runs-on: ubuntu-latest

    permissions:
      contents: read
      id-token: write

    timeout-minutes: 20

    steps:

      - name: Checkout source
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ steps.login-ecr.outputs.registry }}/${{ vars.ECR_REPOSITORY }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

This is a **foundation**, not yet our final enterprise pipeline. In particular, we'll improve image promotion and add stronger supply-chain controls.

---

# 35. What Happens During a Real PR?

Developer changes:

```text
OrderService.java
```

Then:

```bash
git checkout -b feature/order-discount
git add .
git commit -m "feat: add order discount"
git push origin feature/order-discount
```

Creates:

```text
feature/order-discount
           |
           v
Pull Request
           |
           v
main
```

GitHub starts:

```text
Order Service CI
```

Pipeline:

```text
Build/Test       PASS
Secret Scan      PASS
Trivy FS         PASS
SonarQube        PASS
Docker Build     PASS
Trivy Image      PASS
```

Then GitHub reports:

```text
All required checks passed
```

Reviewer approves.

Then:

```text
PR → Merge
```

---

# 36. What Happens After Merge?

The merge creates a new commit on:

```text
main
```

That triggers CI again.

Now:

```text
Build
 ↓
Test
 ↓
Security
 ↓
Docker
 ↓
Trivy
 ↓
AWS OIDC
 ↓
ECR Push
```

Result:

```text
ECR
└── order-service
    └── 4f81e6d9c8a1...
```

---

# 37. What Does NOT Happen?

The CI pipeline does **not** execute:

```bash
kubectl apply
```

It does **not** directly deploy:

```text
EKS
```

It does **not** change:

```text
production namespace
```

Instead:

```text
CI
 ↓
ECR
```

and later:

```text
GitOps
 ↓
ArgoCD
 ↓
EKS
```

This separation will become very important when we build our production deployment architecture.

---

# 38. Failure Scenarios

### Unit test failure

```text
Build
 ↓
Test
 X
```

Result:

```text
Pipeline FAILED
```

No ECR push.

---

### SonarQube failure

```text
SonarQube
 ↓
Quality Gate
 X
```

Result:

```text
Pipeline FAILED
```

No ECR push.

---

### Secret detected

```text
Gitleaks
 ↓
Secret found
 X
```

Result:

```text
Pipeline FAILED
```

---

### Critical vulnerability

```text
Docker
 ↓
Trivy
 ↓
CRITICAL CVE
 ↓
FAIL
```

No ECR push.

---

### AWS authentication failure

```text
Security
 ↓
PASS
 ↓
OIDC
 ↓
AssumeRole
 X
```

Result:

```text
ECR push never happens
```

This is exactly what we want.

---

# 39. Operational Principle

Never design a CI pipeline where:

```text
Security Scan
       |
       v
Warning
       |
       v
Continue anyway
       |
       v
Production
```

unless there is a formally approved exception process.

Prefer:

```text
Security Scan
      |
      ├── PASS → Continue
      |
      └── FAIL → Stop
```

Exceptions should be explicit, documented and auditable.

---

# 40. What You Should Understand From This Chapter

Don't memorize the YAML.

Understand the flow:

```text
Developer
   ↓
Git
   ↓
GitHub Actions
   ↓
Maven
   ↓
Tests
   ↓
SonarQube
   ↓
Dependency Security
   ↓
Gitleaks
   ↓
Docker
   ↓
Trivy
   ↓
AWS OIDC
   ↓
ECR
```

And most importantly:

```text
Git SHA
   ↓
Container Image
   ↓
ECR
   ↓
Future GitOps Deployment
```

That is the foundation for the entire application delivery lifecycle.

---

# 41. Next Chapter

Next we move to:

## `11-GitLab-CI.md`

There we'll take the **same exact CI process** and implement it using GitLab:

```text
GitLab Repository
       |
       v
.gitlab-ci.yml
       |
       ├── GitLab Runner
       ├── Maven
       ├── SonarQube
       ├── Gitleaks
       ├── Trivy
       ├── Docker/BuildKit
       ├── AWS OIDC
       └── ECR
```

We'll go deeper into **GitLab Runner architecture**, because understanding the difference between **GitHub-hosted runners, GitLab runners, and Jenkins agents** is one of the most useful parts of this three-CI approach.
