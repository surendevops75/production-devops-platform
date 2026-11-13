# CHAPTER 192 — MICROSERVICE GOLDEN PATH

## 1. Introduction

In this chapter, we will build a Microservice Golden Path for our 21-microservice e-commerce platform running on AWS EKS.

A Golden Path is a standardized, production-ready approach that helps developers create, build, secure, deploy, monitor, and operate microservices without designing the entire infrastructure and delivery process from scratch.

Imagine a developer wants to create a new microservice called `wishlist-service`.

Without a Golden Path, the developer might need to figure out:

- How to create the GitHub repository.
- Which Dockerfile to use.
- How to write Kubernetes manifests.
- How to configure Helm.
- How to create CI/CD pipelines.
- How to integrate SonarQube and Trivy.
- How to publish images to ECR.
- How to configure ArgoCD.
- How to create ServiceAccounts.
- How to configure monitoring and alerts.
- How to deploy safely into production.

This creates inconsistent implementations across teams.

With a Golden Path, the developer provides a few service details, and the platform provides the standard repository structure, pipeline, deployment configuration, security controls, and operational requirements.

Our objective: A developer should be able to onboard a new microservice using a repeatable process while the platform team maintains the underlying standards.

## 2. What are we building?

Our Golden Path will provide the following capabilities:

| Capability        | Standard provided                           |
| ----------------- | ------------------------------------------- |
| Repository        | Standard GitHub repository structure        |
| Application       | Java, Node.js, and Python starter templates |
| Containerization  | Standard Dockerfile                         |
| CI/CD             | Reusable GitHub Actions workflows           |
| Security          | SonarQube and Trivy integration             |
| Image management  | AWS ECR                                     |
| Infrastructure    | Reusable Terraform modules                  |
| Kubernetes        | Standard Helm chart                         |
| Workload identity | ServiceAccount templates                    |
| GitOps            | ArgoCD application generator                |
| Deployment        | DEV, QA, STAGING, PROD                      |
| Monitoring        | Prometheus and Grafana                      |
| Logging           | ELK with Logstash                           |
| Alerting          | Standard alert rules                        |
| Documentation     | README, runbooks, ownership                 |
| Governance        | Security and platform policies              |

The Golden Path is not a replacement for developer ownership.

It provides approved defaults, automation, and guardrails while allowing controlled customization when a service has a genuine special requirement.

## 3. Production architecture

