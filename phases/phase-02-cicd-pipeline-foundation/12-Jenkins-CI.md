# 12-Jenkins-CI.md

## 1. Purpose

We have now implemented the CI foundation using:

* GitHub Actions
* GitLab CI/CD

Now we implement the same `order-service` CI process using **Jenkins**.

The objective is not to memorize Groovy syntax.

The objective is to understand how Jenkins works in a real enterprise environment:

```text
Developer
    |
    v
GitHub
    |
    | Webhook
    v
Jenkins Controller
    |
    v
Jenkins Agent
    |
    ├── Checkout
    ├── Build
    ├── Test
    ├── SonarQube
    ├── Security Scan
    ├── Docker Build
    ├── Trivy
    └── ECR
```

Jenkins Pipeline supports defining pipelines as code in a `Jenkinsfile`, which can be stored and reviewed alongside the application source code.

---

# 2. Why Jenkins Is Different

With GitHub Actions:

```text
GitHub
  |
  v
GitHub Runner
```

With GitLab:

```text
GitLab
  |
  v
GitLab Runner
```

With Jenkins:

```text
GitHub
  |
  v
Jenkins Controller
  |
  v
Jenkins Agent
```

The Jenkins controller coordinates the pipeline, while agents perform the actual work.

This introduces more infrastructure responsibility.

---

# 3. Jenkins Architecture

Our learning architecture:

```text
                         GitHub
                            |
                         Webhook
                            |
                            v
                  ┌──────────────────┐
                  │ Jenkins Controller│
                  │                  │
                  │ Pipeline Engine  │
                  │ Credentials      │
                  │ Job Configuration │
                  └────────┬─────────┘
                           |
                 ┌─────────┼─────────┐
                 |         |         |
                 v         v         v
             Agent-1   Agent-2   Agent-3
             Linux     Docker    Security
                 |         |         |
                 └─────────┼─────────┘
                           |
                           v
                          ECR
```

Later we will replace much of this with:

```text
Jenkins Controller
        |
        v
Kubernetes Plugin
        |
        v
EKS
        |
        ├── ephemeral-agent-1
        ├── ephemeral-agent-2
        └── ephemeral-agent-3
```

---

# 4. Controller vs Agent

This distinction is critical.

## Jenkins Controller

The controller handles things such as:

```text
Pipeline orchestration
Job scheduling
Pipeline metadata
Credentials management
Plugin configuration
Build coordination
```

It should **not** be treated as a general-purpose build server.

---

## Jenkins Agent

An agent executes:

```text
./mvnw
docker build
trivy
aws
```

For example:

```text
Jenkins Controller
       |
       | assigns job
       v
Jenkins Linux Agent
       |
       ├── Java
       ├── Maven
       ├── Docker
       ├── AWS CLI
       └── Trivy
```

---

# 5. Why Not Build on the Controller?

Bad architecture:

```text
Jenkins Controller
      |
      ├── Maven
      ├── Docker
      ├── Trivy
      ├── AWS CLI
      └── All builds
```

Better:

```text
Jenkins Controller
       |
       v
Jenkins Agents
       |
       ├── Build
       ├── Test
       ├── Security
       └── Docker
```

The controller should remain focused on orchestration.

---

# 6. Jenkinsfile

Create:

```text
Jenkinsfile
```

in the root of `order-service`.

Repository:

```text
order-service/
│
├── src/
├── pom.xml
├── Dockerfile
├── Jenkinsfile
│
├── .github/
│   └── workflows/
│
└── .gitlab-ci.yml
```

Now the same repository contains:

```text
GitHub Actions
GitLab CI
Jenkins
```

for learning purposes.

---

# 7. Jenkins Declarative Pipeline

We'll use Declarative Pipeline syntax.

Basic structure:

```groovy
pipeline {

    agent any

    stages {

        stage('Example') {

            steps {
                echo 'Hello'
            }
        }
    }
}
```

Conceptually:

```text
pipeline
   |
   ├── agent
   |
   ├── stages
   │    |
   │    ├── stage
   │    │     └── steps
   │    |
   │    └── stage
   |
   └── post
```

