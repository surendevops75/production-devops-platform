# PHASE 21 — FINAL PRODUCTION SIMULATION

# CHAPTER 196 — SECURITY SCANNING PIPELINE

Filename: `196-Security-Scanning-Pipeline.md`

Project: Enterprise E-Commerce Platform — 21 Microservices\
Service: `order-service`\
Cloud: AWS\
Region: `ap-south-1`\
Kubernetes: Amazon EKS\
Application: Java 21 / Spring Boot / Maven\
CI/CD: GitHub Actions, GitLab CI/CD\
Security Tools: SonarQube, Trivy, Veracode\
Container Registry: Amazon ECR\
Deployment: Helm + ArgoCD\
GitOps: GitHub repository\
Issue: `ECOM-4821` — Order Status Tracking Feature\
Source Commit: `d4e5f6a...` — illustrative placeholder

# 1. Objective

In Chapter 195, we completed the CI pipeline execution for the Order Status Tracking feature.

The source code has passed the initial build and test stages.

However, a successful build does not mean that the application is secure.

Before building and publishing a production Docker image, we must verify that the source code, dependencies, configuration files, and application packages do not contain unacceptable security risks.

In this chapter, we implement a production-oriented DevSecOps security scanning pipeline.

Our goal is to identify security problems as early as possible, before they reach production.

By the end of this chapter, we should understand:

- How security scanning fits into a CI/CD pipeline.
- How SonarQube analyzes source code.
- How Trivy scans source repositories and dependencies.
- How to detect exposed credentials and secrets.
- How Veracode integrates into the pipeline.
- How security policies determine whether a pipeline passes or fails.
- How to handle false positives.
- How to manage vulnerability exceptions.
- How to generate security reports.
- How to troubleshoot security pipeline failures.
- How to prevent insecure code from moving to the image-building stage.

Important: This chapter implements source-level and dependency-level security checks. The complete Docker image build is Chapter 197. ECR publishing is Chapter 198. We will explain how image scanning fits into the overall design, but we will not publish an image or claim that an image scan has already passed.

# 2. Where Security Scanning Fits in the Production Pipeline

Our pipeline now has the following flow:

Developer commits source code

Feature branch → Pull Request

Chapter 195 — CI Pipeline

Compile → Unit tests → Integration tests → Package

Current chapter

Chapter 196 — Security Scanning

1\. SonarQube

Code quality and security analysis

2\. Trivy filesystem and configuration scans

Dependencies, vulnerabilities, misconfigurations

3\. Secret scanning

Credentials, tokens, private keys

4\. Veracode

Application security analysis

5\. Security policy evaluation

Pass, fail, or require an approved exception

Chapter 197 — Docker Image Build

Build the approved application artifact into a container image

Chapter 198 — ECR Image Publishing

Publish the approved image to Amazon ECR

The important principle is:

Build success is not security approval.

The security pipeline must independently evaluate the risks before the artifact moves forward.

# 3. Security Responsibilities and Ownership

Security scanning is not only a DevOps responsibility. Multiple teams participate.

| Team                 | Responsibility                                          |
| -------------------- | ------------------------------------------------------- |
| Developer            | Fix insecure code, dependencies and exposed secrets     |
| DevOps               | Implement and maintain CI security stages               |
| Platform Engineering | Provide reusable security workflows and standards       |
| Security Team        | Define policies, thresholds and exceptions              |
| QA                   | Validate functional behavior after security fixes       |
| SRE / Operations     | Monitor production risks and security-related incidents |
| Change Manager       | Approve production changes according to company policy  |

Example:

A developer introduces an outdated Java dependency.

Trivy identifies a vulnerability in that dependency.

The developer is responsible for updating the dependency and testing the application.

The DevOps team maintains the pipeline that detects the issue.

The Security team determines the acceptable severity threshold and exception process.

# 4. Repository Structure

Our existing repository structure is extended with security configuration.

```
company-ecommerce/
│
├── order-service/
│   │
│   ├── src/
│   │   ├── main/
│   │   │   └── java/
│   │   └── test/
│   │
│   ├── pom.xml
│   ├── mvnw
│   ├── Dockerfile
│   │
│   ├── helm/
│   │
│   ├── security/
│   │   ├── trivy.yaml
│   │   ├── trivyignore
│   │   └── security-policy.md
│   │
│   ├── .github/
│   │   └── workflows/
│   │       └── security-scan.yml
│   │
│   └── README.md
│
├── platform-golden-path/
│   ├── workflows/
│   │   ├── reusable-ci.yml
│   │   └── reusable-security-scan.yml
│   │
│   ├── policies/
│   │   ├── security/
│   │   │   ├── vulnerability-policy.md
│   │   │   ├── secret-policy.md
│   │   │   └── exception-policy.md
│   │
│   └── templates/
│
├── infrastructure/
│   └── terraform/
│
└── gitops/
    ├── apps/
    └── environments/
```

## 4.1 Why separate security configuration?

Imagine we have 21 microservices.

Without standardization, each service team may implement security checks differently.

For example:

- `order-service` blocks critical vulnerabilities.
- `payment-service` blocks critical and high vulnerabilities.
- `product-service` does not run dependency scanning.
- `cart-service` ignores secret scanning failures.

This creates inconsistent security.

Instead, Platform Engineering provides a common security workflow and common policy.

Individual microservices consume that reusable workflow.

The Security team owns the policy definitions and approves exceptions.

# 5. Understand the Different Security Scanning Types

Before implementing the tools, understand what each scanning category checks.

| Security category    | What it checks               | Example                     |
| -------------------- | ---------------------------- | --------------------------- |
| SAST                 | Application source code      | SQL injection patterns      |
| SCA                  | Third-party dependencies     | Vulnerable Maven library    |
| Secret scanning      | Exposed credentials          | AWS access key in Git       |
| IaC scanning         | Infrastructure configuration | Publicly accessible storage |
| Container scanning   | Container image packages     | Vulnerable OS package       |
| DAST                 | Running application behavior | Runtime web vulnerabilities |
| API security testing | API-specific risks           | Broken authentication       |

Our current pipeline focuses on:

- SonarQube for code quality and source-code security analysis.
- Trivy for filesystem, dependency and configuration scanning.
- Secret detection.
- Veracode for application security analysis.

Container scanning is part of the complete security lifecycle, but the actual image scan is implemented once the image exists in the later build and registry stages.

DAST and penetration testing are separate security activities. Passing this pipeline does not mean an application has passed every possible security assessment.

# 6. SonarQube — Source Code Security Analysis

## 6.1 What is SonarQube?

SonarQube analyzes source code and identifies issues such as:

- Bugs.
- Code smells.
- Security vulnerabilities.
- Security hotspots.
- Maintainability issues.
- Duplicated code.
- Test coverage information when coverage reports are supplied.

Example:

A developer writes unsafe SQL query construction:

```
String query = "SELECT * FROM orders WHERE id = " + orderId;
```

This can create a SQL injection risk if `orderId` is influenced by untrusted input.

SonarQube can identify certain vulnerable coding patterns depending on the language, rules and analysis configuration.

A safer implementation uses parameterized queries:

```
String query = "SELECT * FROM orders WHERE id = ?";
```

The actual implementation should use the appropriate database API, such as Spring Data JPA or prepared statements.

## 6.2 How SonarQube works

The execution flow:

1. Developer creates a pull request.
2. GitHub Actions checks out the repository.
3. Maven compiles and tests the Java application.
4. JaCoCo generates the test coverage report.
5. SonarScanner analyzes the source code.
6. SonarQube receives the analysis report.
7. SonarQube evaluates the configured quality gate.
8. CI checks the result.
9. The pipeline passes or fails according to the organization's policy.

SonarQube execution flow

Developer PR

GitHub Actions runner

SonarScanner

SonarQube server

Quality Gate

Pass

Fail

## 6.3 Where do we install SonarQube in production?

SonarQube is generally operated as a centrally managed security/code-quality service.

For our example, the organization may run it on a dedicated VM or an appropriately sized Kubernetes deployment.

Example architecture:

```
Developer
    |
    v
GitHub Actions
    |
    v
SonarScanner
    |
    v
SonarQube Server
    |
    v
SonarQube Database
```