[Philip Oludolamu | DevOps, Cloud & Platform Engineer](https://images.openai.com/static-rsc-4/2-OmAxrWRJ1XgUfmjzixV5tPcFxa3gcGuUUzp67Pa3vjv7vibf08Qfdqpejghen2mUf2zndpntFzIMniRhndQ6CkAFO1p3tQeMRP_ACTqgwN-naNSNKZTrK6tF8SmCp3l2d1RlUNxdAVOEuc-zY5RlSirvax4y_yQRSf-WAqIJk?purpose=inline)

[philipoludolamu.com](https://philipoludolamu.com/?utm_source=chatgpt.com)

```
                    Developer
                        |
                        v
                Golden Path Request
                        |
                        v
                 Platform Templates
                        |
             +----------+----------+
             |          |          |
             v          v          v
          GitHub     Application   Helm
          Repo       Template      Chart
             |          |          |
             +----------+----------+
                        |
                        v
                  GitHub Actions
                        |
             +----------+----------+
             |          |          |
             v          v          v
           Tests     SonarQube    Trivy
             |          |          |
             +----------+----------+
                        |
                        v
                  Docker Build
                        |
                        v
                       ECR
                        |
                        v
                GitOps Repository
                        |
                        v
                     ArgoCD
                        |
                        v
                     AWS EKS
                        |
             +----------+----------+
             |          |          |
             v          v          v
           DEV         QA        STAGING
                        |
                        v
                  PROD Approval
                        |
                        v
                 Production EKS
                        |
             +----------+----------+
             |          |          |
             v          v          v
         Prometheus  Grafana      ELK
```

The important separation is:

- Developers own application code and business logic.
- Platform Engineering owns the Golden Path and reusable standards.
- DevOps/Cloud Engineering owns infrastructure and delivery automation.
- Security owns security policies and reviews.
- Application teams own service-level reliability and operational behavior.

## 4. Repository architecture

We will maintain the Golden Path in a dedicated platform repository.

```
company-ecommerce/
│
├── platform-golden-path/
│   │
│   ├── templates/
│   │   ├── java-springboot/
│   │   ├── nodejs/
│   │   └── python/
│   │
│   ├── standards/
│   │   ├── docker/
│   │   ├── kubernetes/
│   │   ├── helm/
│   │   ├── cicd/
│   │   ├── security/
│   │   └── observability/
│   │
│   ├── generators/
│   │   ├── service-generator/
│   │   ├── repository-generator/
│   │   └── gitops-generator/
│   │
│   ├── workflows/
│   │   ├── java-ci.yml
│   │   ├── nodejs-ci.yml
│   │   ├── python-ci.yml
│   │   └── docker-build.yml
│   │
│   ├── charts/
│   │   └── microservice/
│   │
│   ├── policies/
│   │   ├── security/
│   │   ├── rbac/
│   │   └── deployment/
│   │
│   ├── scripts/
│   │   ├── create-service.sh
│   │   ├── validate-service.sh
│   │   └── register-service.sh
│   │
│   └── README.md
│
├── infrastructure/
│
├── gitops/
│
└── applications/
    ├── auth-service/
    ├── user-service/
    ├── customer-service/
    ├── catalog-service/
    ├── product-service/
    ├── inventory-service/
    ├── pricing-service/
    ├── promotion-service/
    ├── cart-service/
    ├── order-service/
    ├── payment-service/
    ├── fraud-service/
    ├── shipping-service/
    ├── delivery-service/
    ├── notification-service/
    ├── search-service/
    ├── recommendation-service/
    ├── review-service/
    ├── tax-service/
    ├── invoice-service/
    └── reporting-service/
```

We can keep each microservice in a separate GitHub repository while maintaining the shared templates in the platform repository.

For this implementation, we will use separate application repositories, a central platform repository, an infrastructure repository, and a GitOps repository.

## 5. Define the Golden Path service contract

Before creating templates, we need to define what information is required to onboard a service.

Example service configuration:

`wishlist-service.yaml`

```
service:
  name: wishlist-service

  team: customer-experience

  owner: customer-platform

  language: java

  framework: spring-boot

  port: 8080

  replicas:
    dev: 1
    qa: 2
    staging: 2
    prod: 3

  resources:
    requests:
      cpu: 200m
      memory: 256Mi

    limits:
      cpu: 1000m
      memory: 1Gi

  health:
    liveness: /actuator/health/liveness
    readiness: /actuator/health/readiness

  deployment:
    strategy: rolling

  security:
    serviceAccount: wishlist-service
    automountServiceAccountToken: false

  observability:
    prometheus: true
    grafana: true
    logging: true

  environments:
    - dev
    - qa
    - staging
    - prod
```

This file is the service's configuration contract.

It tells the generator what to create.

The generator should validate the values rather than blindly copying them into production configuration.

For example:

- Service names must follow naming standards.
- Ports must be valid.
- Production replicas must meet minimum policy.
- The owner must be provided.
- Health endpoints must be defined.
- Security settings must follow platform rules.

## 6. Create the Java Spring Boot starter template

We will use Java Spring Boot as the first example because our platform includes Java-based microservices.

Directory:

```
templates/java-springboot/
│
├── pom.xml
├── Dockerfile
├── .dockerignore
├── .gitignore
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   └── resources/
│   │
│   └── test/
│
├── helm/
│   └── values.yaml
│
├── .github/
│   └── workflows/
│       └── ci.yml
│
└── README.md
```

### 6.1 Spring Boot application

Example:

`src/main/java/com/company/wishlist/WishlistApplication.java`

```
package com.company.wishlist;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class WishlistApplication {

    public static void main(String[] args) {

        SpringApplication.run(
            WishlistApplication.class,
            args
        );

    }

}
```

### 6.2 Health endpoint configuration

`application.yaml`

```
server:
  port: 8080

spring:
  application:
    name: wishlist-service

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus

  endpoint:
    health:
      probes:
        enabled: true

  metrics:
    tags:
      application: wishlist-service
```

This enables Spring Boot health probes and Prometheus metrics when the required Micrometer Prometheus registry dependency is included.

### 6.3 Maven configuration

`pom.xml`

```
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.company</groupId>

    <artifactId>wishlist-service</artifactId>

    <version>1.0.0</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.0</version>
        <relativePath/>
    </parent>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

For production, pin and regularly review dependency versions rather than copying this example version indefinitely.

## 7. Standard Dockerfile

File:

`templates/java-springboot/Dockerfile`

```
FROM eclipse-temurin:21-jdk AS build

WORKDIR /app

COPY pom.xml .

COPY src ./src

RUN ./mvnw -B dependency:go-offline

RUN ./mvnw -B clean package -DskipTests


FROM eclipse-temurin:21-jre

WORKDIR /app

RUN groupadd --system appgroup && \
    useradd --system --gid appgroup appuser

COPY --from=build \
    /app/target/*.jar \
    /app/application.jar

USER appuser

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app/application.jar"]
```

This is a starter example. The template should include the Maven wrapper files and a verified build configuration.

Production improvements should include:

- Pinning base images by digest.
- Updating vulnerable dependencies.
- Using a minimal runtime image.
- Running as a non-root user.
- Avoiding secrets in image layers.
- Scanning the final image.
- Generating an SBOM.
- Signing images where supported by the organization's policy.

`.dockerignore`:

```
.git
.github
target
.idea
.vscode
*.log
.env
secrets/
```

Never copy credentials or local secret files into the image.

## 8. Create the reusable microservice Helm chart

Directory:

```
platform-golden-path/charts/microservice/
│
├── Chart.yaml
├── values.yaml
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    ├── ingress.yaml
    └── servicemonitor.yaml
```

### 8.1 Chart.yaml

```
apiVersion: v2

name: microservice

description: Standard production microservice chart

type: application

version: 1.0.0

appVersion: "1.0.0"
```

### 8.2 values.yaml

```
replicaCount: 2

image:
  repository: example/wishlist-service
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

containerPort: 8080

serviceAccount:
  create: true
  name: wishlist-service
  automountServiceAccountToken: false
  annotations: {}

resources:
  requests:
    cpu: 200m
    memory: 256Mi

  limits:
    cpu: 1000m
    memory: 1Gi

probes:
  liveness:
    enabled: true
    path: /actuator/health/liveness

  readiness:
    enabled: true
    path: /actuator/health/readiness

  startup:
    enabled: true
    path: /actuator/health

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

monitoring:
  enabled: true
```

The example uses `latest` only as a placeholder. Production GitOps values should use immutable image tags or digests.

### 8.3 \_helpers.tpl

```
{{- define "microservice.name" -}}
{{- .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end }}

{{- define "microservice.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end }}

{{- define "microservice.labels" -}}
app.kubernetes.io/name: {{ include "microservice.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: Helm
{{- end }}
```

### 8.4 deployment.yaml

```
apiVersion: apps/v1

kind: Deployment

metadata:
  name: {{ .Release.Name }}

  labels:
    {{- include "microservice.labels" . | nindent 4 }}

spec:
  replicas: {{ .Values.replicaCount }}

  selector:
    matchLabels:
      app.kubernetes.io/instance: {{ .Release.Name }}

  template:
    metadata:
      labels:
        app.kubernetes.io/instance: {{ .Release.Name }}

    spec:
      serviceAccountName: {{ .Values.serviceAccount.name }}

      containers:
        - name: {{ .Chart.Name }}

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: {{ .Values.image.pullPolicy }}

          ports:
            - name: http
              containerPort: {{ .Values.containerPort }}

          resources:
            {{- toYaml .Values.resources | nindent 12 }}

          {{- if .Values.probes.startup.enabled }}
          startupProbe:
            httpGet:
              path: {{ .Values.probes.startup.path }}
              port: http
            periodSeconds: 10
            failureThreshold: 30
          {{- end }}

          {{- if .Values.probes.liveness.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: http
            periodSeconds: 10
            failureThreshold: 3
          {{- end }}

          {{- if .Values.probes.readiness.enabled }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: http
            periodSeconds: 5
            failureThreshold: 3
          {{- end }}
```

The chart must also render the ServiceAccount using the same configured name.

The template should validate that `serviceAccount.create: false` references a real pre-existing account.

### 8.5 service.yaml

```
apiVersion: v1

kind: Service

metadata:
  name: {{ .Release.Name }}

  labels:
    {{- include "microservice.labels" . | nindent 4 }}

spec:
  type: {{ .Values.service.type }}

  selector:
    app.kubernetes.io/instance: {{ .Release.Name }}

  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
```

### 8.6 servicemonitor.yaml

For clusters where Prometheus Operator and its ServiceMonitor CRD are installed:

```
{{- if .Values.monitoring.enabled }}

apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:
  name: {{ .Release.Name }}

spec:
  selector:
    matchLabels:
      app.kubernetes.io/instance: {{ .Release.Name }}

  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 30s

{{- end }}
```

This example assumes the ServiceMonitor CRD is installed and the Prometheus instance selects this ServiceMonitor. The chart should support disabling this resource where that integration is not present.

## 9. Reusable GitHub Actions workflow

Instead of maintaining a different CI pipeline for every microservice, the platform team maintains reusable workflows.

Directory:

```
platform-golden-path/workflows/
│
├── java-ci.yml
├── nodejs-ci.yml
├── python-ci.yml
└── docker-build.yml
```

### 9.1 Java CI workflow

File:

`workflows/java-ci.yml`

```
name: Reusable Java CI

on:
  workflow_call:
    inputs:
      java-version:
        required: false
        type: string
        default: "21"

      working-directory:
        required: false
        type: string
        default: "."

jobs:

  build:
    runs-on: ubuntu-latest

    permissions:
      contents: read

    steps:

      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: ${{ inputs.java-version }}
          cache: maven

      - name: Build
        working-directory: ${{ inputs.working-directory }}
        run: ./mvnw -B clean verify

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: "**/target/surefire-reports/"
```

Pin third-party Actions to reviewed commit SHAs in the production version of the workflow. The version tags here make the example easier to read.

### 9.2 Application repository workflow

File:

`wishlist-service/.github/workflows/ci.yml`

```
name: Wishlist Service CI

on:
  pull_request:
    branches:
      - main

  push:
    branches:
      - main

jobs:

  java-ci:
    uses: company-ecommerce/platform-golden-path/.github/workflows/java-ci.yml@v1

    with:
      java-version: "21"
```

This application repository calls the centrally maintained workflow.

If the platform team improves the Java workflow, application repositories can adopt the approved version without copying the entire pipeline.

## 10. Security integration

Every microservice must follow the organization's security scanning standards.

Our baseline includes:

- SonarQube for source-code quality and configured static analysis.
- Trivy for image and dependency vulnerability scanning.
- Secret detection.
- Dependency review.
- SBOM generation.
- Image signing and provenance where required.

### Example security workflow stages

```
Developer Pull Request
        |
        v
     Unit Tests
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
    Trivy Image Scan
        |
        v
     SBOM Generate
        |
        v
    Publish Approved Image
```

### Example Trivy command

```
trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  "$IMAGE"
```

This example fails the job when matching HIGH or CRITICAL findings are detected.

In production, define an approved exception process for accepted vulnerabilities, with ownership and expiry dates. Also configure the scanner to use current vulnerability databases.

### Example SonarQube command

```
mvn sonar:sonar \
  -Dsonar.projectKey=wishlist-service \
  -Dsonar.host.url="$SONAR_HOST_URL"
```

The authentication token should be provided through an approved GitHub Actions secret or secure integration, never committed to the repository.

## 11. Image publishing to AWS ECR

The Golden Path should standardize image naming and publishing.

Example image:

```
123456789012.dkr.ecr.ap-south-1.amazonaws.com/wishlist-service
```

Use immutable identifiers such as:

```
wishlist-service:git-<commit-sha>
```

or an image digest:

```
wishlist-service@sha256:<digest>
```

The production GitOps configuration should reference the verified image digest or another immutable artifact identifier.

### Example ECR publishing workflow steps

```
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_GITHUB_ACTIONS_ROLE }}
    aws-region: ap-south-1