---

# 8. Initial Jenkinsfile

Start with:

```groovy
pipeline {

    agent any

    options {

        timeout(
            time: 30,
            unit: 'MINUTES'
        )

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

                    ./mvnw -B \
                      clean package \
                      -DskipTests
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

        stage('Trivy Scan') {

            steps {

                sh '''
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      --exit-code 1 \
                      ${APP_NAME}:${GIT_COMMIT}
                '''
            }
        }

        stage('Publish') {

            when {

                branch 'main'
            }

            steps {

                echo 'Publish image to ECR'
            }
        }
    }

    post {

        always {

            echo 'Pipeline completed'
        }

        success {

            echo 'Pipeline succeeded'
        }

        failure {

            echo 'Pipeline failed'
        }
    }
}
```

This is our first working Jenkins pipeline.

---

# 9. `pipeline`

Everything is inside:

```groovy
pipeline {
}
```

This tells Jenkins that we're using a Declarative Pipeline.

---

# 10. `agent any`

```groovy
agent any
```

means:

> Run this pipeline on any available Jenkins agent.

For learning:

```text
Jenkins
  |
  v
Any available agent
```

For production, we want more control.

For example:

```groovy
agent {
    label 'linux-docker'
}
```

Then Jenkins chooses:

```text
Agent
  |
  └── label = linux-docker
```

---

# 11. Why Labels Matter

Imagine we have:

```text
agent-linux
agent-docker
agent-security
agent-arm64
```

We don't want a Docker build accidentally running on an agent that doesn't have Docker.

So:

```groovy
agent {
    label 'docker'
}
```

can select an appropriate worker.

Later we'll build a proper agent strategy.

---

# 12. Pipeline Options

We have:

```groovy
options {

    timeout(
        time: 30,
        unit: 'MINUTES'
    )

    disableConcurrentBuilds()
}
```

### Timeout

If the pipeline hangs:

```text
Build
 ↓
Waiting...
 ↓
Waiting...
 ↓
Waiting...
```

we don't want it running forever.

After 30 minutes:

```text
Pipeline
 ↓
TIMEOUT
 ↓
FAIL
```

---

# 13. Disable Concurrent Builds

Suppose:

```text
Build #100
```

is running.

Developer pushes another commit:

```text
Build #101
```

With:

```groovy
disableConcurrentBuilds()
```

Jenkins won't execute both builds concurrently for this job.

This is useful for pipelines that manipulate shared resources.

Later, we can use more sophisticated concurrency strategies.

---

# 14. Environment Variables

```groovy
environment {

    APP_NAME = 'order-service'

    AWS_REGION = 'ap-south-1'
}
```

Now:

```bash
${APP_NAME}
```

becomes:

```text
order-service
```

and:

```bash
${AWS_REGION}
```

becomes:

```text
ap-south-1
```

---

# 15. Checkout

```groovy
stage('Checkout') {

    steps {

        checkout scm
    }
}
```

This means Jenkins checks out the source code associated with the job.

Flow:

```text
GitHub
  |
  v
Jenkins
  |
  v
Checkout
  |
  v
Workspace
```

The Jenkins workspace contains:

```text
order-service/
```

---

# 16. Jenkins Workspace

Suppose Jenkins assigns:

```text
agent-01
```

The workspace might conceptually be:

```text
/workspace/order-service/
```

Inside:

```text
/workspace/order-service/
│
├── src/
├── pom.xml
├── Dockerfile
├── Jenkinsfile
└── target/
```

The pipeline operates inside this workspace.

---

# 17. Build

```groovy
stage('Build') {

    steps {

        sh '''
            chmod +x mvnw

            ./mvnw -B \
              clean package \
              -DskipTests
        '''
    }
}
```

This executes:

```text
Maven
  |
  ├── clean
  ├── compile
  └── package
```

We skip tests here because the next stage handles them.

---

# 18. Unit Tests

```groovy
stage('Unit Test') {

    steps {

        sh '''
            ./mvnw -B test
        '''
    }
}
```

