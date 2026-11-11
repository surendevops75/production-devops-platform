Yes. We'll continue with the **three-CI architecture** and keep the implementation production-oriented.

# 09-CI-Pipeline-Foundation.md

## 1. Purpose

This chapter establishes the CI foundation for the `order-service`.

We will implement the same CI lifecycle using:

1. **GitHub Actions**
2. **GitLab CI/CD**
3. **Jenkins**

The important design principle is that **CI logic stays consistent while the implementation changes**.

GitHub Actions workflows are defined in YAML and consist of jobs and steps. ([GitHub Docs][1]) GitLab CI uses `.gitlab-ci.yml`, with stages defining execution order and jobs within the same stage able to run in parallel. ([GitLab Docs][2]) Jenkins uses a `Jenkinsfile` stored in source control to define the pipeline as code. ([Jenkins][3])

---

# 2. CI Objective

Our target pipeline is:

```text
Developer
   |
   v
Git Push / Pull Request
   |
   v
Checkout Source
   |
   v
Compile
   |
   v
Unit Tests
   |
   v
Code Quality
   |
   v
Dependency Security
   |
   v
Secret Scanning
   |
   v
Package Application
   |
   v
Build Docker Image
   |
   v
Trivy Container Scan
   |
   v
Authenticate to AWS
   |
   v
Push Image to ECR
   |
   v
Record Image SHA/Digest
   |
   v
GitOps Promotion
```

The CI pipeline **does not deploy directly to EKS**.

Later:

```text
CI
 |
 v
ECR
 |
 v
GitOps Repository
 |
 v
Argo CD
 |
 v
EKS
```

This keeps CI and CD separated.

---

# 3. CI vs CD

This distinction is extremely important.

## CI

CI answers:

> "Is this source code safe and buildable, and can I produce a trusted container image?"

```text
Source
  ↓
Build
  ↓
Test
  ↓
Scan
  ↓
Docker Build
  ↓
Trivy
  ↓
ECR
```

## CD

CD answers:

> "Which version should run in which environment?"

```text
ECR
 ↓
GitOps
 ↓
DEV
 ↓
QA
 ↓
STAGING
 ↓
PRODUCTION
 ↓
EKS
```

Therefore:

```text
GitHub Actions ──┐
GitLab CI ───────┼──> ECR
Jenkins ─────────┘
                    |
                    v
                 GitOps
                    |
                    v
                  ArgoCD
                    |
                    v
                   EKS
```

---

# 4. Repository Structure

Our `order-service` repository will eventually look like:

```text
order-service/
│
├── src/
│
├── pom.xml
├── mvnw
├── mvnw.cmd
├── .mvn/
│
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
│
├── README.md
│
├── .gitignore
│
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│       ├── ci.yaml
│       ├── security.yaml
│       └── dependency-review.yaml
│
├── .gitlab-ci.yml
│
└── Jenkinsfile
```

The three pipeline files implement essentially the same CI process:

```text
.github/workflows/ci.yaml
.gitlab-ci.yml
Jenkinsfile
```

---

# 5. Why We Keep All Three

In a real company, you normally wouldn't maintain three CI systems for the same service without a reason.

We're doing it here intentionally for learning.

You will understand:

```text
GitHub Actions
     ↓
GitLab CI
     ↓
Jenkins
```

and recognize that:

```text
stage
job
runner
artifact
secret
environment
approval
cache
parallel
condition
```

are concepts that exist across CI platforms.

---

# 6. CI Execution Model

Every pipeline will contain these logical stages:

```text
01 Checkout
02 Build
03 Unit Test
04 Code Quality
05 Dependency Scan
06 Secret Scan
07 Docker Build
08 Container Scan
09 AWS Authentication
10 ECR Push
```

However, we won't necessarily make every stage sequential.

For example:

```text
                 Build
                   |
              ┌────┴────┐
              |         |
           Unit Test   SonarQube
              |         |
              └────┬────┘
                   |
          Security Scanning
             /           \
       Gitleaks          Dependency
             \           /
              \         /
               Docker Build
                    |
                 Trivy
                    |
                  ECR
```

This reduces unnecessary pipeline execution time.

---

# 7. Pull Request Pipeline

When a developer creates:

```text
feature/order-validation
```

and opens:

```text
feature/order-validation
        |
        v
Pull Request
        |
        v
main
```

we run validation.

### PR pipeline

```text
Checkout
   ↓
Compile
   ↓
Unit Tests
   ↓
SonarQube
   ↓
Dependency Scan
   ↓
Secret Scan
   ↓
Docker Build
   ↓
Trivy
```