- name: Login to ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v2

- name: Build image
  run: |
    docker build \
      -t "$ECR_REGISTRY/wishlist-service:$GITHUB_SHA" .

- name: Push image
  run: |
    docker push \
      "$ECR_REGISTRY/wishlist-service:$GITHUB_SHA"
```

This is an illustrative fragment, not a complete workflow. The full implementation must define the ECR registry value, repository lifecycle, permissions, scan gates, and image provenance requirements.

GitHub Actions should authenticate using OIDC with a dedicated CI IAM role. Do not reuse the Payment Service or other application runtime role.

## 12. GitOps integration

After publishing the image, the pipeline should update the GitOps repository.

It should not directly deploy to production using `kubectl apply`.

### GitOps repository structure

```
gitops/
│
├── apps/
│   └── wishlist-service.yaml
│
└── environments/
    │
    ├── dev/
    │   └── wishlist-service/
    │       └── values.yaml
    │
    ├── qa/
    │   └── wishlist-service/
    │       └── values.yaml
    │
    ├── staging/
    │   └── wishlist-service/
    │       └── values.yaml
    │
    └── prod/
        └── wishlist-service/
            └── values.yaml
```

### DEV values

```
replicaCount: 1

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/wishlist-service
  tag: git-abc123

serviceAccount:
  create: true
  name: wishlist-service
  automountServiceAccountToken: false