Flow:

```text
Build
  |
  v
Unit Test
  |
  ├── PASS
  |
  └── FAIL
```

If tests fail, Jenkins stops later stages by default.

---

# 19. Test Reports

We shouldn't just execute tests.

We should archive results.

Add:

```groovy
post {

    always {

        junit(
            testResults: 'target/surefire-reports/*.xml',
            allowEmptyResults: true
        )
    }

    success {

        echo 'Pipeline succeeded'
    }

    failure {

        echo 'Pipeline failed'
    }
}
```

Now Jenkins can display test results.

Example:

```text
Tests
 ├── Total: 42
 ├── Passed: 40
 ├── Failed: 2
 └── Skipped: 0
```

This is much more useful than simply seeing:

```text
BUILD FAILED
```

---

# 20. JaCoCo

Our Maven project also generates:

```text
target/site/jacoco/
```

Jenkins can archive the report:

```groovy
post {

    always {

        junit(
            testResults: 'target/surefire-reports/*.xml',
            allowEmptyResults: true
        )

        archiveArtifacts(
            artifacts: 'target/site/jacoco/**',
            allowEmptyArchive: true
        )
    }
}
```

Later SonarQube will consume coverage information directly from the Maven build.

---

# 21. SonarQube

Add a dedicated stage:

```groovy
stage('SonarQube') {

    steps {

        withSonarQubeEnv('SonarQube') {

            sh '''
                ./mvnw -B \
                  verify sonar:sonar
            '''
        }
    }
}
```

`SonarQube` here refers to the Jenkins SonarQube server configuration name.

---

# 22. Quality Gate

We shouldn't continue automatically if the quality gate fails.

A typical Jenkins approach is:

```groovy
stage('Quality Gate') {

    steps {

        timeout(
            time: 10,
            unit: 'MINUTES'
        ) {

            waitForQualityGate(
                abortPipeline: true
            )
        }
    }
}
```

Flow:

```text
Jenkins
   |
   v
SonarQube
   |
   v
Quality Gate
   |
   ├── PASS → Continue
   |
   └── FAIL → Abort
```

---

# 23. Important Jenkins Plugin Concept

Jenkins often extends functionality through plugins.

Examples:

```text
Pipeline
Git
Credentials
Docker
SonarQube
JUnit
Kubernetes
AWS
```

But don't install plugins randomly.

Every plugin adds:

```text
maintenance
security surface
compatibility concerns
```

Enterprise Jenkins installations should maintain an approved plugin catalog and version-management process.

---

# 24. Gitleaks

We can execute Gitleaks from the agent.

Example:

```groovy
stage('Secret Scan') {

    steps {

        sh '''
            gitleaks detect \
              --source . \
              --verbose \
              --redact
        '''
    }
}
```

If Gitleaks finds a secret:

```text
Gitleaks
   |
   X
Secret detected
   |
   v
Jenkins FAIL
```

---

# 25. Trivy Filesystem Scan

Before building the container:

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

This gives us:

```text
Source
  |
  v
Trivy FS
  |
  ├── PASS
  └── FAIL
```

---

# 26. Docker Build

Now:

```groovy
stage('Docker Build') {

    steps {

        sh '''
            docker build \
              -t ${APP_NAME}:${GIT_COMMIT} \
              .
        '''
    }
}
```

Image:

```text
order-service:<Git SHA>
```

For example:

```text
order-service:4f81e6d9c8a1...
```

---

# 27. Why `GIT_COMMIT`?

Jenkins commonly exposes SCM-related environment variables depending on the job/plugin configuration.

For stronger portability, we can explicitly capture the commit:

```groovy
script {

    env.IMAGE_TAG = sh(
        script: 'git rev-parse HEAD',
        returnStdout: true
    ).trim()
}
```

Then:

```groovy
docker build \
  -t ${APP_NAME}:${IMAGE_TAG} \
  .
```

This makes the image identity explicitly tied to Git.

---

# 28. Better Environment Configuration

We can define:

```groovy
environment {

    APP_NAME = 'order-service'

    AWS_REGION = 'ap-south-1'

    ECR_REPOSITORY = 'order-service'

    IMAGE_TAG = ''
}
```

Then:

```groovy
stage('Determine Version') {

    steps {

        script {

            env.IMAGE_TAG = sh(
                script: 'git rev-parse HEAD',
                returnStdout: true
            ).trim()
        }
    }
}
```

Now:

```text
IMAGE_TAG
    =
Git commit SHA
```

---

# 29. Trivy Image Scan

After Docker build:

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

If:

```text
CRITICAL = 1
```

then:

```text
exit code = 1
```

and Jenkins marks the stage as failed.

---

# 30. Pipeline Security Gate

Our Jenkins pipeline now resembles:

```text
Checkout
   |
   v
Build
   |
   v
Unit Test
   |
   v
SonarQube
   |
   v
Quality Gate
   |
   v
Gitleaks
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
ECR
```

This is the same fundamental lifecycle we implemented with GitHub and GitLab.

---

# 31. Jenkins Credentials

Never do:

```groovy
environment {

    AWS_ACCESS_KEY = 'AKIA...'

    AWS_SECRET = '....'
}
```

Never.

Instead Jenkins provides a credentials store.

Conceptually:

```text
Jenkins Credentials
        |
        ├── GitHub credentials
        ├── SonarQube token
        ├── AWS identity
        └── Other secrets
```

The pipeline references credentials by ID.

---

# 32. SonarQube Credentials

For example:

```groovy
withCredentials([
    string(
        credentialsId: 'sonarqube-token',
        variable: 'SONAR_TOKEN'
    )
]) {

    sh '''
        ./mvnw sonar:sonar \
          -Dsonar.token="${SONAR_TOKEN}"
    '''
}
```

The actual secret isn't stored inside the Jenkinsfile.

---

# 33. AWS Authentication

For our enterprise design, we want to avoid long-lived AWS keys.

Preferred architecture:

```text
Jenkins
   |
   v
AWS Identity
   |
   v
STS
   |
   v
Temporary credentials
   |
   v
ECR
```

Exactly how this is implemented depends on where Jenkins agents run.

---

# 34. If Jenkins Runs in AWS

Suppose:

```text
AWS
 |
 ├── Jenkins Controller
 |
 └── Jenkins Agents
```

Then an agent can use an AWS IAM role.

Architecture:

```text
Jenkins Agent
      |
      v
AWS IAM Role
      |
      v
Temporary Credentials
      |
      v
ECR
```

This is preferable to storing static access keys.

---

# 35. If Jenkins Runs on EKS

Later our architecture can become:

```text
                  AWS
                   |
                   v
                  EKS
                   |
          ┌────────┴────────┐
          |                 |
       Jenkins            Apps
          |
          v
    Dynamic Agents
          |
          v
       AWS IAM
          |
          v
         ECR
```

Jenkins agents can be dynamically created as Kubernetes pods.

This is a powerful enterprise pattern because agents can be ephemeral.

---

# 36. Jenkins Kubernetes Agents

Instead of:

```text
Jenkins Agent
   |
   └── Always running
```

we can have:

```text
Jenkins
   |
   v
Kubernetes
   |
   ├── Build Pod
   |
   ├── Security Pod
   |
   └── Docker Build Pod
```

After the pipeline:

```text
Build Pod
   |
   v
Destroyed
```

This reduces persistent build-environment drift.

We'll build this architecture later.

---

# 37. ECR Login

Once AWS authentication is available:

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
  order-service:${IMAGE_TAG} \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:${IMAGE_TAG}