Production considerations:

- Keep SonarQube accessible only through approved internal network paths.
- Use HTTPS.
- Configure authentication.
- Store tokens in a secrets manager or CI secret store.
- Back up the SonarQube database and required configuration.
- Monitor server health and storage.
- Manage plugin and version upgrades.
- Restrict administrative access.
- Define project-level permissions.

The SonarQube database is not an application database. It stores SonarQube's own operational and analysis data.

Do not expose the SonarQube administration interface publicly without an approved security design.

## 6.4 Configure the SonarQube project

In SonarQube:

1. Log in with an authorized account.
2. Create a project.
3. Select the appropriate project key.
4. Generate an authentication token.
5. Configure the project analysis settings.
6. Configure a quality gate.
7. Add the required token to GitHub Actions secrets.

Example project key:

```
order-service
```

Example project name:

```
Enterprise Ecommerce - Order Service
```

The project key must match the key used in the pipeline.

## 6.5 Configure GitHub Actions secrets

Navigate to:

```
GitHub Repository
    |
    v
Settings
    |
    v
Secrets and variables
    |
    v
Actions
```

Create:

```
SONAR_HOST_URL
SONAR_TOKEN
```

Example:

```
SONAR_HOST_URL=https://sonarqube.company.internal
SONAR_TOKEN=<secure-token>
```

Never commit a real token into Git.

Do not put tokens in:

- `pom.xml`.
- Dockerfile.
- Application source code.
- Git repository configuration.
- Public documentation.
- Workflow logs.

## 6.6 Maven configuration

In `pom.xml`, configure the Sonar Maven plugin.

Example:

```
<properties>
    <sonar.projectKey>order-service</sonar.projectKey>
    <sonar.projectName>Order Service</sonar.projectName>

    <sonar.java.coveragePlugin>jacoco</sonar.java.coveragePlugin>

    <sonar.coverage.jacoco.xmlReportPaths>
        ${project.basedir}/target/site/jacoco/jacoco.xml
    </sonar.coverage.jacoco.xmlReportPaths>
</properties>

<build>
    <plugins>

        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.13</version>

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

The version above is an illustrative pinned version; verify compatibility with the Java and Maven versions used by the project before adopting it.

The JaCoCo plugin generates test coverage information.

SonarQube can import that report to display coverage metrics.

Important:

Coverage is not the same as security.

A codebase can have high test coverage and still contain security vulnerabilities.

## 6.7 Run SonarQube analysis locally

First, build and test:

```
./mvnw clean verify
```

Check the JaCoCo report:

```
ls -l target/site/jacoco/
```

Expected file:

```
target/site/jacoco/jacoco.xml
```

Run the Sonar analysis:

```
./mvnw sonar:sonar \
  -Dsonar.host.url="$SONAR_HOST_URL" \
  -Dsonar.token="$SONAR_TOKEN"
```

Expected behavior:

```
[INFO] Analyzing on SonarQube server
[INFO] Analysis report generated
[INFO] Analysis report uploaded
[INFO] ANALYSIS SUCCESSFUL
```

These are illustrative output messages, not the result of an actual run.

## 6.8 Quality Gate configuration

A quality gate defines the conditions that code must satisfy.

Example organizational policy:

| Condition                    | Example policy               |
| ---------------------------- | ---------------------------- |
| New critical vulnerabilities | Zero                         |
| New blocker bugs             | Zero                         |
| New security hotspots        | Must be reviewed             |
| New-code coverage            | Organization-defined minimum |
| New-code duplication         | Organization-defined maximum |

These values are examples. The Security and Engineering teams must agree on actual thresholds.

The important idea is that a quality gate should focus on the organization's risk requirements.

For a mature application, teams often apply a new-code quality gate so that existing technical debt does not prevent every future release.

However, existing vulnerabilities must still have a remediation plan.

## 6.9 Quality Gate failure example

Suppose the developer introduces insecure SQL construction.

SonarQube reports a security issue.

The quality gate fails.

The developer must:

1. Open the SonarQube project.
2. Review the issue.
3. Understand the affected file and line.
4. Fix the insecure code.
5. Add or update tests.
6. Commit the fix.
7. Push the changes.
8. Rerun CI.

Do not simply disable the quality gate to make the pipeline green.

# 7. Trivy — Vulnerability and Configuration Scanning

## 7.1 What is Trivy?

Trivy is a security scanner used to identify several categories of risks.

It can scan:

- Filesystem dependencies.
- Git repositories.
- Container images.
- Infrastructure-as-Code configuration.
- Kubernetes configuration.
- Software bills of materials (SBOMs).
- Secrets, depending on scan configuration and enabled scanners.

In our current stage, we focus on source repository, dependency, configuration and secret checks.

## 7.2 Why do we need Trivy if we already use SonarQube?

They serve different purposes.

| SonarQube                      | Trivy                                          |
| ------------------------------ | ---------------------------------------------- |
| Source code quality            | Vulnerability and configuration scanning       |
| Bugs and code smells           | Vulnerable dependencies                        |
| Source-level security analysis | Package and filesystem vulnerability detection |
| Quality gates                  | Scanner exit-code and policy controls          |
| Coverage integration           | SBOM generation and package analysis           |

Using both gives broader coverage.

Neither tool replaces the complete security testing process.

## 7.3 Install Trivy on an Ubuntu runner

For a self-hosted Ubuntu runner, use the vendor's supported installation method and pin the approved version.

One installation approach is the official APT repository.

Example:

```
sudo apt-get update

sudo apt-get install -y \
  wget \
  gnupg \
  lsb-release
```

Add the official repository key and repository using the current installation instructions published by Aqua Security.

Then:

```
sudo apt-get update

sudo apt-get install -y trivy
```

Verify:

```
trivy --version
```

Example output:

```
Version: 0.x.x
```

The exact version depends on the approved installation package.

For GitHub-hosted runners, use a pinned Trivy setup action or install a pinned CLI release. Do not depend on whatever happens to be the newest version during every pipeline execution.

Production recommendation: Pin the scanner version and update it through a controlled maintenance process.

## 7.4 Trivy scan types

Understand the different commands.

| Command        | Purpose                    |
| -------------- | -------------------------- |
| `trivy fs`     | Scan a local filesystem    |
| `trivy repo`   | Scan a repository          |
| `trivy image`  | Scan a container image     |
| `trivy config` | Scan IaC and configuration |
| `trivy k8s`    | Scan Kubernetes resources  |
| `trivy sbom`   | Analyze an SBOM            |

For our source pipeline, we will use filesystem and configuration scanning.

## 7.5 Filesystem vulnerability scanning

Move to the service directory:

```
cd order-service
```

Run:

```
trivy fs \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  .
```

What happens?

1. Trivy discovers supported package manifests and lock files.
2. It identifies dependency versions.
3. It checks vulnerability information.
4. It evaluates discovered vulnerabilities against the selected severity filter.
5. It displays the results.

Example output:

```
order-service/pom.xml

Library                  Installed Version   Severity
------------------------------------------------------
example-library          1.2.0               HIGH
another-library          2.0.1               CRITICAL
```

These are illustrative findings.

A real result depends on the actual dependency tree and vulnerability database.

## 7.6 Fail the pipeline for selected vulnerabilities

A scan that only prints findings does not automatically block a pipeline.

We must configure the exit-code behavior.

Example:

```
trivy fs \
  --scanners vuln \
  --severity CRITICAL \
  --exit-code 1 \
  .
```

Explanation:

- `fs`: scan the current filesystem.
- `--scanners vuln`: perform vulnerability scanning.
- `--severity CRITICAL`: filter for critical severity.
- `--exit-code 1`: return a non-zero exit code if matching findings exist.
- `.`: scan the current directory.

If a matching vulnerability is found, the command returns a failure status.

For a policy that blocks both high and critical findings:

```
trivy fs \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  .
```

Whether HIGH should block every service is an organizational decision. A payment service might have stricter requirements than an internal non-sensitive service.

Also remember: severity is not the complete measure of exploitability. Review the vulnerability details, affected package, fix availability and actual application exposure.

## 7.7 Generate a Trivy JSON report

For audit and troubleshooting, save the scan result.

```
mkdir -p security-reports
```

Run:

```
trivy fs \
  --scanners vuln \
  --format json \
  --output security-reports/trivy-fs-report.json \
  .