```

### PROD values

```
replicaCount: 3

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/wishlist-service
  digest: sha256:REPLACE_WITH_VERIFIED_DIGEST

serviceAccount:
  create: true
  name: wishlist-service
  automountServiceAccountToken: false
```

The chart must support digest-based image references if this format is used.

### ArgoCD Application example

`gitops/apps/wishlist-service.yaml`

```
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:
  name: wishlist-service-dev

  namespace: argocd

spec:
  project: ecommerce

  source:
    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: environments/dev/wishlist-service

    helm:
      valueFiles:
        - values.yaml

  destination:
    server: https://kubernetes.default.svc

    namespace: ecommerce-dev

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

The Application must point to the actual chart source or a directory containing a chart reference and values configuration. The example above assumes that the environment directory is structured accordingly.

Production Applications should use the organization's approval and sync policy rather than blindly copying the automated DEV settings.

## 13. Create the service generator

Now we will automate the onboarding process.

The developer should not manually create every directory and configuration file.

The platform generator will:

1. Read service metadata.
2. Validate the service name.
3. Select the application template.
4. Generate the repository files.
5. Add the standard Dockerfile.
6. Add CI workflow references.
7. Add Helm values.
8. Generate documentation.
9. Prepare GitOps onboarding information.

For this example, we will implement a simple Bash-based generator.