```

Then:

```bash
docker push \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:${IMAGE_TAG}
```

---

# 38. Jenkins Publish Stage

Our initial stage:

```groovy
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
```

Later we'll replace this with the actual ECR authentication and push.

The important logic is:

```text
feature/*
    |
    X
    |
   ECR

main
 |
 v
ECR
```

---

# 39. Pull Request vs Main

Our three CI systems now follow the same philosophy.

### Pull Request

```text
PR
 |
 ├── Build
 ├── Test
 ├── Sonar
 ├── Gitleaks
 ├── Trivy
 └── Docker
 |
 X
ECR
```

### Main

```text
main
 |
 ├── Build
 ├── Test
 ├── Sonar
 ├── Gitleaks
 ├── Trivy
 ├── Docker
 |
 v
ECR
```

---

# 40. Jenkins Webhook

We don't want developers manually clicking:

```text
Build Now
```

every time.

Instead:

```text
Developer
    |
    v
Git Push
    |
    v
GitHub
    |
    | Webhook
    v
Jenkins
    |
    v
Pipeline
```

GitHub notifies Jenkins that a repository event occurred.

---

# 41. Multibranch Pipeline

For a real GitHub repository, Jenkins should generally discover branches/PRs automatically through a Multibranch Pipeline setup.

Conceptually:

```text
GitHub
 |
 ├── main
 ├── feature/order-validation
 ├── feature/payment
 └── bugfix/inventory
       |
       v
Jenkins Multibranch Pipeline
       |
       ├── main pipeline
       ├── feature pipeline
       ├── feature pipeline
       └── bugfix pipeline
```

This is much better than manually creating a Jenkins job for every branch.

---

# 42. Branch Discovery

Suppose a developer creates:

```text
feature/order-discount
```

Jenkins detects:

```text
New branch
   |
   v
Creates branch job
   |
   v
Reads Jenkinsfile
   |
   v
Runs pipeline
```

When the branch is deleted:

```text
feature/order-discount
   |
   v
Deleted
   |
   v
Jenkins branch job can be cleaned up
```

---

# 43. Complete Jenkins Foundation

Our Jenkinsfile now becomes approximately:

```groovy
pipeline {

    agent any

    options {

        timeout(
            time: 30,
            unit: 'MINUTES'
        )

        disableConcurrentBuilds()
    }

    environment {

        APP_NAME = 'order-service'

        AWS_REGION = 'ap-south-1'

        ECR_REPOSITORY = 'order-service'
    }

    stages {

        stage('Checkout') {

            steps {

                checkout scm
            }
        }

        stage('Determine Version') {

            steps {

                script {

                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()
                }

                echo "Image tag: ${IMAGE_TAG}"
            }
        }

        stage('Build') {

            steps {

                sh '''
                    chmod +x mvnw

                    ./mvnw -B \
                      clean package \
                      -DskipTests
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

        stage('SonarQube') {

            steps {

                withSonarQubeEnv('SonarQube') {

                    sh '''
                        ./mvnw -B \
                          verify sonar:sonar
                    '''
                }
            }
        }

        stage('Quality Gate') {

            steps {

                timeout(
                    time: 10,
                    unit: 'MINUTES'
                ) {

                    waitForQualityGate(
                        abortPipeline: true
                    )
                }
            }
        }

        stage('Secret Scan') {

            steps {

                sh '''
                    gitleaks detect \
                      --source . \
                      --verbose \
                      --redact
                '''
            }
        }

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

        stage('Docker Build') {

            steps {

                sh '''
                    docker build \
                      -t ${APP_NAME}:${IMAGE_TAG} \
                      .
                '''
            }
        }

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

        stage('Publish') {

            when {

                branch 'main'
            }

            steps {

                echo 'Publish image to ECR'
            }
        }
    }

    post {

        always {

            junit(
                testResults: 'target/surefire-reports/*.xml',
                allowEmptyResults: true
            )

            archiveArtifacts(
                artifacts: 'target/site/jacoco/**',
                allowEmptyArchive: true
            )
        }

        success {

            echo 'Jenkins CI succeeded'
        }

        failure {

            echo 'Jenkins CI failed'
        }
    }
}
```

---

# 44. Compare All Three Pipelines

We now have:

```text
GitHub:

.github/workflows/ci.yaml
```

```text
GitLab:

.gitlab-ci.yml
```

```text
Jenkins:

Jenkinsfile
```

All three implement:

```text
             CI PROCESS

Checkout
   ↓
Build
   ↓
Test
   ↓
SonarQube
   ↓
Quality Gate
   ↓
Gitleaks
   ↓
Trivy
   ↓
Docker
   ↓
Trivy Image
   ↓
AWS
   ↓
ECR
```

---

# 45. Platform Comparison

| Concept             | GitHub Actions     | GitLab CI           | Jenkins                         |
| ------------------- | ------------------ | ------------------- | ------------------------------- |
| Pipeline definition | Workflow YAML      | `.gitlab-ci.yml`    | `Jenkinsfile`                   |
| Execution           | Runner             | Runner              | Agent                           |
| Controller          | GitHub platform    | GitLab platform     | Jenkins Controller              |
| Job syntax          | YAML               | YAML                | Groovy                          |
| Secrets             | Secrets            | Variables           | Credentials                     |
| Artifacts           | Actions artifacts  | GitLab artifacts    | Jenkins artifacts               |
| Branch automation   | Native             | Native              | Multibranch                     |
| Reusable pipelines  | Reusable workflows | Includes/templates  | Shared Libraries                |
| Dynamic workers     | Runners            | Runners             | Kubernetes agents               |
| AWS auth            | OIDC               | OIDC                | IAM/OIDC depending architecture |
| Docker              | Buildx             | Runner/build engine | Agent/build engine              |
| Enterprise legacy   | Lower              | Medium/High         | Very High                       |

---

# 46. The Real Skill We Are Building

At this point, don't think:

```text
"I know Jenkins syntax."
```

Think:

```text
"I understand CI architecture."
```

For example:

### GitHub

```text
Workflow
   ↓
Job
   ↓
Runner
```

### GitLab

```text
Pipeline
   ↓
Stage
   ↓
Job
   ↓
Runner
```

### Jenkins

```text
Pipeline
   ↓
Stage
   ↓
Agent
   ↓
Steps
```

The terminology changes.

The engineering concepts remain similar.

---

# 47. Important Production Improvement

Our current Jenkins pipeline still has:

```text
Docker Build
      ↓
Trivy
      ↓
Docker Build again
      ↓
ECR
```

We don't want this.

Our target is:

```text
Build
  ↓
Exact Image
  ↓
Scan
  ↓
Exact Image
  ↓
ECR
```

The same principle applies to GitHub Actions and GitLab.

We will solve this in the dedicated artifact/image promotion chapter.

---

# 48. Our Three-CI Architecture Is Now Clear

The complete learning architecture is:

```text
                    Developers
                         |
                         v
                 Source Repository
                         |
          ┌──────────────┼──────────────┐
          |              |              |
          v              v              v
       GitHub          GitLab        Jenkins
       Actions           CI          Pipeline
          |              |              |
          └──────────────┼──────────────┘
                         |
                         v
                  Common CI Process
                         |
          ┌──────────────┼──────────────┐
          |              |              |
       Quality        Security       Container
          |              |              |
          └──────────────┼──────────────┘
                         |
                         v
                     AWS ECR
                         |
                         v
                    Image Digest
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

---

# 49. CI Section Completed So Far

We have now covered:

```text
09  CI Pipeline Foundation
10  GitHub Actions CI
11  GitLab CI
12  Jenkins CI
```

The next logical step is **not Kubernetes yet**.

We should first harden the CI security and quality controls.

## Next: `13-CI-Security-and-Quality-Gates.md`

We'll consolidate the three pipelines around:

```text
                    CI
                     |
        ┌────────────┼────────────┐
        ↓            ↓            ↓
    SonarQube     Dependency    Secret
                  Scanning      Scanning
        |            |            |
        └────────────┼────────────┘
                     ↓
                 Docker Build
                     ↓
                   Trivy
                     ↓
                    SBOM
                     ↓
                Image Signing
                     ↓
              Security Decision
                     ↓
                  ECR
```

Then we'll build the **SonarQube implementation in depth**, including project configuration, quality profiles, quality gates, coverage, PR analysis, authentication, Jenkins/GitLab/GitHub integration, and what happens when the quality gate fails.
