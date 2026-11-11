# 11-GitLab-CI.md

## 1. Purpose

In the previous chapter, we implemented the CI pipeline using **GitHub Actions**.

Now we implement the **same CI lifecycle using GitLab CI/CD**.

Our goal is not to memorize `.gitlab-ci.yml`.

We want to understand:

* GitLab pipeline architecture
* GitLab Runner
* stages
* jobs
* `needs`
* artifacts
* cache
* variables
* protected variables
* rules
* merge request pipelines
* branch pipelines
* Docker builds
* Trivy
* SonarQube
* Gitleaks
* AWS authentication
* ECR
* pipeline failures
* reusable GitLab CI templates

---

# 2. GitLab CI Architecture

The basic architecture is:

```text
Developer
    |
    v
GitLab Repository
    |
    v
.gitlab-ci.yml
    |
    v
GitLab CI/CD
    |
    v
GitLab Runner
    |
    ├── Build
    ├── Test
    ├── Security
    ├── Docker
    └── Publish
    |
    v
AWS ECR
```

GitLab itself coordinates the pipeline.

The actual commands execute on a **GitLab Runner**.

---

# 3. GitLab Runner

This is an important distinction.

GitLab:

```text
GitLab
```

doesn't necessarily execute:

```bash
./mvnw clean verify
```

itself.

Instead:

```text
GitLab
   |
   | schedules job
   v
GitLab Runner
   |
   | executes commands
   v
./mvnw clean verify
```

The runner is the execution environment for the CI job.

---

# 4. GitLab Runner Types

A runner can be:

### GitLab-hosted

```text
GitLab
   |
   v
GitLab-hosted Runner
```

### Self-managed

```text
GitLab
   |
   v
Your Runner
   |
   ├── EC2
   ├── VM
   ├── Kubernetes
   └── On-prem server
```

For our enterprise architecture, we'll eventually use:

```text
GitLab
   |
   v
Kubernetes Runner
   |
   v
AWS / EKS
```

But initially we'll use a simpler runner so that we understand the pipeline first.

---

# 5. GitLab CI Configuration

Create this file in the `order-service` repository:

```text
.gitlab-ci.yml
```

GitLab automatically detects this file and uses it as the pipeline configuration.

---

# 6. Basic Pipeline

Start with:

```yaml
stages:
  - build
  - test
  - security
  - docker
  - publish
```

This creates:

```text
build
  ↓
test
  ↓
security
  ↓
docker
  ↓
publish
```

Jobs assigned to the same stage can execute in parallel.

For example:

```text
             build
               |
               v
          ┌────┴────┐
          ↓         ↓
        test      sonar
          ↓         ↓
          └────┬────┘
               ↓
             docker
```

Later we'll use `needs:` to create a more efficient DAG.

---

# 7. Global Variables

Add:

```yaml
variables:

  APP_NAME: "order-service"

  AWS_REGION: "ap-south-1"

  ECR_REPOSITORY: "order-service"

  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
```

These variables are available to jobs unless overridden.

---

# 8. Maven Cache

Maven repeatedly downloads dependencies.

For example:

```text
Spring Boot
Hibernate
PostgreSQL driver
JUnit
Lombok
Micrometer
```

Without caching:

```text
Job 1 → Download dependencies
Job 2 → Download dependencies
Job 3 → Download dependencies
```

With caching:

```text
Job 1
  ↓
Download
  ↓
Cache

Job 2
  ↓
Reuse cache
```

Configure:

```yaml
cache:

  key: "$CI_PROJECT_NAME"

  paths:
    - .m2/repository
```

---

# 9. Build Job

Create:

```yaml
build:

  stage: build

  image:
    name: maven:3-eclipse-temurin-21

  script:

    - chmod +x mvnw
    - ./mvnw -B clean package -DskipTests

  artifacts:

    paths:
      - target/*.jar

    expire_in: 7 days
```

The job performs:

```text
Checkout
   ↓
Java
   ↓
Maven
   ↓
Compile
   ↓
Package
   ↓
JAR
```

---

# 10. GitLab Predefined Variables

GitLab provides predefined variables.

Some important ones:

```text
CI_COMMIT_SHA
CI_COMMIT_BRANCH
CI_COMMIT_REF_NAME
CI_PROJECT_NAME
CI_PROJECT_PATH
CI_PIPELINE_ID
CI_JOB_ID
CI_MERGE_REQUEST_IID
```

For our image:

```yaml
${CI_COMMIT_SHA}
```

is particularly important.

Example:

```text
4f81e6d9c8a1...
```

---

# 11. Unit Test Job

Add:

```yaml
test:

  stage: test

  image:
    name: maven:3-eclipse-temurin-21

  script:

    - chmod +x mvnw
    - ./mvnw -B test

  artifacts:

    when: always

    reports:
      junit:
        - target/surefire-reports/*.xml

    paths:
      - target/surefire-reports/

    expire_in: 7 days
```

Now GitLab can expose JUnit test results in the pipeline UI.

---

# 12. Why `when: always`?

Suppose tests fail:

```text
Test
 |
 X
```

Without `when: always`:

```text
Test report
   |
   X
```

The report may not be uploaded.

With:

```yaml
when: always
```

we tell GitLab:

> Upload the test results even when the job fails.

This is very useful for troubleshooting.

---

# 13. Test Failure Example

Suppose:

```text
OrderServiceTest
```

fails.

Pipeline:

```text
Build       PASS
Test        FAIL
```

But GitLab still stores:

```text
target/surefire-reports/
```

So we can inspect:

```text
Which test failed?
Why?
Expected value?
Actual value?
Stack trace?
```

---

# 14. SonarQube Job

Add:

```yaml
sonarqube:

  stage: security

  image:
    name: maven:3-eclipse-temurin-21

  script:

    - chmod +x mvnw

    - >
      ./mvnw -B verify sonar:sonar
      -Dsonar.host.url="${SONAR_HOST_URL}"
      -Dsonar.token="${SONAR_TOKEN}"
      -Dsonar.qualitygate.wait=true
```

Configure:

```text
SONAR_HOST_URL
SONAR_TOKEN
```

through GitLab CI/CD variables.

Do not hardcode the token in `.gitlab-ci.yml`.

---

# 15. GitLab Variables

Navigate to:

```text
Project
  ↓
Settings
  ↓
CI/CD
  ↓
Variables
```

Create:

```text
SONAR_HOST_URL
SONAR_TOKEN
AWS_ROLE_ARN
```

For secrets such as:

```text
SONAR_TOKEN
```

use masked/protected settings where appropriate.

---

# 16. Protected Variables

For production-sensitive values, use protected variables.

Conceptually:

```text
Feature Branch
      |
      X
Protected AWS credentials

main
 |
 v
Protected AWS credentials
```

This prevents arbitrary branches from accessing sensitive deployment credentials.

---

# 17. Gitleaks

Create another job:

```yaml
gitleaks:

  stage: security

  image:
    name: zricethezav/gitleaks:latest
    entrypoint: [""]

  script:

    - gitleaks detect
      --source .
      --verbose
      --redact
```

In a production environment, **pin the Gitleaks container image to an approved immutable version/digest** rather than using `latest`.

The same principle applies to CI images generally.

---

# 18. What Gitleaks Does

Imagine somebody accidentally commits:

```text
application.yaml
```

containing:

```yaml
aws:
  access-key: AKIA...
  secret-key: ...
```

Gitleaks detects the secret pattern.

Pipeline:

```text
Git
 |
 v
Gitleaks
 |
 X
Secret detected
 |
 v
Pipeline FAILED
```

The developer must remove the secret.

And if a real credential was committed, **removing it from Git is not sufficient**—the credential must also be rotated/revoked.

---

# 19. Trivy Filesystem Scan

Add:

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

This scans the repository filesystem.

It can identify issues such as:

```text
dependency vulnerabilities
configuration problems
secret findings
misconfigurations
```

depending on the scanners and configuration enabled.

---

# 20. Docker Build

Now we need to build:

```text
order-service
```

into:

```text
order-service:<commit-sha>
```

The basic job:

```yaml
docker-build:

  stage: docker

  script:

    - docker build
      -t ${APP_NAME}:${CI_COMMIT_SHA}
      .
```

But there's a major question:

> Where is Docker actually running?

---

# 21. Docker-in-Docker

One common GitLab CI pattern is Docker-in-Docker.

Architecture:

```text
GitLab Runner
     |
     ├── CI Container
     |
     └── Docker Daemon
             |
             v
        Docker Image
```

The CI container executes:

```bash
docker build
```

against a Docker daemon.

This can work, but Docker-in-Docker introduces security and operational considerations.

Therefore, we should understand alternatives.

---

# 22. Docker Build Options

Common approaches include:

```text
1. Docker-in-Docker
2. Docker socket mounting
3. BuildKit
4. Kaniko
5. Rootless BuildKit
6. Kubernetes-native builders
```

We should not blindly choose:

```text
docker:dind
```

for every enterprise environment.

Later we'll build a secure runner architecture and choose an appropriate builder.

---

# 23. Trivy Image Scan

After creating:

```text
order-service:<SHA>
```

scan it:

```yaml
trivy-image:

  stage: security

  script:

    - trivy image
      --severity HIGH,CRITICAL
      --ignore-unfixed
      --exit-code 1
      ${APP_NAME}:${CI_COMMIT_SHA}
```

Pipeline:

```text
Docker Build
     |
     v
Trivy
     |
     ├── PASS
     |
     └── FAIL
```

---

# 24. Security Gate

We want:

```text
Trivy
  |
  ├── PASS → Publish
  |
  └── FAIL → STOP
```

Not:

```text
Trivy
  |
  v
WARNING
  |
  v
ECR anyway
```

Our security jobs should therefore fail the pipeline when the defined threshold is exceeded.

---

# 25. AWS Authentication

Now we need:

```text
GitLab
   |
   v
AWS
   |
   v
ECR
```

Do **not** put:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

into GitLab unless there is a specific legacy requirement and appropriate controls.

Prefer:

```text
GitLab OIDC
     |
     v
AWS STS
     |
     v
IAM Role
     |
     v
Temporary credentials
```

This is the same fundamental concept we used with GitHub Actions.

---

# 26. GitLab OIDC Concept

The flow becomes:

```text
GitLab Job
    |
    | OIDC identity token
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

The IAM trust policy should restrict:

```text
Who?
Which GitLab project?
Which branch/environment?
Which identity?
```

rather than trusting every GitLab project.

---

# 27. ECR Login

Once AWS authentication succeeds:

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
docker tag \
  order-service:${CI_COMMIT_SHA} \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:${CI_COMMIT_SHA}
```

Then:

```bash
docker push \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:${CI_COMMIT_SHA}
```

---

# 28. Publish Rules

We do not want feature branches publishing deployment images.

Use:

```yaml
publish:

  stage: publish

  script:

    - echo "Push image to ECR"

  rules:

    - if: '$CI_COMMIT_BRANCH == "main"'
```

Therefore:

```text
feature/*
   |
   X
   |
  ECR
```

while:

```text
main
 |
 v
ECR
```

---

# 29. Merge Request Behavior

We want:

```text
Merge Request
      |
      v
CI
      |
      ├── Build
      ├── Test
      ├── SonarQube
      ├── Gitleaks
      ├── Trivy
      └── Docker
```

but:

```text
NO ECR PUSH
```

This gives us:

```text
MR = validation
main = validation + publish
```

---

# 30. Complete Initial `.gitlab-ci.yml`

A foundation version:

```yaml
stages:
  - build
  - test
  - security
  - docker
  - publish


variables:

  APP_NAME: "order-service"

  AWS_REGION: "ap-south-1"

  ECR_REPOSITORY: "order-service"

  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"


cache:

  key: "$CI_PROJECT_NAME"

  paths:
    - .m2/repository


build:

  stage: build

  image:
    name: maven:3-eclipse-temurin-21

  script:

    - chmod +x mvnw

    - ./mvnw -B clean package -DskipTests

  artifacts:

    paths:
      - target/*.jar

    expire_in: 7 days


test:

  stage: test

  image:
    name: maven:3-eclipse-temurin-21

  script:

    - chmod +x mvnw

    - ./mvnw -B test

  artifacts:

    when: always

    reports:

      junit:
        - target/surefire-reports/*.xml

    paths:

      - target/surefire-reports/

    expire_in: 7 days


sonarqube:

  stage: security

  image:
    name: maven:3-eclipse-temurin-21

  script:

    - chmod +x mvnw

    - >
      ./mvnw -B verify sonar:sonar
      -Dsonar.host.url="${SONAR_HOST_URL}"
      -Dsonar.token="${SONAR_TOKEN}"
      -Dsonar.qualitygate.wait=true


gitleaks:

  stage: security

  image:

    name: zricethezav/gitleaks:<approved-version>

    entrypoint: [""]

  script:

    - gitleaks detect
      --source .
      --verbose
      --redact


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


docker-build:

  stage: docker

  script:

    - docker build
      -t ${APP_NAME}:${CI_COMMIT_SHA}
      .


trivy-image:

  stage: security

  script:

    - trivy image
      --severity HIGH,CRITICAL
      --ignore-unfixed
      --exit-code 1
      ${APP_NAME}:${CI_COMMIT_SHA}


publish:

  stage: publish

  script:

    - echo "AWS authentication"

    - echo "ECR login"

    - echo "Tag image"

    - echo "Push image"

  rules:

    - if: '$CI_COMMIT_BRANCH == "main"'
```