```

The JSON report can be:

- Uploaded as a CI artifact.
- Processed by security reporting tools.
- Retained according to audit policy.
- Used for vulnerability trend analysis.

Do not upload reports containing sensitive source or secret values to public locations.

## 7.8 Generate a SARIF report

SARIF is a standard format used by code-scanning systems.

Example:

```
trivy fs \
  --scanners vuln \
  --format sarif \
  --output security-reports/trivy-fs.sarif \
  .
```

SARIF can be uploaded to supported code-scanning interfaces.

Be careful with permissions and visibility: security findings should not become publicly accessible unintentionally.

# 8. Dependency Scanning — Maven Dependencies

Our Order Service is a Java Spring Boot application.

It uses Maven dependencies.

Example:

```
<dependencies>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>

</dependencies>
```

The dependencies themselves may contain security vulnerabilities.

For example:

```
order-service
    |
    ├── spring-boot-starter-web
    │       |
    │       ├── spring-web
    │       └── spring-core
    │
    ├── spring-boot-starter-data-jpa
    │       |
    │       ├── spring-data-jpa
    │       └── hibernate
    │
    └── postgresql
```

A vulnerability may exist in a transitive dependency rather than a dependency directly declared by the developer.

## 8.1 Understand direct and transitive dependencies

Direct dependency: A library explicitly declared in `pom.xml`.

Transitive dependency: A library pulled in automatically by another dependency.

Example:

```
Your application
    |
    v
spring-boot-starter-web
    |
    v
spring-web
    |
    v
another-library
```

Even if you never explicitly added `another-library`, it can still become part of your application.

That is why scanning only the top-level dependencies is insufficient.

## 8.2 View the dependency tree

Run:

```
./mvnw dependency:tree
```

To search for a specific dependency:

```
./mvnw dependency:tree \
  -Dincludes=org.springframework
```

To produce a file:

```
./mvnw dependency:tree \
  -DoutputFile=dependency-tree.txt
```

This helps identify where a vulnerable transitive dependency originates.

## 8.3 Remediate vulnerable dependencies

Suppose Trivy reports a vulnerability in a dependency.

The remediation workflow is:

1. Identify the affected library.
2. Identify whether it is direct or transitive.
3. Review the vulnerability advisory.
4. Check the fixed version.
5. Update the direct dependency or its parent dependency.
6. Rebuild the application.
7. Run unit and integration tests.
8. Rerun security scans.
9. Review compatibility.
10. Commit the fix.

Example:

```
<properties>
    <example-library.version>2.4.0</example-library.version>
</properties>
```

The version shown is only illustrative. Use the actual fixed version recommended for the affected library and your dependency compatibility requirements.

Do not blindly upgrade every library to the newest version in production.

A dependency update can introduce:

- Breaking API changes.
- Java compatibility problems.
- Runtime failures.
- Configuration changes.
- Performance regressions.

Always validate the updated dependency.

# 9. Trivy Configuration Scanning

Source repositories contain more than Java code.

They also contain infrastructure and deployment configuration.

Examples:

```
order-service/
├── Dockerfile
├── helm/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
└── security/
```

A configuration file can introduce security problems even if the application code is secure.

Examples:

- Running containers as root.
- Privileged containers.
- Unrestricted ingress.
- Missing resource limits.
- Publicly exposed services.
- Missing security contexts.
- Weak container filesystem settings.

## 9.1 Scan configuration files

Run:

```
trivy config \
  --severity HIGH,CRITICAL \
  .
```

Trivy checks supported configuration formats against security rules.

Example findings:

```
helm/templates/deployment.yaml

Finding:
Container is configured to run as root.

Severity:
HIGH
```

This is an illustrative finding.

## 9.2 Example insecure Kubernetes Deployment

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: order-service

spec:
  replicas: 2

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
          image: order-service:example

          securityContext:
            privileged: true
            runAsUser: 0
```

Problems:

- Privileged execution increases container capabilities.
- Running as root increases the impact of a container compromise.

## 9.3 Improved secure configuration

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: order-service

spec:
  replicas: 2

  selector:
    matchLabels:
      app: order-service

  template:
    metadata:
      labels:
        app: order-service

    spec:
      automountServiceAccountToken: false

      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: order-service

          image: order-service:example

          securityContext:
            allowPrivilegeEscalation: false
            privileged: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL

          resources:
            requests:
              cpu: 250m
              memory: 512Mi

            limits:
              cpu: 1000m
              memory: 1Gi
```

This is an illustrative secure baseline.

A read-only root filesystem requires the application to use writable mounted volumes for locations that genuinely need writes, such as temporary files.

Also, `automountServiceAccountToken: false` is appropriate only if the application does not need Kubernetes API access through its service account. If it needs AWS access through EKS Pod Identity or IRSA, design and test the identity configuration appropriately.

## 9.4 Scan the Helm directory

Run:

```
trivy config \
  --severity HIGH,CRITICAL \
  helm/
```

Review every reported finding.

Do not assume every finding is automatically exploitable.

For example, a Helm template may contain a configurable value that is overridden safely in the production values file.

The correct approach is to inspect the rendered manifests for each environment.

Example:

```
helm template order-service ./helm \
  -f ./helm/values.yaml \
  -f ./helm/values-staging.yaml \
  > rendered-staging.yaml
```

Then:

```
trivy config rendered-staging.yaml
```

Repeat for the other environment values.

# 10. Secret Scanning

## 10.1 What is a secret?

A secret is sensitive information used to authenticate or authorize access.

Examples:

- AWS access keys.
- Database passwords.
- API tokens.
- GitHub personal access tokens.
- Private SSH keys.
- TLS private keys.
- Cloud service credentials.

A developer may accidentally commit a secret.

Example:

```
String awsAccessKey = "AKIAEXAMPLE123456";
String databasePassword = "MyPassword123";
```

This is dangerous because anyone with repository access may be able to obtain the credential.

If the repository is public, the exposure can be even more serious.

## 10.2 Why `.gitignore` is not sufficient

Consider:

1. Developer creates `application.properties`.
2. Developer adds a database password.
3. Developer commits the file.
4. Developer later adds it to `.gitignore`.

The file may still exist in Git history.

Adding a file to `.gitignore` does not remove its previous commits.

A leaked secret must be treated as compromised.

## 10.3 Scan secrets using Trivy

Run:

```
trivy fs \
  --scanners secret \
  .
```

This checks supported files for patterns that may represent secrets.

For example, Trivy can identify certain token and credential patterns.

The result depends on scanner coverage and configuration.

No secret scanner can guarantee detection of every possible credential.

## 10.4 Combine vulnerability and secret scanning

Example:

```
trivy fs \
  --scanners vuln,secret \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  .
```

This combines vulnerability and secret scanning.

For production, configure and test each scanner explicitly rather than assuming a severity option applies identically to every finding type.

## 10.5 What to do if a secret is exposed?

Suppose a developer commits AWS credentials.

Do not simply delete the line and push another commit.

The correct response is:

1. Identify the exposed credential.
2. Immediately revoke or rotate it.
3. Review access logs and usage.
4. Check whether unauthorized activity occurred.
5. Remove the secret from the current source.
6. Review Git history and repository exposure.
7. Remove historical copies where appropriate.
8. Add a secret scanning control.
9. Document the incident.

If an AWS access key is exposed, disable or delete the compromised access key and issue a replacement only if the workload genuinely requires it.

Prefer IAM roles with short-lived credentials instead of long-lived access keys.

For GitHub Actions accessing AWS, prefer OIDC federation with a narrowly scoped IAM role.

## 10.6 Use environment variables and secret managers

Do not store passwords directly in source code.

For example, Spring Boot can read environment variables:

```
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

In Kubernetes, the application can receive credentials from a supported secret integration.

For AWS environments, possible approaches include:

- AWS Secrets Manager.
- AWS Systems Manager Parameter Store.
- Kubernetes Secrets backed by an approved external secret integration.