**No ECR push.**

This is important.

We don't want every pull request creating production-eligible images in ECR.

---

# 8. Main Branch Pipeline

After approval:

```text
PR
 ↓
Review
 ↓
Merge
 ↓
main
```

the pipeline runs:

```text
Checkout
   ↓
Build
   ↓
Tests
   ↓
Quality
   ↓
Security
   ↓
Docker Build
   ↓
Trivy
   ↓
AWS Authentication
   ↓
ECR Push
```

Example:

```text
order-service:4f81e6d9...
```

The Git commit becomes part of the image identity.

---

# 9. Image Tagging Strategy

Never use:

```text
latest
```

for production deployment.

Instead:

```text
order-service:<git-sha>
```

Example:

```text
order-service:4f81e6d9c8a1...
```

We can additionally support release tags:

```text
order-service:1.5.0
```

But the immutable Git SHA remains the primary traceability mechanism.

---

# 10. Build Once

One of the most important principles in this project:

> **Build once, promote the same artifact.**

Bad:

```text
DEV
 ↓
rebuild image
 ↓
QA
 ↓
rebuild image
 ↓
PROD
 ↓
rebuild image
```

Good:

```text
Git Commit
    ↓
Build
    ↓
Image
    ↓
SHA/Digest
    ↓
DEV
    ↓
QA
    ↓
PROD
```

The image should not be rebuilt simply because it is being promoted.

---

# 11. Maven Wrapper

Before creating CI, make sure the repository contains:

```text
mvnw
mvnw.cmd
.mvn/
```

Check:

```bash
ls -la mvnw
ls -la .mvn
```

On Linux:

```bash
chmod +x mvnw
```

Then:

```bash
git update-index --chmod=+x mvnw
```

Verify:

```bash
git ls-files --stage mvnw
```

The executable permission should be present.

---

# 12. Local CI Simulation

Before depending on GitHub, GitLab or Jenkins, make sure the commands work locally.

Run:

```bash
./mvnw -B clean verify
```

Expected:

```text
BUILD SUCCESS
```

Then:

```bash
docker build -t order-service:test .
```

Run:

```bash
docker run --rm \
  -p 8080:8080 \
  order-service:test
```

Test:

```bash
curl http://localhost:8080/actuator/health
```

Expected:

```json
{
  "status": "UP"
}
```

Only after local validation should we automate these commands.

---

# 13. GitHub Actions Foundation

Create:

```text
.github/workflows/ci.yaml
```