This is deliberately a **learning foundation**.

We will improve the Docker/runner architecture before treating this as production-ready.

---

# 31. Problem With the Initial Pipeline

Look carefully.

We have:

```text
docker-build
     |
     v
trivy-image
```

But GitLab jobs normally execute in separate environments.

That means:

```text
docker-build job
     |
     | Docker image exists here
     v
trivy-image job
     |
     X
Different job environment
```

The image isn't automatically available to the next job.

This is a critical CI concept.

---

# 32. Job Isolation

Think of jobs as:

```text
Job A
┌─────────────────┐
│ Build image     │
│ order-service   │
└─────────────────┘
        |
        X
        |
Job B
┌─────────────────┐
│ Empty runner    │
│ No image        │
└─────────────────┘
```

Therefore we need one of these strategies:

### Strategy A

Build and scan in the same job.

```text
Build
 ↓
Trivy
```

### Strategy B

Push temporary image to a registry.

```text
Build
 ↓
Temporary ECR tag
 ↓
Trivy
```

### Strategy C

Export image as artifact.

```text
Build
 ↓
docker save
 ↓
Artifact
 ↓
download
 ↓
docker load
 ↓
Trivy
```

### Strategy D

Use a registry-based promotion architecture.

```text
Build
 ↓
Registry
 ↓
Scan
 ↓
Promote by digest
```

For our enterprise project, **Strategy D is ultimately the strongest model**.

---

# 33. Why Registry-Based Promotion Is Better

Instead of:

```text
Build
 ↓
Save huge Docker tarball
 ↓
Upload GitLab artifact
 ↓
Download
 ↓
Load
```

we can use:

```text
Build
 ↓
ECR
 ↓
Scan
 ↓
Image Digest
 ↓
Promote
```

Example:

```text
order-service
   |
   └── sha256:abc123...
```

The digest identifies the exact image.

Later:

```text
DEV
 ↓
QA
 ↓
PROD
```

can all use:

```text
sha256:abc123...
```

rather than rebuilding.

---

# 34. `needs:` and DAG Pipelines

Our first pipeline is:

```text
build
 ↓
test
 ↓
security
 ↓
docker
 ↓
publish
```

But some jobs don't actually depend on one another.

For example:

```text
             Build
               |
       ┌───────┼────────┐
       ↓       ↓        ↓
      Test   Gitleaks  Trivy FS
       |       |        |
       └───────┼────────┘
               ↓
             Docker
```

GitLab supports `needs:` to express actual dependencies.

Example:

```yaml
sonarqube:

  stage: security

  needs:
    - build
```

and:

```yaml
gitleaks:

  stage: security

  needs: []
```

This can allow independent jobs to start earlier.

---

# 35. Why DAG Matters

Imagine:

```text
Build = 5 min
Test = 5 min
Gitleaks = 30 sec
Trivy FS = 1 min
```

Sequential execution could unnecessarily wait.

With parallelization:

```text
          Build
            |
       ┌────┼────┐
       ↓    ↓    ↓
     Test Gitleaks Trivy
       |
       └────┬────┘
            ↓
          Docker
```

Overall pipeline time can be reduced.

But don't parallelize blindly.

Dependency relationships should remain clear.

---

# 36. GitLab Pipeline Mental Model

Remember this:

```text
Pipeline
   |
   ├── Stage
   │     |
   │     ├── Job
   │     ├── Job
   │     └── Job
   |
   ├── Stage
   │     |
   │     ├── Job
   │     └── Job
   |
   └── Stage
```

And:

```text
Runner
  |
  └── executes Job
```

This distinction is important:

```text
GitLab
   = orchestration

Runner
   = execution
```

---

# 37. GitHub vs GitLab

We now have:

```text
GitHub Actions:

Workflow
   ↓
Job
   ↓
Runner
   ↓
Steps
```

GitLab:

```text
Pipeline
   ↓
Stage
   ↓
Job
   ↓
Runner
   ↓
Script
```

Conceptually:

```text
GitHub                     GitLab

Workflow        ≈         Pipeline
Job             ≈         Job
Runner          ≈         Runner
Step            ≈         Script command
Artifact        ≈         Artifact
Secret          ≈         Variable
Environment     ≈         Environment
```

The syntax differs.

The DevOps concepts don't.

---

# 38. Enterprise GitLab Architecture

Eventually our architecture should look more like:

```text
                    GitLab
                       |
                       v
                GitLab CI Pipeline
                       |
             ┌─────────┴─────────┐
             |                   |
             v                   v
       Linux Runner        Security Runner
             |                   |
             └─────────┬─────────┘
                       |
                       v
                 Container Build
                       |
                       v
                     ECR
                       |
                       v
                  GitOps Repo
                       |
                       v
                    ArgoCD
                       |
                       v
                      EKS
```

For Kubernetes-based runners:

```text
GitLab
  |
  v
GitLab Runner
  |
  v
EKS
  |
  ├── ephemeral build pod
  ├── ephemeral test pod
  ├── ephemeral security pod
  └── ephemeral Docker/build pod
```

This avoids maintaining large permanent CI servers for every job.

---

# 39. Security Model

The runner is powerful.

It may have access to:

```text
Source code
Docker
AWS
ECR
Secrets
```

Therefore:

```text
Runner compromise
      |
      v
Potential AWS compromise
```

We need controls such as:

```text
Least privilege
Ephemeral runners
Protected branches
Protected variables
OIDC
Network restrictions
No privileged containers unless required
Separate production credentials
Audit logging
Runner isolation
```

We'll cover this in detail when we build the GitLab runner architecture.

---

# 40. Final Target for GitLab CI

Our finished GitLab implementation will become:

```text
                       GitLab
                          |
                          v
                   Merge Request
                          |
                          v
                   GitLab Pipeline
                          |
          ┌───────────────┼────────────────┐
          |               |                |
          v               v                v
        Maven          SonarQube        Security
        Build                            Scans
          |               |                |
          └───────────────┼────────────────┘
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
                          v
                    GitLab OIDC
                          |
                          v
                     AWS IAM Role
                          |
                          v
                        ECR
                          |
                          v
                   Image Digest
                          |
                          v
                     GitOps
```

---

# 41. Three CI Systems So Far

We now understand:

### GitHub Actions

```text
.github/workflows/ci.yaml
```

### GitLab CI

```text
.gitlab-ci.yml
```

Next:

### Jenkins

```text
Jenkinsfile
```

And all three ultimately produce:

```text
                CI
                 |
      ┌──────────┼──────────┐
      ↓          ↓          ↓
   GitHub      GitLab     Jenkins
   Actions       CI       Pipeline
      └──────────┼──────────┘
                 ↓
                ECR
                 ↓
              GitOps
                 ↓
               ArgoCD
                 ↓
                EKS
```

---

# 42. What We Learned

The key lessons from this chapter are:

```text
1. GitLab orchestrates pipelines.
2. GitLab Runner executes jobs.
3. stages define broad execution order.
4. jobs perform actual work.
5. needs can create DAG-style execution.
6. artifacts move build outputs between jobs.
7. cache accelerates dependency downloads.
8. variables hold configuration.
9. protected variables protect sensitive values.
10. rules control when jobs execute.
11. Merge Requests should validate, not publish.
12. main can publish immutable images.
13. OIDC is preferred over long-lived AWS keys.
14. ECR should receive trusted images.
15. Docker images must be traceable to Git commits.
16. Build-once/promote-by-digest is the long-term target.
```

---

## Next: `12-Jenkins-CI.md`

Now we'll build the **Jenkins implementation** of the same `order-service` CI pipeline.

We'll go deeper into the architecture because Jenkins introduces concepts that GitHub Actions and GitLab hide from you:

```text
Jenkins Controller
        |
        v
Jenkins Agents
        |
        ├── Linux Agent
        ├── Docker Agent
        ├── Security Agent
        └── Kubernetes Dynamic Agent
```

Then we'll build:

```text
Jenkinsfile
   ↓
Checkout
   ↓
Maven
   ↓
Unit Tests
   ↓
JaCoCo
   ↓
SonarQube
   ↓
Gitleaks
   ↓
Trivy
   ↓
Docker
   ↓
AWS Authentication
   ↓
ECR
```

After GitHub Actions + GitLab + Jenkins are complete, we'll consolidate the **common CI security/quality architecture** before moving into Kubernetes/EKS.