File:

`platform-golden-path/scripts/create-service.sh`

```
#!/usr/bin/env bash

set -euo pipefail

SERVICE_NAME="${1:-}"
LANGUAGE="${2:-}"

if [[ -z "$SERVICE_NAME" || -z "$LANGUAGE" ]]; then
    echo "Usage: ./create-service.sh <service-name> <language>"
    echo "Supported languages: java, nodejs, python"
    exit 1
fi

if [[ ! "$SERVICE_NAME" =~ ^[a-z][a-z0-9-]{1,50}$ ]]; then
    echo "Invalid service name: $SERVICE_NAME"
    exit 1
fi

TEMPLATE_DIR="templates/$LANGUAGE"

if [[ ! -d "$TEMPLATE_DIR" ]]; then
    echo "Template not found: $TEMPLATE_DIR"
    exit 1
fi

OUTPUT_DIR="../generated/$SERVICE_NAME"

if [[ -e "$OUTPUT_DIR" ]]; then
    echo "Output directory already exists: $OUTPUT_DIR"
    exit 1
fi

mkdir -p "$OUTPUT_DIR"

cp -R "$TEMPLATE_DIR/." "$OUTPUT_DIR/"

echo "Service generated successfully."

echo "Service: $SERVICE_NAME"

echo "Language: $LANGUAGE"

echo "Location: $OUTPUT_DIR"
```

Make it executable:

```
chmod +x scripts/create-service.sh
```

Generate a service:

```
./scripts/create-service.sh wishlist-service java
```

The generator currently copies a template. It does not yet replace every placeholder or create the remote GitHub repository.

That is intentional: the next implementation step is to add metadata rendering and repository provisioning with approved GitHub credentials.

## 14. Add metadata substitution

We need to replace placeholder names in generated files.

A simple approach is to use a template engine such as Cookiecutter, Copier, or a controlled Python generator.

For example, with a Python generator, we can render files from a service contract.

File:

`generators/service-generator/generate.py`

```
import argparse
import re
import shutil
from pathlib import Path

parser = argparse.ArgumentParser()

parser.add_argument("--name", required=True)
parser.add_argument("--language", required=True)

args = parser.parse_args()

service_name = args.name
language = args.language

if not re.fullmatch(r"[a-z][a-z0-9-]{1,50}", service_name):
    raise SystemExit("Invalid service name")

template_dir = Path("templates") / language

output_dir = Path("generated") / service_name

if not template_dir.is_dir():
    raise SystemExit("Template not found")

if output_dir.exists():
    raise SystemExit("Output already exists")

shutil.copytree(template_dir, output_dir)

for file_path in output_dir.rglob("*"):

    if not file_path.is_file():
        continue

    try:
        content = file_path.read_text()
    except UnicodeDecodeError:
        continue

    content = content.replace(
        "{{SERVICE_NAME}}",
        service_name
    )

    file_path.write_text(content)

print(f"Generated {service_name}")
```

This is a basic implementation. A production generator should use a proper template engine, validate the full service contract, preserve binary files, support versioned templates, and test every generated artifact.

## 15. GitHub repository provisioning

After generation, the platform workflow needs to create the new repository.

Example using GitHub CLI:

```
gh repo create \
  company-ecommerce/wishlist-service \
  --private \
  --source ../generated/wishlist-service \
  --remote origin \
  --push
```

The platform automation identity needs only the permissions required to create and configure the target repository.

Do not embed a personal access token in the script.

Use an approved GitHub App or other organization-approved identity integration.

### Repository standards

Every generated repository should include:

```
wishlist-service/
│
├── src/
├── Dockerfile
├── pom.xml
├── helm/
├── .github/
│   └── workflows/
│       └── ci.yml
├── README.md
├── CODEOWNERS
├── SECURITY.md
└── .gitignore
```

The platform team should configure:

- Protected branches.
- Required pull request reviews.
- Required CI checks.
- CODEOWNERS.
- Secret scanning.
- Dependency review.
- Repository visibility.
- Access permissions.
- Approved GitHub Actions policies.

Repository creation is not complete until these controls are configured.

## 16. GitOps onboarding generator

Chapter 191 established the GitOps generator concept.

Now we integrate it into the Golden Path.

For each new service, generate the environment directories:

```
gitops/environments/
│
├── dev/wishlist-service/
│   └── values.yaml
│
├── qa/wishlist-service/
│   └── values.yaml
│
├── staging/wishlist-service/
│   └── values.yaml
│
└── prod/wishlist-service/
    └── values.yaml
```

Generate the ArgoCD Application configuration:

```
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:
  name: wishlist-service-dev
  namespace: argocd

spec:
  project: ecommerce

  source:
    repoURL: https://github.com/company-ecommerce/gitops.git
    targetRevision: main
    path: environments/dev/wishlist-service

  destination:
    server: https://kubernetes.default.svc
    namespace: ecommerce-dev
```

The generator should also validate:

- The namespace exists or is provisioned.
- The Helm chart version is approved.
- The ServiceAccount exists or is generated.
- AWS workload identity is configured if required.
- The image repository exists.
- The team owner is recorded.
- The service has resource requests and limits.
- Monitoring configuration is present.

Do not generate a production application with elevated permissions simply because the developer requests it.

## 17. Developer experience: complete onboarding example

Let's imagine a developer needs a new service:

`wishlist-service`

The developer submits:

```
service:
  name: wishlist-service

  team: customer-experience

  language: java

  framework: spring-boot

  port: 8080

  owner: customer-platform

  database:
    required: true

  messaging:
    required: false

  aws:
    required: true
    identity: eks-pod-identity

  monitoring:
    required: true

  environments:
    - dev
    - qa
    - staging
    - prod
```

The platform workflow processes the request.

### What happens?

1. Repository generation

   Creates `company-ecommerce/wishlist-service` using the Java template.
2. Application scaffold

   Adds Spring Boot source structure, Maven configuration, health endpoints, and tests.
3. CI/CD configuration

   Connects the repository to reusable GitHub Actions workflows.
4. Security configuration

   Adds SonarQube and Trivy scanning stages and the organization's required security checks.
5. Infrastructure request

   Creates a reviewed request for the required database, IAM permissions, and workload identity.
6. GitOps generation

   Creates DEV, QA, STAGING, and PROD deployment configuration.
7. Monitoring setup

   Adds the approved Prometheus scrape configuration and Grafana dashboard conventions.
8. Developer handoff

   Provides repository URL, deployment instructions, ownership details, and operational runbooks.

The developer can now focus on wishlist business functionality rather than repeatedly rebuilding platform infrastructure.

## 18. Golden Path customization model

Not every microservice has identical requirements.

For example:

- Payment Service has strict security requirements.
- Reporting Service may require scheduled workloads.
- Notification Service may consume messages.
- Search Service may need more memory.
- Inventory Service may require special scaling behavior.

We should support controlled customization.

| Requirement | Standard approach            | Customization                      |
| ----------- | ---------------------------- | ---------------------------------- |
| CPU/memory  | Standard resource defaults   | Service-specific sizing            |
| Replicas    | Environment defaults         | Approved scaling values            |
| IAM         | Least-privilege identity     | Reviewed service-specific policy   |
| Networking  | Standard NetworkPolicy       | Approved dependency rules          |
| Monitoring  | Common dashboards and alerts | Service-specific metrics           |
| Deployment  | Standard rolling deployment  | Approved canary strategy           |
| Database    | Approved connection pattern  | Service-specific database design   |
| Messaging   | Approved integration pattern | Service-specific queue/topic setup |

A Golden Path should make the safe option easy, not prevent legitimate engineering decisions.

## 19. Platform governance and versioning

The platform team must own and maintain the templates.

For example:

```
platform-golden-path/
│
├── releases/
│   ├── v1/
│   ├── v2/
│   └── v3/
│
├── templates/
├── workflows/
├── charts/
└── policies/
```

Use versioned references:

```
uses: company-ecommerce/platform-golden-path/.github/workflows/java-ci.yml@v1
```

The `v1` reference should be a controlled release reference. For production security, pin the actual workflow dependency to a reviewed immutable commit SHA where required.

### Template upgrade process

```
Platform Team Changes Template
             |
             v
         Pull Request
             |
             v
       Automated Testing
             |
             v
        Security Review
             |
             v
       Release New Version
             |
             v
      Application Adoption
             |
             v
       GitOps Deployment
```

Do not automatically push breaking changes to all 21 application repositories.

Publish migration instructions and provide a controlled adoption period.

## 20. Production testing strategy

The Golden Path itself must be tested.

We should test not just the generated application but also the templates, pipelines, policies, and generated Kubernetes resources.

### Test 1 — Generator validation

```
python generators/service-generator/generate.py \
  --name wishlist-service \
  --language java
```

Expected:

- Valid service name accepted.
- Unsupported language rejected.
- Existing destination rejected.
- Required files generated.

### Test 2 — Docker build

```
docker build \
  -t wishlist-service:test .
```

### Test 3 — Unit tests