Initial pipeline:

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
  group: order-service-ci-${{ github.ref }}
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

      - name: Verify Maven Wrapper
        run: |
          chmod +x mvnw
          ./mvnw --version

      - name: Build and test
        run: |
          ./mvnw -B clean verify

      - name: Upload application artifact
        uses: actions/upload-artifact@v4
        with:
          name: order-service-${{ github.sha }}
          path: target/*.jar
          retention-days: 7
```

---

# 14. Understanding This Workflow

The workflow starts with:

```yaml
name: Order Service CI
```

This is the name displayed in GitHub Actions.

---

## Trigger

```yaml
on:
  pull_request:
    branches:
      - main

  push:
    branches:
      - main
```

This means:

```text
PR → main
```

runs CI.

And:

```text
push → main
```

also runs CI.

GitHub workflow triggers can be configured for events such as pushes and pull requests. ([GitHub Docs][1])

---

# 15. Permissions

```yaml
permissions:
  contents: read
```

We start with the minimum required permission.

Later the ECR job will require:

```yaml
id-token: write
```

for AWS OIDC.

We do **not** give every job every permission.

---

# 16. Concurrency

```yaml
concurrency:
  group: order-service-ci-${{ github.ref }}
  cancel-in-progress: true
```

Suppose the developer pushes:

```text
commit A
```

then immediately:

```text
commit B
```

The pipeline for A may still be running.

Since B supersedes A for the same branch, we can cancel the older pipeline.

This saves CI resources.

---

# 17. Java Setup

```yaml
- name: Setup Java
  uses: actions/setup-java@v4
  with:
    distribution: temurin
    java-version: '21'
    cache: maven
```

Our application uses:

```text
Java 21
```

and Maven dependency caching reduces repeated dependency downloads.

---

# 18. Build and Test

```yaml
- name: Build and test
  run: |
    ./mvnw -B clean verify
```

This is better than doing:

```bash
mvn clean package
mvn test
```

because we're otherwise repeating Maven lifecycle work.

The `verify` lifecycle executes the relevant earlier Maven lifecycle phases.

---

# 19. Artifact

We upload:

```yaml
target/*.jar
```

as a GitHub Actions artifact.

Example:

```text
order-service-a81f0b9....
        |
        v
order-service-1.0.0.jar
```

This gives us an auditable build output.

Later we'll discuss why the **container image**, rather than the JAR artifact, becomes the deployment artifact for Kubernetes.

---

# 20. Production Action Pinning

The example uses:

```yaml
actions/checkout@v4
```

for readability.

In a security-sensitive enterprise environment, action versions should be controlled, and GitHub specifically recommends pinning actions to a commit SHA for stronger stability/security. ([GitHub Docs][1])

For example:

```yaml
uses: actions/checkout@<approved-commit-sha>
```

Our learning repository can use major versions first.

Later we'll create the enterprise hardened version.

---

# 21. GitLab CI Foundation

Now create:

```text
.gitlab-ci.yml
```

Initial version:

```yaml
stages:
  - build
  - test
  - security
  - docker
  - publish

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
  IMAGE_NAME: "order-service"

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


security:
  stage: security

  image:
    name: maven:3-eclipse-temurin-21

  script:
    - chmod +x mvnw
    - ./mvnw -B verify


docker-build:
  stage: docker

  script:
    - docker build -t ${IMAGE_NAME}:${CI_COMMIT_SHA} .


publish:
  stage: publish

  script:
    - echo "Push image to ECR"

  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

GitLab's `stages` define execution order; jobs in the same stage can run in parallel, while later stages wait for earlier stages to succeed. ([GitLab Docs][2])

---

# 22. GitLab Pipeline Flow

The pipeline is:

```text
build
  |
  v
test
  |
  v
security
  |
  v
docker
  |
  v
publish
```

Later we'll optimize this.

For example:

```text
              build
                |
        ┌───────┼────────┐
        ↓       ↓        ↓
      test    sonar   dependency
        └───────┼────────┘
                ↓
             Docker
                ↓
             Trivy
                ↓
               ECR
```

GitLab's `needs` keyword can be used later to reduce unnecessary stage waiting and create more efficient DAG-style pipelines. ([GitLab Docs][2])

---

# 23. Jenkins Foundation

Create:

```text
Jenkinsfile
```

Use Declarative Pipeline initially:

```groovy
pipeline {

    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')

        disableConcurrentBuilds()
    }

    environment {
        APP_NAME = 'order-service'
        AWS_REGION = 'ap-south-1'
    }

    stages {

        stage('Checkout') {

            steps {
                checkout scm
            }
        }

        stage('Build') {

            steps {
                sh '''
                    chmod +x mvnw
                    ./mvnw -B clean package -DskipTests
                '''
            }
        }

        stage('Unit Test') {

            steps {
                sh '''
                    ./mvnw -B test
                '''
            }
        }

        stage('Security') {

            steps {
                sh '''
                    ./mvnw -B verify
                '''
            }
        }

        stage('Docker Build') {

            steps {
                sh '''
                    docker build \
                      -t ${APP_NAME}:${GIT_COMMIT} \
                      .
                '''
            }
        }

        stage('Container Scan') {

            steps {
                sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      ${APP_NAME}:${GIT_COMMIT}
                '''
            }
        }

        stage('Publish') {

            when {
                branch 'main'
            }

            steps {
                sh '''
                    echo "Push image to ECR"
                '''
            }
        }
    }

    post {

        always {
            echo 'CI pipeline completed.'
        }

        failure {
            echo 'CI pipeline failed.'
        }

        success {
            echo 'CI pipeline succeeded.'
        }
    }
}
```

Jenkins supports Declarative and Scripted Pipeline syntax, and keeping the `Jenkinsfile` in source control provides reviewability and an auditable pipeline definition. ([Jenkins][3])

---

# 24. Jenkins `agent`

```groovy
agent any
```

means Jenkins can execute the pipeline on an available agent.

Later we'll replace this with a proper architecture:

```text
Jenkins Controller
       |
       ├── Linux Build Agent
       |
       ├── Docker Agent
       |
       └── Kubernetes Dynamic Agents
```

We will **not** run production workloads on the Jenkins controller.

---

# 25. Jenkins `post`

This:

```groovy
post {

    always {
        echo 'CI pipeline completed.'
    }

    failure {
        echo 'CI pipeline failed.'
    }

    success {
        echo 'CI pipeline succeeded.'
    }
}
```

is extremely useful in real pipelines.

Later we'll use it for:

```text
failure
   ↓
Publish logs
   ↓
Archive test reports
   ↓
Send notification
```

---

# 26. Comparing the Three

At this point:

### GitHub Actions

```text
.github/workflows/ci.yaml
```

### GitLab

```text
.gitlab-ci.yml
```

### Jenkins

```text
Jenkinsfile
```

But conceptually:

```text
                  CI
                   |
       ┌───────────┼───────────┐
       |           |           |
    GitHub       GitLab      Jenkins
    Actions        CI        Pipeline
       |           |           |
       └───────────┼───────────┘
                   |
                Same Goal
                   |
                   v
                 ECR
```

---

# 27. What We Have NOT Done Yet

These first versions are intentionally foundational.

They are **not yet production-ready**.

We still need:

```text
❌ SonarQube quality gate
❌ JaCoCo coverage
❌ Dependency vulnerability scanning
❌ Gitleaks
❌ CodeQL
❌ Trivy filesystem scanning
❌ Trivy image scanning policy
❌ AWS OIDC
❌ GitLab → AWS OIDC
❌ Jenkins → AWS authentication
❌ ECR push
❌ immutable image handling
❌ image digest capture
❌ SBOM
❌ image signing
❌ artifact promotion
❌ PR-specific behavior
❌ main-branch behavior
❌ release tagging
❌ pipeline notifications
❌ retry/timeout strategy
❌ reusable pipelines
❌ Jenkins shared libraries
❌ GitLab templates
❌ GitHub reusable workflows
```

We will build these **one by one**, rather than dumping one giant YAML file and pretending you understand it.

---

# 28. The Target Production Pipeline

By the end of the CI section, all three should achieve this:

```text
                         Git Repository
                              |
                              v
                         CI Trigger
                              |
                              v
                       ┌─────────────┐
                       │   Checkout  │
                       └──────┬──────┘
                              |
                              v
                         Compile/Build
                              |
                              v
                         Unit Tests
                              |
                ┌─────────────┼─────────────┐
                ↓             ↓             ↓
            SonarQube      Gitleaks     Dependency
                                          Scan
                └─────────────┼─────────────┘
                              ↓
                        Docker Build
                              |
                              v
                         Trivy Scan
                              |
                              v
                       Security Gate
                              |
                         PASS / FAIL
                              |
                              v
                      AWS OIDC Authentication
                              |
                              v
                             ECR
                              |
                              v
                      Immutable Image
                              |
                              v
                     Image Digest
                              |
                              v
                        GitOps Update
                              |
                              v
                           ArgoCD
                              |
                              v
                            EKS
```

And the three implementations will be:

```text
GitHub Actions
      │
      ├── .github/workflows/ci.yaml
      │
      └── GitHub Runner


GitLab CI
      │
      ├── .gitlab-ci.yml
      │
      └── GitLab Runner


Jenkins
      │
      ├── Jenkinsfile
      │
      └── Jenkins Agent
```

This gives you a genuine understanding of **CI architecture**, rather than memorizing one platform's syntax.

---

# 29. Important Rule Going Forward

We will **not duplicate every chapter three times unnecessarily**.

Instead:

```text
Concept
  ↓
Understand it once
  ↓
GitHub implementation
  ↓
GitLab implementation
  ↓
Jenkins implementation
```

For example, when we reach AWS authentication:

```text
AWS OIDC concept
       ↓
GitHub OIDC
       ↓
GitLab OIDC
       ↓
Jenkins AWS authentication
```

When we reach reusable pipelines:

```text
Reusable CI concept
       ↓
GitHub reusable workflow
       ↓
GitLab include/template
       ↓
Jenkins Shared Library
```

That's much closer to how an experienced DevOps engineer thinks.

---

## Next: `10-GitHub-Actions-CI.md`

We'll now take the basic GitHub workflow above and turn it into a **real CI pipeline**, including:

* PR vs `main` behavior
* Maven caching
* unit tests
* test reports
* JaCoCo
* SonarQube
* quality gate
* dependency scanning
* Gitleaks
* Docker BuildKit
* Trivy filesystem scan
* Trivy image scan
* build-once strategy
* immutable image tags
* AWS OIDC
* ECR authentication
* ECR push
* image digest
* failure handling
* artifacts
* GitHub environments
* secrets vs variables
* security permissions

Then we'll implement the **same CI logic in GitLab CI**, followed by **Jenkins**.

[1]: https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax?utm_source=chatgpt.com "Workflow syntax for GitHub Actions - GitHub Docs"
[2]: https://docs.gitlab.com/ci/yaml/?utm_source=chatgpt.com "CI/CD YAML syntax reference | GitLab Docs"
[3]: https://www.jenkins.io/doc/book/pipeline/jenkinsfile/?utm_source=chatgpt.com "Using a Jenkinsfile"