Kubernetes Secrets are not automatically encrypted end-to-end merely because they are stored as Secret objects. Configure encryption at rest, access restrictions, and secure delivery.

# 11. Veracode — Application Security Testing

## 11.1 What is Veracode?

Veracode is an application security testing platform.

It can be used as part of an organization's application security program.

Depending on the licensed product and configured integration, it supports application analysis such as:

- Static analysis.
- Software composition analysis.
- Dynamic analysis.
- Policy evaluation.

The exact coverage depends on the Veracode products enabled by the organization.

## 11.2 Why use Veracode with SonarQube and Trivy?

These tools are not interchangeable.

| Tool      | Main use in our pipeline                                     |
| --------- | ------------------------------------------------------------ |
| SonarQube | Code quality and source-code security analysis               |
| Trivy     | Filesystem, dependency, configuration and container scanning |
| Veracode  | Application security testing and policy assessment           |

An organization may use multiple tools because it needs different levels of analysis, compliance evidence, security workflows and reporting.

Avoid running redundant scans without understanding the results and ownership.

## 11.3 Veracode integration architecture

```
Developer Pull Request
        |
        v
GitHub Actions
        |
        v
Build Application
        |
        v
Prepare Security Package
        |
        v
Veracode Integration
        |
        v
Veracode Platform
        |
        v
Security Results
        |
        v
Policy Evaluation
        |
        v
Pipeline Decision
```

Veracode is typically consumed as an external service.

We do not need to install Veracode as an application inside our EKS cluster just to use its CI integration.

## 11.4 Credentials

The security team or authorized administrator provisions the required API credentials.

Example GitHub Actions secrets:

```
VERACODE_API_ID
VERACODE_API_KEY
```

Store these using the organization's approved secrets management process.

Never print credentials in logs.

Do not expose these credentials to untrusted pull-request code.

## 11.5 Prepare an application package

For a Java application, the build artifact may be a JAR.

Example:

```
./mvnw clean package
```

Check:

```
ls -lh target/*.jar
```

Example:

```
target/order-service.jar
```

The exact packaging and upload format depend on the selected Veracode scanning product.

For example, a pipeline may use the supported Veracode CLI or an approved GitHub Actions integration.

## 11.6 Example integration pattern

The following illustrates the structure of a Veracode integration stage. It is not a complete, vendor-version-specific workflow; the exact action inputs and package requirements must be taken from the organization's approved Veracode integration documentation.

```
- name: Prepare Veracode package
  run: |
    ./mvnw clean package

    mkdir -p security-reports

    cp target/*.jar security-reports/

- name: Run Veracode analysis
  env:
    VERACODE_API_ID: ${{ secrets.VERACODE_API_ID }}
    VERACODE_API_KEY: ${{ secrets.VERACODE_API_KEY }}
  run: |
    echo "Starting Veracode security analysis"

    # Invoke the organization's approved
    # Veracode CLI or integration here.

    echo "Veracode submission stage completed"
```

Important: This example prepares the package and shows the credential boundary. The `echo` statements do not perform a Veracode scan. A production implementation must invoke the actual approved CLI or integration, wait for the required analysis result, retrieve the policy status, and fail or pass CI based on that result.

## 11.7 Veracode policy evaluation

A scan may finish successfully while the application still violates a security policy.

These are different conditions:

- Scanner execution completed.
- Security analysis completed.
- Security policy passed.

A production pipeline must evaluate the actual policy result.

For example:

```
Veracode submission: Completed
Analysis: Completed
Policy evaluation: Failed
Pipeline: Failed
```

Do not treat successful file upload as successful security approval.

# 12. Docker Image Scanning — Where It Fits

Container image scanning is essential, but the complete image does not exist yet in Chapter 196.

We will implement the Docker build in Chapter 197 and ECR publication in Chapter 198.

For now, understand the lifecycle.

```
Source Code
    |
    v
Dependency Scan
    |
    v
Dockerfile Review
    |
    v
Build Docker Image
    |
    v
Image Vulnerability Scan
    |
    v
Security Approval
    |
    v
Publish Image to ECR
```

Why scan the image separately?

Because an image contains more than the application JAR.

It may include:

- Base operating system packages.
- Runtime packages.
- System libraries.
- Application dependencies.
- Additional binaries.
- Configuration files.

A clean Maven dependency scan does not guarantee a clean container image.

## 12.1 Example Dockerfile security considerations

A Dockerfile should use an approved, pinned base image.

Example:

```
FROM eclipse-temurin:21-jre

WORKDIR /app

COPY target/order-service.jar app.jar

USER 10001

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

This is a simplified example, not yet the final production Dockerfile.

Important improvements to consider:

- Pin the base image to a digest.
- Use a minimal trusted runtime image.
- Avoid running as root.
- Avoid including build tools in the runtime image.
- Avoid copying credentials into the image.
- Keep the image small.
- Use a read-only root filesystem where supported.
- Scan the resulting image before publishing.

We will implement the complete build and scanning flow in the appropriate later chapters without rebuilding the image between environments.

# 13. Security Policies — What Should Block CI?

A scanning tool identifies issues. A security policy determines what happens next.

We need to distinguish three outcomes.

BLOCK

Pipeline failure

Critical security finding, exposed credential, failed mandatory quality gate, or a policy violation without an approved exception.

WARN

Pipeline continues with a warning

A finding that the policy permits temporarily, but which requires tracking and remediation.

PASS

Security checks satisfied

Required scans completed and no blocking policy violations remain.

## 13.1 Example vulnerability policy

This is a sample policy. The Security team must approve the actual thresholds.

| Finding                                    | Example action                         |
| ------------------------------------------ | -------------------------------------- |
| Critical vulnerability with applicable fix | Block                                  |
| High vulnerability                         | Block or require risk-based approval   |
| Medium vulnerability                       | Track and remediate within SLA         |
| Low vulnerability                          | Track according to policy              |
| Confirmed exposed secret                   | Block and initiate credential response |
| False positive                             | Review and approve exception           |
| Sonar quality gate failure                 | Block                                  |
| Veracode policy failure                    | Block if required by release policy    |

The exact blocking rules must be consistent across the 21 services.

## 13.2 Example policy document

Create:

`platform-golden-path/policies/security/vulnerability-policy.md`

```
# Vulnerability Policy

## Purpose

Define minimum vulnerability handling requirements
for enterprise e-commerce microservices.

## Scope

- Java dependencies
- Application source code
- Container images
- Kubernetes configuration
- Infrastructure configuration

## Severity handling

### Critical
- Must be reviewed immediately.
- Block promotion unless an approved exception exists.
- Remediate according to the security incident process.

### High
- Must be reviewed by the Security team.
- Block according to service risk and approved policy.
- Require documented remediation or exception.

### Medium
- Create a remediation task.
- Track against the organization's remediation SLA.

### Low
- Track and resolve according to the normal maintenance process.

## Secret exposure

Confirmed credential exposure is a blocking event.

Credentials must be revoked or rotated.

## Exceptions

Exceptions require:

- Finding identifier
- Affected component
- Business justification
- Risk assessment
- Compensating controls
- Owner
- Approval
- Expiry date

## Reassessment