```
./mvnw clean verify
```

### Test 4 — Helm validation

```
helm lint ./charts/microservice
```

```
helm template wishlist-service \
  ./charts/microservice \
  -n ecommerce-dev \
  -f values.yaml \
  > rendered.yaml
```

### Test 5 — Kubernetes schema validation

```
kubeconform -strict rendered.yaml
```

### Test 6 — Security validation

```
trivy config .
```

### Test 7 — Policy validation

```
conftest test rendered.yaml
```

### Test 8 — GitOps validation

Check that:

- ArgoCD Application manifests are valid.
- The destination namespace is correct.
- The repository path exists.
- The chart version is available.
- The image reference is immutable.
- Production approvals are enforced.

### Test 9 — Integration deployment

Deploy a generated test service into DEV.

Validate:

```
kubectl get deployments -n ecommerce-dev
```

```
kubectl get pods -n ecommerce-dev
```

```
kubectl get svc -n ecommerce-dev
```

```
kubectl get serviceaccounts -n ecommerce-dev
```

### Test 10 — Observability validation

Confirm:

- Prometheus discovers the application metrics.
- Grafana displays the service metrics.
- Logstash receives the application logs.
- ELK indexes the logs.
- Alerts trigger under controlled test conditions.

## 21. Operational standards

A generated service must provide the following operational information.

### README.md

Every service repository should contain:

```
# Wishlist Service

## Overview

Wishlist microservice for the e-commerce platform.

## Technology

- Java 21
- Spring Boot
- Maven
- Docker
- Kubernetes
- Helm

## Local Development

./mvnw spring-boot:run

## Build

./mvnw clean package

## Docker

docker build -t wishlist-service .

## Health Endpoints

/actuator/health
/actuator/health/liveness
/actuator/health/readiness

## Metrics

/actuator/prometheus

## Deployment

Managed through GitOps and ArgoCD.

## Monitoring

Prometheus and Grafana.

## Logging

ELK Stack.

## Ownership

Team: customer-experience

Service owner: customer-platform
```

### CODEOWNERS

Example:

```
* @company-ecommerce/customer-platform

/.github/ @company-ecommerce/platform-team

/helm/ @company-ecommerce/platform-team
```

This is an illustrative ownership configuration. The organization must use its actual GitHub team names and repository permissions.

### Runbook

Each service should document:

- How to identify a failed deployment.
- How to check Pod health.
- How to inspect logs.
- How to identify database connection failures.
- How to identify queue processing failures.
- How to validate dependencies.
- How to initiate a rollback.
- Who to contact during incidents.

## 22. Golden Path production failure scenarios

### Scenario 1 — Generated service has incorrect ServiceAccount

Problem: The application starts but cannot access its required AWS resource.

Check:

```
kubectl get deployment wishlist-service \
  -n ecommerce-dev \
  -o jsonpath='{.spec.template.spec.serviceAccountName}'
```

Then verify the EKS identity association and IAM role.

Preventive action: Add a Golden Path validation rule that requires an explicit ServiceAccount and verifies the expected workload identity configuration.

### Scenario 2 — Reusable CI workflow breaks multiple services

Problem: The platform team updates the Java CI workflow, and several application repositories fail.

Root cause: An incompatible shared workflow change was released.

Preventive actions:

- Version reusable workflows.
- Test against representative services.
- Publish release notes.
- Maintain backward compatibility.
- Provide rollback to the previous workflow version.

### Scenario 3 — Generated Helm chart causes production failure

Problem: A chart update changes readiness probe behavior, causing Pods to remain unready.

Preventive actions:

- Add Helm rendering tests.
- Test with representative application workloads.
- Validate probe endpoints.
- Test upgrades in DEV and STAGING.
- Require production promotion approval.

### Scenario 4 — Image is built but cannot be deployed

Problem: The image exists in ECR, but ArgoCD deployment fails.

Check:

```
kubectl describe pod <pod-name> \
  -n ecommerce-dev
```

Possible causes:

- Incorrect ECR repository.
- Wrong image tag or digest.
- Node or execution-role image pull permissions.
- Registry connectivity.
- Image architecture mismatch.

Remember that ECR image-pull permissions are separate from the application's runtime IAM permissions.

### Scenario 5 — Generated service has no monitoring

Problem: The service is healthy, but Prometheus has no metrics.

Check:

```
kubectl get servicemonitor -n ecommerce-dev
```

Check the Prometheus target page and verify:

- ServiceMonitor selector.
- Namespace selector.
- Service labels.
- Metrics endpoint.
- Port name.
- Application metrics dependency.

Preventive action: Make monitoring validation part of the service onboarding acceptance tests.

## 23. Golden Path KPIs

We should measure whether the Golden Path actually improves developer productivity and platform quality.