Exceptions must be reviewed before expiry.
```

## 13.3 Security policy must consider exploitability

Severity alone is not enough.

For each vulnerability, consider:

- Is the affected package actually included in the application?
- Is the vulnerable function reachable?
- Is the vulnerable component exposed to untrusted input?
- Is there a known exploit?
- Is a fixed version available?
- Is the application internet-facing?
- Does the service process sensitive information?
- Are compensating controls present?

For example, a vulnerability in a package used only in a development tool may have a different production impact from one in an internet-facing request-processing library.

The Security team must document its risk assessment.

# 14. Blocking Versus Non-Blocking Security Scans

A common production mistake is making every security scan blocking immediately.

This can create pipeline instability, particularly in large applications with existing technical debt.

A better approach is to define a deliberate enforcement model.

## 14.1 Blocking scans

Typical blocking checks:

- Confirmed exposed secrets.
- Critical vulnerabilities that meet the organization's blocking criteria.
- Mandatory SonarQube quality gate failures.
- Required Veracode policy failures.
- High-risk configuration findings covered by mandatory policies.

## 14.2 Non-blocking scans

Some scans may initially run in reporting mode while an organization establishes a vulnerability baseline.

Example:

```
trivy fs \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  .
```

This produces results but does not explicitly configure the scanner to return a non-zero exit code for findings.

However, a production pipeline must still check the command's execution status. A scanner crash, invalid configuration or failed vulnerability database download should not be silently interpreted as a clean security result.

Use separate handling for:

- Findings.
- Scanner execution errors.
- Policy violations.

## 14.3 Migration from reporting to enforcement

Suppose we have 21 microservices and many existing vulnerabilities.

A practical rollout could be:

Stage 1 — Visibility

- Run all scans.
- Collect findings.
- Establish a baseline.
- Identify application owners.

Stage 2 — New-code enforcement

- Block newly introduced critical issues.
- Enforce secret scanning.
- Enforce mandatory code quality gates.

Stage 3 — Risk-based enforcement

- Add high-severity blocking policies.
- Define service-specific risk controls.
- Introduce remediation SLAs.

Stage 4 — Full governance

- Enforce approved security policies.
- Track exceptions and expiry.
- Report service-level vulnerability trends.

This allows the organization to improve security without hiding the existing backlog.

# 15. False Positives and Security Exceptions

## 15.1 What is a false positive?

A false positive occurs when a scanner reports an issue that is not actually a vulnerability in the specific context.

Example:

Trivy reports a vulnerable dependency.

After investigation, the team determines:

- The affected code path is not included.
- The dependency is not used at runtime.
- The package version information is incorrect.
- The vulnerability does not apply to the deployed platform.

The finding may qualify for a documented exception, depending on the evidence.

Do not assume every unexpected finding is a false positive.

## 15.2 Exception management process

```
Security Finding
       |
       v
Developer Review
       |
       v
Security Team Assessment
       |
       v
Is Finding Valid?
       |
       +-------------------+
       |                   |
      Yes                  No
       |                   |
       v                   v
Remediation         Document False Positive
       |                   |
       v                   v
Rescan              Approved Exception
```

## 15.3 Trivy ignore file

Trivy supports an ignore file for approved findings.

Create:

`order-service/security/trivyignore`

Example:

```
# Illustrative vulnerability identifier.
# Use only after security approval.

CVE-2025-00000
```

This is a placeholder identifier, not a real approval.

The scan can use:

```
trivy fs \
  --ignorefile security/trivyignore \
  --scanners vuln \
  .
```

A production exception must have supporting evidence outside the ignore line, including:

- Ticket or exception ID.
- Finding ID.
- Owner.
- Justification.
- Security approval.
- Expiration date.
- Remediation plan.

Do not create broad, permanent ignore rules.

Do not suppress a finding simply because it makes the pipeline fail.

# 16. Secure GitHub Actions Credentials

Security tooling itself needs credentials.

Examples:

- SonarQube token.
- Veracode API credentials.
- Private package registry credentials.
- AWS role credentials.
- GitHub App token.

These credentials must be protected.

## 16.1 Avoid hardcoding credentials

Bad:

```
env:
  SONAR_TOKEN: "actual-secret-token"
```

Good:

```
env:
  SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

## 16.2 Restrict permissions

Use the minimum permissions required by the workflow.

Example:

```
permissions:
  contents: read
```

If the workflow needs to publish a code-scanning result, add only the required permission for that operation.

Do not give every workflow:

```
permissions: write-all
```

## 16.3 Protect untrusted pull requests

This is extremely important.

Suppose an external contributor submits a pull request containing malicious code.

If the workflow exposes repository secrets to that untrusted code, the code could attempt to steal them.

Avoid exposing privileged credentials to untrusted pull-request workflows.

Recommended controls:

- Use read-only permissions for untrusted PR jobs.
- Do not expose AWS credentials unnecessarily.
- Do not expose SonarQube or Veracode secrets to untrusted code.
- Use separate trusted workflows for privileged operations.
- Protect branches and environments.
- Require approval for privileged workflows.
- Pin third-party actions to verified commit SHAs.

For example, GitHub Actions workflows using `pull_request_target` require special care because they can run with access to the base repository's permissions and secrets. Never check out and execute untrusted PR code in a privileged context without a carefully reviewed security design.

## 16.4 Use AWS OIDC instead of static access keys

When a workflow needs AWS access, prefer OIDC federation.

The flow:

```
GitHub Actions
      |
      v
GitHub OIDC Token
      |
      v
AWS IAM OIDC Provider
      |
      v
IAM Role Trust Policy
      |
      v
Temporary AWS Credentials
```

The CI workflow should assume only the role required for its task.

In this chapter, we do not need production deployment credentials.

The security pipeline should not have permission to deploy to production.

Production deployment is controlled through the later GitOps and approval process.

# 17. Complete GitHub Actions Security Workflow

Now we combine the source-level security stages into one practical workflow.

File:

`order-service/.github/workflows/security-scan.yml`

This example assumes:

- The repository uses Maven Wrapper.
- SonarQube is centrally available.
- The runner has access to the approved security services.
- Trivy is installed through a pinned, approved setup method.
- The organization has approved a Veracode integration.
- Required GitHub secrets are configured.
- The workflow is executed only in a trusted context when privileged credentials are needed.

The example deliberately separates scanner execution from policy enforcement.

```
name: Order Service Security Pipeline

on:
  pull_request:
    branches:
      - main

  push:
    branches:
      - main

permissions:
  contents: read

jobs:

  security-scan:

    name: Source Security Scanning

    runs-on: ubuntu-latest

    steps:

      # ------------------------------------
      # 1. Checkout
      # ------------------------------------

      - name: Checkout source code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # ------------------------------------
      # 2. Java Setup
      # ------------------------------------

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      # ------------------------------------
      # 3. Maven Build
      # ------------------------------------

      - name: Build application
        run: |
          chmod +x mvnw
          ./mvnw clean verify

      # ------------------------------------
      # 4. Check JaCoCo Report
      # ------------------------------------

      - name: Verify coverage report
        run: |
          test -f target/site/jacoco/jacoco.xml

      # ------------------------------------
      # 5. SonarQube Analysis
      # ------------------------------------

      - name: SonarQube analysis
        env:
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          ./mvnw sonar:sonar \
            -Dsonar.host.url="$SONAR_HOST_URL" \
            -Dsonar.token="$SONAR_TOKEN"

      # ------------------------------------
      # 6. Install Trivy
      # ------------------------------------

      - name: Install Trivy
        run: |
          echo "Install the organization-approved pinned Trivy version"
          # Replace this step with the approved pinned installer.

      # ------------------------------------
      # 7. Prepare Reports
      # ------------------------------------

      - name: Create report directory
        run: |
          mkdir -p security-reports

      # ------------------------------------
      # 8. Dependency Vulnerability Scan
      # ------------------------------------

      - name: Trivy filesystem scan
        run: |
          trivy fs \
            --scanners vuln \
            --format json \
            --output security-reports/trivy-fs.json \
            .

      # ------------------------------------
      # 9. Configuration Scan
      # ------------------------------------

      - name: Trivy configuration scan
        run: |
          trivy config \
            --format json \
            --output security-reports/trivy-config.json \
            helm/

      # ------------------------------------
      # 10. Secret Scan
      # ------------------------------------

      - name: Trivy secret scan
        run: |
          trivy fs \
            --scanners secret \
            --format json \
            --output security-reports/trivy-secrets.json \
            .

      # ------------------------------------
      # 11. Security Policy Evaluation
      # ------------------------------------

      - name: Enforce vulnerability policy
        run: |
          trivy fs \
            --scanners vuln \
            --severity CRITICAL \
            --exit-code 1 \
            .

      # ------------------------------------
      # 12. Upload Reports
      # ------------------------------------

      - name: Upload security reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: order-service-security-reports
          path: security-reports/
          retention-days: 14
```

This is a workflow skeleton, not yet a complete production enforcement implementation. The Trivy installation step must be replaced with a pinned installer. The example enforcement step currently enforces critical filesystem vulnerabilities only; it does not yet enforce every JSON report, configuration finding, secret finding or Veracode policy. The Veracode submission and result polling stage must also be added using the organization's approved integration.

For a fully governed production implementation, each required scanner needs an explicit pass/fail result and a defined policy decision.

Also, GitHub action version tags such as `@v4` are readable examples. Production workflows should pin actions to verified immutable commit SHAs and manage updates through a controlled process.

# 18. Improve the Pipeline — Separate Scan and Policy Jobs

In a large organization, one long security job can become difficult to maintain.

A more scalable design separates execution and enforcement.

Example:

```
security-pipeline
│
├── build-and-test
│
├── sonar-analysis
│
├── dependency-scan
│
├── secret-scan
│
├── configuration-scan
│
├── veracode-analysis
│
└── security-policy
```

The policy job consumes the results of the required scanner jobs.

Example GitHub Actions structure:

```
jobs:

  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Build and test"

  sonar:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Run SonarQube"

  dependency-scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Run dependency scan"

  secret-scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Run secret scan"

  configuration-scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Run configuration scan"

  veracode:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Run approved Veracode integration"

  security-policy:
    needs:
      - sonar
      - dependency-scan
      - secret-scan
      - configuration-scan
      - veracode

    runs-on: ubuntu-latest

    steps:
      - name: Evaluate required security results
        run: |
          echo "Evaluate all required security results"
```

The commands above are placeholders illustrating job dependencies, not functioning scanners.

In a real implementation, scanner jobs must produce machine-readable results and the policy job must evaluate those results. A scanner job that was skipped, failed to start or failed to retrieve its database must not be mistaken for a successful clean scan.

# 19. Reusable Security Workflow for 21 Microservices

Our company has 21 microservices.

We should not independently maintain 21 copies of the same security workflow.

Platform Engineering maintains a reusable workflow.

Repository:

`platform-golden-path`

Example:

```
platform-golden-path/
└── workflows/
    └── reusable-security-scan.yml
```

A reusable workflow should contain the approved scanner setup, execution steps, report handling and policy evaluation.

Example caller:

`order-service/.github/workflows/security-scan.yml`

```
name: Order Service Security

on:
  pull_request:
    branches:
      - main

jobs:

  security:

    uses: company/platform-golden-path/.github/workflows/reusable-security-scan.yml@<APPROVED_REF>

    with:
      service-name: order-service
      java-version: '21'
      scan-configuration: true
      scan-secrets: true
```

`<APPROVED_REF>` is a placeholder. In production, use a protected, immutable reference to the approved reusable workflow revision.

The reusable workflow must explicitly declare and securely receive any required secrets. Avoid passing secrets to workflows that do not need them.

Advantages:

- Consistent security checks.
- Centralized maintenance.
- Easier policy changes.
- Less duplicated YAML.
- Easier onboarding of new microservices.
- Better auditability.

## 19.1 Example service coverage

| Microservice         | Security workflow        |
| -------------------- | ------------------------ |
| user-service         | Shared security workflow |
| product-service      | Shared security workflow |
| cart-service         | Shared security workflow |
| order-service        | Shared security workflow |
| payment-service      | Shared security workflow |
| inventory-service    | Shared security workflow |
| notification-service | Shared security workflow |
| Remaining services   | Shared security workflow |

Every service may have different risk requirements, but the baseline should be consistent.

For example, the payment service may require additional security controls because it processes payment-related information.

# 20. GitLab CI/CD Integration

Some organizations use GitHub Actions, while others use GitLab CI/CD.

The same security stages can be implemented in GitLab.

Example:

`order-service/.gitlab-ci.yml`

```
stages:
  - build
  - sonar
  - dependency-scan
  - secret-scan
  - configuration-scan
  - policy

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"

build:
  stage: build

  image: maven:3.9-eclipse-temurin-21

  script:
    - mvn clean verify

sonar:
  stage: sonar

  image: maven:3.9-eclipse-temurin-21

  script:
    - >
      mvn sonar:sonar
      -Dsonar.host.url="$SONAR_HOST_URL"
      -Dsonar.token="$SONAR_TOKEN"

dependency_scan:
  stage: dependency-scan

  image: aquasec/trivy:<APPROVED_VERSION>

  script:
    - trivy fs --scanners vuln --format json --output trivy-fs.json .

  artifacts:
    when: always
    paths:
      - trivy-fs.json

secret_scan:
  stage: secret-scan

  image: aquasec/trivy:<APPROVED_VERSION>

  script:
    - trivy fs --scanners secret .

configuration_scan:
  stage: configuration-scan

  image: aquasec/trivy:<APPROVED_VERSION>

  script:
    - trivy config helm/

security_policy:
  stage: policy

  image: aquasec/trivy:<APPROVED_VERSION>

  script:
    - trivy fs --scanners vuln --severity CRITICAL --exit-code 1 .
```

Replace the placeholder Trivy version with an approved version and use a trusted, pinned container image reference.

Configure SonarQube and other required credentials using protected GitLab CI/CD variables.

For Veracode, add a separate job that invokes the organization's approved integration and checks its actual policy result.

This example demonstrates stage placement; it is not a complete production security policy implementation.

# 21. Security Scanning in a Production Environment

A production-grade pipeline must also manage scanner availability and security data.

## 21.1 Scanner availability

Suppose the Trivy vulnerability database cannot be downloaded.

Should the pipeline pass?

No.

The scanner did not complete successfully, so we cannot claim that the code passed vulnerability scanning.

The pipeline should report a scanner execution failure.

For some environments, security teams may approve a temporary, documented fallback during a service outage. Such a fallback must not silently convert an incomplete scan into a successful security approval.

## 21.2 Vulnerability database updates

Trivy uses vulnerability information to identify affected packages.

Its results depend on the database version and available advisory information.

Production controls should include:

- Approved database update procedures.
- Database availability monitoring.
- Proxy configuration if required.
- Controlled scanner version upgrades.
- Reproducible scan metadata.
- Clear handling of database download failures.

## 21.3 Network access

Security runners may require access to:

- SonarQube.
- Veracode.
- Approved vulnerability database endpoints.
- Private Maven repositories.
- GitHub.
- Internal artifact repositories.

Use restricted outbound access.

Do not allow unrestricted access just because a scanner needs internet connectivity.

## 21.4 Runner isolation

For sensitive projects:

- Use isolated runners.
- Avoid sharing privileged runners with untrusted workloads.
- Clean workspaces after execution.
- Restrict runner network permissions.
- Avoid persisting secrets in the workspace.
- Limit access to build artifacts.
- Rotate runner credentials.
- Keep runner images patched.

## 21.5 Report retention

Security reports can contain sensitive information.

The company should define:

- Which reports are retained.
- Who can access them.
- How long reports are retained.
- Where reports are stored.
- Whether reports are encrypted.
- How reports are linked to the commit and CI run.

A report should include enough information to reproduce the security decision.

Recommended metadata:

```
Service: order-service
Jira: ECOM-4821
Commit: d4e5f6a...
CI Run: <ACTUAL_RUN_ID>
Scanner: Trivy
Scanner Version: <APPROVED_VERSION>
Database Version: <DATABASE_METADATA>
Scan Time: <UTC_TIMESTAMP>
Policy Version: <POLICY_REVISION>
Result: <PASS_OR_FAIL>
```

All angle-bracketed values are placeholders until an actual pipeline execution supplies them.

# 22. SBOM — Software Bill of Materials

An SBOM is an inventory of the software components included in an application or artifact.

For our Order Service, it can describe:

- Application dependencies.
- Package names.
- Package versions.
- Component relationships.
- Licenses and other metadata, depending on format and tool.

Why is this important?

Suppose a vulnerability is announced in a Java library.

The Security team needs to identify which of the 21 microservices contain that library.

Without an SBOM, the team may need to inspect every dependency tree manually.

With an SBOM, the organization has a structured component inventory.

## 22.1 Generate an SBOM with Trivy

Example:

```
trivy fs \
  --format cyclonedx \
  --output security-reports/order-service-sbom.json \
  .
```

This generates a CycloneDX-format SBOM for the scanned filesystem content supported by Trivy.