| KPI                       | What it measures                                        |
| ------------------------- | ------------------------------------------------------- |
| Time to first deployment  | Time from service request to first DEV deployment       |
| Template adoption         | Percentage of services using approved templates         |
| Pipeline success rate     | Successful pipeline executions                          |
| Deployment frequency      | How often teams deploy                                  |
| Change failure rate       | Percentage of changes causing failures                  |
| Mean time to recovery     | Time required to restore service                        |
| Security compliance       | Percentage of services meeting required security checks |
| Configuration drift       | Difference between desired and actual configuration     |
| Template upgrade adoption | Percentage of services on supported template versions   |

Example targets for a platform improvement initiative:

- New service scaffold generated within minutes.
- First DEV deployment within one working day, subject to infrastructure dependencies.
- All production services use approved CI/CD workflows.
- All services have defined owners.
- All production services have health checks and monitoring.
- All application workloads use explicit ServiceAccounts.
- All production images are immutable and traceable.

These are proposed targets, not measured results.

## 24. Complete Golden Path lifecycle

The following is the end-to-end lifecycle for a new microservice.

```
              DEVELOPER REQUEST
                     |
                     v
              SERVICE CONTRACT
                     |
                     v
             TEMPLATE GENERATION
                     |
                     v
             GITHUB REPOSITORY
                     |
                     v
             APPLICATION CODING
                     |
                     v
               PULL REQUEST
                     |
                     v
               CI PIPELINE
                     |
             +-------+-------+
             |       |       |
             v       v       v
           Tests  SonarQube Trivy
             |       |       |
             +-------+-------+
                     |
                     v
              DOCKER BUILD
                     |
                     v
                   ECR
                     |
                     v
             GITOPS CONFIG UPDATE
                     |
                     v
                  ARGOCD
                     |
                     v
                DEV DEPLOY
                     |
                     v
                 QA DEPLOY
                     |
                     v
              STAGING DEPLOY
                     |
                     v
             PRODUCTION APPROVAL
                     |
                     v
                PROD DEPLOY
                     |
                     v
          PROMETHEUS / GRAFANA / ELK
                     |
                     v
             OPERATIONS AND RCA
                     |
                     v
             PLATFORM IMPROVEMENT
```

## 25. Production Definition of Done

### Platform templates

- Java Spring Boot starter template created.
- Node.js starter template planned or implemented.
- Python starter template planned or implemented.
- Standard Dockerfile provided.
- Standard Helm chart created.
- ServiceAccount integration included.
- Resource defaults configured.
- Health probes configured.
- Monitoring integration included.

### CI/CD

- Reusable GitHub Actions workflows created.
- Unit tests included.
- SonarQube integration included.
- Trivy integration included.
- ECR publishing configured.
- GitHub Actions OIDC authentication configured.
- Immutable image references enforced.

### GitOps

- ArgoCD Application configuration generated.
- DEV configuration generated.
- QA configuration generated.
- STAGING configuration generated.
- PROD configuration generated.
- Production approval controls enforced.
- No direct CI deployment to production.

### Security and governance

- Service ownership recorded.
- CODEOWNERS configured.
- IAM roles reviewed.
- Kubernetes RBAC validated.
- Secrets managed through approved mechanisms.
- Security scanning gates enabled.
- Template versions maintained.
- Upgrade and rollback processes documented.

### Operations

- Prometheus metrics available.
- Grafana dashboard available.
- ELK logging integration verified.
- Alerting configured.
- Service runbook created.
- Incident ownership documented.
- Deployment and rollback validated.

## 26. Phase 20 completion

Phase 20 — Platform Engineering completed

We have covered the platform engineering building blocks:

| Chapter | Topic                             |
| ------- | --------------------------------- |
| 182     | Reusable Helm Templates           |
| 183     | Reusable GitHub Actions           |
| 184     | Terraform Modules                 |
| 185     | GitHub Actions Reusable Workflows |
| 186     | Security Policy Templates         |
| 187     | Monitoring Templates              |
| 188     | Alert Rule Templates              |
| 189     | NetworkPolicy Templates           |
| 190     | ServiceAccount Templates          |
| 191     | GitOps Generators                 |
| 192     | Microservice Golden Path          |

The next phase moves from designing reusable platform components to running a complete production-style simulation.

# NEXT: PHASE 21 — FINAL PRODUCTION SIMULATION

We will now use the platform built in the previous chapters to simulate a real developer change from the initial feature request through production deployment, monitoring, incident handling, rollback, recovery, and RCA.

Chapter 193 — Developer Feature Workflow

The next chapter will demonstrate how a developer picks up a Jira feature, creates a branch, implements a change in one of the 21 microservices, commits the code, and follows the organization's development standards before opening a pull request.