The organization can retain the SBOM alongside the relevant build metadata.

Important: An SBOM generated from the source filesystem is not automatically identical to the final container image's complete inventory. Generate an image SBOM after the actual image is built.

## 22.2 Why maintain SBOMs?

- Vulnerability impact analysis.
- Dependency inventory.
- Software supply-chain visibility.
- Audit evidence.
- License review.
- Faster response to newly disclosed vulnerabilities.

# 23. Software Supply Chain Security

A secure application also depends on the integrity of the build process.

Potential risks:

- Malicious third-party GitHub Actions.
- Compromised Maven dependencies.
- Untrusted build scripts.
- Stolen CI credentials.
- Tampered build artifacts.
- Unapproved workflow modifications.

Production controls include:

1. Pin GitHub Actions to verified commit SHAs.
2. Review third-party actions before adoption.
3. Restrict workflow permissions.
4. Protect important branches.
5. Use dependency review controls.
6. Use trusted package repositories.
7. Generate SBOMs.
8. Retain build provenance where supported.
9. Protect release credentials.
10. Verify the artifact before promotion.

The security pipeline should establish evidence for the source commit that was scanned.

For our simulation, that commit is:

```
d4e5f6a...
```

The actual SHA must be recorded from the real Git repository.

# 24. Security Pipeline Troubleshooting

This section is important for production DevOps work and interviews.

## 24.1 SonarQube server is unavailable

Symptom:

```
Failed to connect to SonarQube server
```

Possible causes:

- SonarQube is down.
- DNS resolution failure.
- Firewall blocks access.
- Incorrect host URL.
- TLS certificate issue.
- Proxy misconfiguration.

Check connectivity:

```
curl -I "$SONAR_HOST_URL"
```

Check DNS:

```
nslookup sonarqube.company.internal
```

Check TLS:

```
curl -v "$SONAR_HOST_URL"
```

Fix:

- Verify the service health.
- Verify the URL.
- Verify DNS.
- Check firewall rules.
- Validate the certificate chain.
- Check the runner's network route.

Do not disable TLS verification as a permanent workaround.

## 24.2 SonarQube authentication failure

Symptom:

```
Not authorized
```

Possible causes:

- Incorrect token.
- Expired or revoked token.
- Missing project permissions.
- Secret not configured.
- Wrong secret name.

Check:

```
test -n "$SONAR_TOKEN" && echo "Token is configured"
```

Do not print the token.

Fix:

1. Verify the secret name.
2. Confirm the token is active.
3. Check project permissions.
4. Rotate the token if required.
5. Rerun the pipeline.

## 24.3 SonarQube analysis succeeds but the quality gate fails

Symptom:

```
Analysis completed
Quality Gate: FAILED
```

This is not necessarily a connectivity or scanner error.

The analysis was completed, but the configured quality requirements were not satisfied.

Check:

- New vulnerabilities.
- New bugs.
- Coverage.
- Duplicated code.
- Security hotspots.

Fix the underlying issue or follow the formal exception process.

Do not confuse analysis completion with security approval.

## 24.4 Trivy cannot download its vulnerability database

Symptom:

```
Failed to download vulnerability database
```

Check:

```
trivy --version
```

Check network connectivity to the approved database endpoint.

For proxy environments:

```
export HTTPS_PROXY="http://proxy.company.internal:8080"
export HTTP_PROXY="http://proxy.company.internal:8080"
```

Verify the actual proxy requirements with the network team.

Fix:

- Confirm outbound network access.
- Check proxy configuration.
- Check DNS.
- Review firewall rules.
- Confirm the configured database source.
- Retry the scan.

Do not treat a failed database download as a clean vulnerability result.

## 24.5 Trivy reports a vulnerability in a transitive dependency

Symptom:

```
Vulnerability detected in dependency
```

Investigate:

```
./mvnw dependency:tree
```

Identify the parent dependency.

Update the relevant dependency or dependency-management version.

Then:

```
./mvnw clean verify
```

Rerun Trivy.

## 24.6 Trivy finds no vulnerabilities but the scanner failed

A scanner may fail before completing its work.

A missing or incomplete report is not equivalent to a clean scan.

Check:

- Exit status.
- Scanner logs.
- Database update status.
- Report existence.
- Report validity.

A robust pipeline distinguishes:

```
SCAN PASSED
SCAN FAILED
SCAN INCOMPLETE
POLICY FAILED
```

## 24.7 Secret scanning blocks the pipeline

Investigate:

1. Which file triggered the finding?
2. Which line was identified?
3. Is it a genuine credential?
4. Was it committed to Git history?
5. Is the credential still active?
6. Does it require revocation?

If genuine:

- Revoke or rotate it.
- Investigate access.
- Remove it from the repository.
- Review Git history.
- Record the incident.

Do not simply add it to the ignore list.

## 24.8 Veracode submission succeeds but analysis is incomplete

Possible causes:

- The analysis is still processing.
- Incorrect package format.
- Invalid credentials.
- Application profile mismatch.
- Incorrect policy configuration.

Check the actual Veracode submission and analysis status.

Do not treat a successful upload as a completed scan.

## 24.9 Security scan takes too long

Possible causes:

- Repeated dependency downloads.
- Large repository.
- Unnecessary repeated analysis.
- Slow external service.
- Inefficient runner.
- Scanner database download on every run.

Improvements:

- Cache Maven dependencies safely.
- Reuse approved scanner installation.
- Separate independent scan jobs.
- Use appropriate runner resources.
- Avoid duplicate scans.
- Use approved vulnerability database caching.
- Monitor external service latency.

Do not skip mandatory checks just to reduce execution time.

# 25. Real Production Failure Scenario

Now simulate a real problem.

## Scenario: Vulnerable dependency introduced into Order Service

Business requests the Order Status Tracking feature.

Developer updates the Order Service.

The feature compiles successfully.

Unit tests pass.

SonarQube analysis completes.

However, Trivy identifies a critical vulnerability in a transitive Java dependency.

Illustrative finding:

```
Service: order-service

Dependency:
example-library

Installed Version:
1.2.0

Severity:
CRITICAL

Status:
A fixed version is available
```

## 25.1 What happens next?

1. Developer receives pipeline failure

   The PR security check fails because the critical finding matches the blocking policy.
2. Developer investigates the dependency

   Runs:
   ```
   ./mvnw dependency:tree
   ```
   Identifies which dependency introduced the vulnerable library.
3. Developer fixes the dependency

   Updates the direct or parent dependency to a compatible fixed version.
4. Developer validates the application
   ```
   ./mvnw clean verify
   ```
   Runs functional and integration tests.
5. Security pipeline runs again

   SonarQube and Trivy repeat their checks against the new source commit.
6. Security approval is recorded

   The actual scanner results and policy status are retained.
7. Pipeline becomes eligible for the next stage

   Only after all required security checks pass can the process move to Chapter 197.

## 25.2 Why not just ignore the vulnerability?

Because the vulnerability may affect a package used by the production application.

Ignoring it without investigation can introduce risk into:

- Order processing.
- Customer information.
- Database connectivity.
- Application authentication.
- External integrations.

An exception must be based on evidence, risk assessment, an owner and an expiry date.

# 26. End-to-End Security Execution for ECOM-4821

Now connect everything to our ongoing production simulation.

| Step | Activity                         | Responsible team   | Expected outcome            |
| ---- | -------------------------------- | ------------------ | --------------------------- |
| 1    | Developer pushes feature changes | Developer          | New commit                  |
| 2    | CI build and tests               | DevOps / CI        | Build and test result       |
| 3    | SonarQube analysis               | DevOps / Developer | Code analysis result        |
| 4    | Dependency scanning              | DevOps / Security  | Dependency findings         |
| 5    | Secret scanning                  | DevOps / Security  | Secret findings             |
| 6    | Configuration scanning           | Platform / DevOps  | Configuration findings      |
| 7    | Veracode analysis                | Security / DevOps  | Application security result |
| 8    | Security policy evaluation       | Security           | Pass or fail                |
| 9    | Security evidence retained       | DevOps / Security  | Audit records               |
| 10   | Eligibility for Docker build     | DevOps             | Ready for Chapter 197       |

## 26.1 Current simulation metadata

```
Jira Issue:
ECOM-4821

Feature:
Order Status Tracking

Microservice:
order-service

Source Commit:
d4e5f6a...

CI Run:
<ACTUAL_CI_RUN_ID>

SonarQube Project:
order-service

Trivy Scan:
<PENDING_ACTUAL_EXECUTION>

Veracode Analysis:
<PENDING_ACTUAL_EXECUTION>

Security Policy:
<PENDING_ACTUAL_EVALUATION>

Docker Image:
NOT BUILT YET

ECR Image:
NOT PUBLISHED YET
```

These statuses are intentionally explicit.

We have described and prepared the implementation, but we have not actually executed the organization's scanners or verified a real security result.

## 26.2 Expected security evidence

Before moving to Chapter 197, retain:

```
security-evidence/
│
├── sonar-analysis-link.txt
├── trivy-fs-report.json
├── trivy-config-report.json
├── trivy-secrets-report.json
├── order-service-sbom.json
├── veracode-result.json
├── security-policy-result.json
└── scan-metadata.json
```

The Veracode result and policy result filenames represent the desired evidence format; the actual integration may generate different report formats.

Store evidence in an approved restricted location rather than committing sensitive reports to the application repository.

# 27. Security Approval and Handoff Rules

Before the application proceeds to Docker image creation, the following conditions must be satisfied:

1. CI build and tests passed.
2. SonarQube analysis completed.
3. Required quality gate passed.
4. Trivy dependency scan completed.
5. Required vulnerability policy passed.
6. Secret scan completed.
7. No unapproved exposed credentials remain.
8. Configuration scan completed.
9. Required configuration policy passed.
10. Veracode analysis completed if mandated by policy.
11. Veracode policy passed if mandated by policy.
12. All exceptions are formally approved.
13. Security reports are retained.
14. Results are associated with the correct source commit.

Only then should the source revision become eligible for the next stage.

Important: A security approval is associated with a particular source revision and scan configuration. If the source code changes, the relevant checks must be repeated.

# 28. Common Interview Scenarios

## Scenario 1: SonarQube passes, but Trivy fails. What do you do?

Answer:

SonarQube and Trivy analyze different security areas.

I first review the Trivy finding to identify the affected dependency or package.

I check whether it is a direct or transitive dependency using Maven's dependency tree.

Then I identify the fixed version, update the dependency, run application tests and execute the security scan again.

If the finding is valid and violates the policy, I do not promote the application.

## Scenario 2: A developer accidentally commits AWS credentials. What do you do?

Answer:

I treat the credentials as compromised.

First, I revoke or rotate the exposed credentials.

Then I investigate whether they were used, check relevant access logs, remove them from the source and review Git history.

I also introduce or strengthen secret scanning to prevent similar mistakes.

## Scenario 3: The security scan takes 15 minutes. How do you optimize it?

Answer:

I analyze the time taken by each scanning stage.

I check dependency download time, scanner initialization, vulnerability database updates and external service response time.

I use safe dependency caching, approved scanner version management and parallel execution for independent stages.

I do not remove mandatory security checks just to improve pipeline speed.

## Scenario 4: A critical vulnerability is found, but the business wants immediate deployment. What do you do?

Answer:

I communicate the security impact and explain the release policy.

I work with the Security team to determine whether the vulnerability is applicable and exploitable.

If it is a genuine blocking issue, deployment should not proceed without a formally approved exception under the organization's policy.

Any exception must include a documented risk assessment, compensating controls, an owner and an expiry date.

## Scenario 5: Trivy cannot access its vulnerability database. Should the pipeline pass?

Answer:

No. A failed scanner execution is not a successful security scan.

I check DNS, network connectivity, proxy settings and database access.

If the issue cannot be resolved immediately, I follow the organization's documented scanner-outage procedure.

I never silently report an incomplete scan as clean.

## Scenario 6: Why do we need image scanning when dependencies are already scanned?

Answer:

Dependency scanning checks application dependencies, while container image scanning checks packages and components included in the final image.

The image may contain vulnerable operating system packages, runtime libraries or other binaries that are not covered by application dependency scanning.

Therefore, both checks are necessary.

## Scenario 7: Should every vulnerability fail the pipeline?

Answer:

Not necessarily.

We define blocking thresholds based on severity, exploitability, service exposure and business risk.

Critical vulnerabilities and confirmed exposed credentials may be blocking according to company policy.

Other findings can be tracked with remediation deadlines.

All exceptions must be documented and approved.

## Scenario 8: How do you implement security scanning across 21 microservices?

Answer:

I create a reusable security workflow in the Platform Engineering repository.

Each microservice consumes the shared workflow.

We standardize SonarQube, Trivy, secret scanning, reporting and policy enforcement.

The Security team maintains the policy, while the application teams own remediation.

This avoids duplicated workflow code and inconsistent security controls.

# 29. Production Security Checklist

Chapter 196 readiness

# 0/24

SonarQube project configured for order-service

SonarQube authentication stored securely

Maven coverage report generated

SonarQube quality gate configured

Trivy installed at an approved pinned version

Filesystem vulnerability scan configured

Dependency tree reviewed when necessary

Configuration scanning enabled

Secret scanning enabled

Security thresholds approved

Blocking and warning policies documented

False-positive process documented

Exception ownership and expiry defined

Veracode integration configured if required

Veracode result and policy status checked

GitHub Actions permissions restricted

Untrusted pull requests cannot access privileged secrets

Third-party actions pinned to verified SHAs

Security reports retained securely

SBOM generation configured

Scanner execution failures handled correctly

Security results linked to source commit

No Docker image built prematurely

No image published to ECR prematurely

Reset checklist

# 30. Final Understanding

At this point, understand the difference between these stages:

| Stage                  | What it proves                                                           |
| ---------------------- | ------------------------------------------------------------------------ |
| Build                  | Application compiles                                                     |
| Unit testing           | Individual code behavior is tested                                       |
| Integration testing    | Components work together in tested scenarios                             |
| SonarQube              | Configured source-code analysis completed and quality gate evaluated     |
| Trivy filesystem       | Filesystem and supported dependencies were scanned                       |
| Secret scanning        | Supported secret patterns were checked                                   |
| Configuration scanning | Supported configuration files were assessed                              |
| Veracode               | Configured application security analysis and policy evaluation completed |
| Security approval      | Required organizational security conditions are satisfied                |

A green CI pipeline alone is not enough.

We need evidence that all mandatory security checks completed and the policy passed for the exact source revision.

# 31. Chapter 196 Completion Status

Implementation design documented

| Item                  | Status in our simulation                                      |
| --------------------- | ------------------------------------------------------------- |
| Source commit         | `d4e5f6a...` — illustrative                                   |
| CI build and tests    | Passed in Chapter 195 simulation                              |
| SonarQube             | Configuration and execution documented; actual result pending |
| Trivy dependency scan | Configuration documented; actual result pending               |
| Secret scan           | Configuration documented; actual result pending               |
| Configuration scan    | Configuration documented; actual result pending               |
| Veracode              | Integration design documented; actual result pending          |
| Security policy       | Defined as an example; actual evaluation pending              |
| Docker image          | Not built                                                     |
| ECR publishing        | Not started                                                   |

Do not mark the production release as security-approved until the real scan results and required policy decisions have been verified.

# 32. Next Chapter

Next — Chapter 197

# Docker Image Build

Filename: `197-Docker-Image-Build.md`

We will take the approved Order Service source and build its production Docker image.

Topics:

1. Production Docker architecture.
2. Multi-stage Docker builds.
3. Java 21 runtime image.
4. Build optimization and layer caching.
5. Non-root container execution.
6. Image size optimization.
7. Docker build arguments and environment variables.
8. Secure handling of build secrets.
9. Image labels and commit metadata.
10. Immutable image tagging.
11. Container image scanning.
12. SBOM generation for the built image.
13. Image digest verification.
14. Build reproducibility.
15. Troubleshooting and production validation.

Important: We will build the image once and carry that exact artifact through the remaining environments. We will not rebuild it separately for DEV, QA, STAGING and PROD.