# 14-SonarQube.md

## 1. Purpose

We now have three CI systems:

```text
GitHub Actions
GitLab CI
Jenkins
```

All three will use the same SonarQube quality process:

```text
                  Source Code
                       |
          ┌────────────┼────────────┐
          ↓            ↓            ↓
       GitHub        GitLab       Jenkins
       Actions         CI         Pipeline
          |            |            |
          └────────────┼────────────┘
                       ↓
                   SonarQube
                       |
          ┌────────────┼────────────┐
          ↓            ↓            ↓
        Bugs      Vulnerabilities  Code Smells
          |            |            |
          └────────────┼────────────┘
                       ↓
                    Coverage
                       |
                       v
                 Quality Gate
                   /       \
                PASS       FAIL
                  |          |
                  v          X
                 ECR       STOP
```

The goal is to understand **how SonarQube becomes an actual CI quality gate**, not merely how to execute `mvn sonar:sonar`.

---

# 2. What SonarQube Does

SonarQube performs automated analysis of source code.

For our Java `order-service`, it can analyze areas such as:

```text
Bugs
Vulnerabilities
Code Smells
Security Hotspots
Duplications
Coverage
Maintainability
Reliability
Security
```

The exact rules and metrics depend on the SonarQube edition/version and configured quality profile.

---

# 3. What SonarQube Does NOT Do

SonarQube is not:

```text
❌ A replacement for unit tests
❌ A replacement for Trivy
❌ A replacement for dependency scanning
❌ A replacement for penetration testing
❌ A replacement for runtime monitoring
❌ A replacement for code review
❌ A replacement for WAF
❌ A replacement for Kubernetes security
```

Instead:

```text
                  Application Security
                          |
       ┌──────────────────┼──────────────────┐
       ↓                  ↓                  ↓
   Source Code       Dependencies        Container
    SonarQube          Scanner             Trivy
       |                  |                  |
       └──────────────────┼──────────────────┘
                          ↓
                       Runtime
```

---

# 4. SonarQube Architecture

Our learning architecture:

```text
Developer
    |
    v
Git Repository
    |
    v
CI Pipeline
    |
    v
Sonar Scanner
    |
    v
SonarQube Server
    |
    ├── Analysis
    ├── Quality Profiles
    ├── Quality Gates
    ├── Projects
    └── Database
```

The CI runner/agent executes the scanner.

The SonarQube server performs/stores the analysis and evaluates the configured policies.

---

# 5. CI Integration

We have:

```text
GitHub Actions
      |
      | Maven Sonar Scanner
      v
  SonarQube
```

```text
GitLab CI
      |
      | Maven Sonar Scanner
      v
  SonarQube
```

```text
Jenkins
      |
      | Maven Sonar Scanner
      v
  SonarQube
```

One SonarQube platform can therefore serve multiple CI systems.

---

# 6. SonarQube Project

Create a project for:

```text
order-service
```

A project needs a unique project key.

Example:

```text
company-ecommerce-order-service
```

We should establish a naming convention for all 21 services.

For example:

```text
company-ecommerce-auth-service
company-ecommerce-user-service
company-ecommerce-customer-service
company-ecommerce-catalog-service
company-ecommerce-product-service
company-ecommerce-inventory-service
company-ecommerce-pricing-service
company-ecommerce-promotion-service
company-ecommerce-cart-service
company-ecommerce-order-service
company-ecommerce-payment-service
...
```

This makes searching and administration easier.

---

# 7. Why Project Keys Matter

Suppose SonarQube contains:

```text
21 microservices
```

Without a consistent naming convention:

```text
order
orderservice
order-api
order-service-prod
orders
```

becomes confusing.

Use:

```text
company-ecommerce-<service-name>
```

Example:

```text
company-ecommerce-order-service
```

---

# 8. `pom.xml` Configuration

For Maven projects, SonarQube can be invoked through Maven.

Our CI command can be:

```bash
./mvnw -B verify sonar:sonar
```

with parameters:

```bash
-Dsonar.host.url="..."
-Dsonar.token="..."
```

We can also define project properties explicitly.

For example:

```xml
<properties>

    <sonar.projectKey>
        company-ecommerce-order-service
    </sonar.projectKey>

</properties>
```

However, keeping CI/project configuration centralized can also be preferable.

For this project, we'll keep the essential project identity in source configuration and environment-specific credentials outside Git.

---

# 9. Never Commit the Token

Never do:

```xml
<sonar.token>
    abc123...
</sonar.token>
```

Never put:

```text
SONAR_TOKEN=...
```

into:

```text
application.yaml
```

or:

```text
.gitlab-ci.yml
```

or:

```text
Jenkinsfile
```

Instead:

```text
CI Secret Store
       |
       v
SONAR_TOKEN
       |
       v
Sonar Scanner
```

---

# 10. SonarQube Token Flow

For GitHub:

```text
GitHub Secret
     |
     v
SONAR_TOKEN
     |
     v
Maven
     |
     v
SonarQube
```

GitLab:

```text
GitLab Protected Variable
     |
     v
SONAR_TOKEN
     |
     v
Maven
```

Jenkins:

```text
Jenkins Credentials
     |
     v
SONAR_TOKEN
     |
     v
Maven
```

---

# 11. Quality Profile

A **Quality Profile** defines which rules are active.

Think:

```text
Quality Profile
       |
       ├── Java Rule A
       ├── Java Rule B
       ├── Java Rule C
       └── Security Rule D
```

For example, the profile may detect:

```text
unused code
bad exception handling
unsafe patterns
complex methods
duplicated logic
security-sensitive code
```

---

# 12. Quality Profile vs Quality Gate

These are different.

### Quality Profile

Answers:

> **What rules should SonarQube analyze?**

```text
Profile
   |
   ├── Rule 1
   ├── Rule 2
   └── Rule 3
```

### Quality Gate

Answers:

> **When should the CI pipeline pass or fail?**

```text
Analysis
   |
   v
Quality Gate
   |
   ├── PASS
   └── FAIL
```

Remember:

```text
Profile = rules
Gate    = decision
```

---

# 13. Quality Gate

For our project, define a starting policy around **new code**.

Example conceptual policy:

```text
New Code:

Coverage             >= 80%
New Bugs             = 0
New Vulnerabilities  = 0
Duplications         <= 3%
Security Rating      = acceptable
Reliability Rating  = acceptable
Maintainability     = acceptable
```

These are example project targets, not universal values.

The organization should decide the actual thresholds.

---

# 14. Why New Code?

Suppose our application eventually has:

```text
500,000 lines of legacy code
```

and:

```text
8,000 historical code smells
```

If we require the entire repository to become perfect before every PR:

```text
Developer
   |
   v
Fix 8,000 issues
   |
   v
PR
```

the team will struggle to make progress.

Instead:

```text
Legacy
   |
   └── Existing technical debt

New Code
   |
   └── Must meet quality standards
```

This allows technical debt to be reduced progressively.

---

# 15. Coverage

Coverage measures how much code is exercised by tests.

Example:

```text
100 executable lines
80 covered
```

gives roughly:

```text
80% coverage
```

But:

> High coverage does not automatically mean high-quality tests.

For example:

```java
assertTrue(true);
```

can technically execute code without meaningfully validating behavior.

Therefore:

```text
Coverage
+
Test quality
+
Code review
```

all matter.

---

# 16. JaCoCo

Our Maven project uses JaCoCo.

The flow:

```text
JUnit Tests
    |
    v
JaCoCo Agent
    |
    v
Coverage Data
    |
    v
jacoco.xml
    |
    v
SonarQube
```

Typical location:

```text
target/site/jacoco/jacoco.xml
```

---

# 17. Verify Coverage Locally

Run:

```bash
./mvnw clean verify
```

Then:

```bash
ls target/site/jacoco/
```

You should see coverage output.

Open the HTML report locally if needed.

The important CI artifact is generally:

```text
jacoco.xml
```

---

# 18. SonarQube + JaCoCo

The flow is:

```text
./mvnw verify
      |
      ├── Compile
      ├── Test
      ├── JaCoCo
      └── Coverage Report
              |
              v
         sonar:sonar
              |
              v
          SonarQube
```

SonarQube then uses the analysis data and coverage report to calculate the relevant quality metrics.

---

# 19. GitHub Actions Integration

Our GitHub workflow contains:

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

The critical sequence is:

```text
Maven
 ↓
SonarQube Analysis
 ↓
Wait
 ↓
Quality Gate
```

---

# 20. What `qualitygate.wait` Does

Without waiting:

```text
Jenkins/GitHub/GitLab
      |
      v
Submit analysis
      |
      v
CI continues
      |
      v
ECR
```

The quality result may not yet be available.

With:

```text
sonar.qualitygate.wait=true
```

the CI process waits for the quality gate result.

```text
CI
 |
 v
Submit Analysis
 |
 v
Wait
 |
 v
Quality Gate
 |
 ├── PASS
 |
 └── FAIL
```

---

# 21. GitLab Integration

Our GitLab job:

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

Variables come from:

```text
GitLab CI/CD Variables
```

rather than source control.

---

# 22. Jenkins Integration

Our Jenkins stage:

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

Then:

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

Architecture:

```text
Jenkins
   |
   v
SonarQube Analysis
   |
   v
Webhook / Result
   |
   v
Quality Gate
   |
   ├── PASS
   └── FAIL
```

---

# 23. Why Jenkins Uses a Webhook

A mature Jenkins/SonarQube integration should avoid unnecessarily polling continuously.

The flow can be:

```text
Jenkins
   |
   | Start analysis
   v
SonarQube
   |
   | Analyze
   v
Quality Gate
   |
   | Webhook
   v
Jenkins
```

Jenkins then knows:

```text
PASS
```

or:

```text
FAIL
```

and can continue/abort the pipeline.

---

# 24. Pull Request Analysis

We also want SonarQube to understand:

```text
PR
```

rather than treating every scan as an unrelated branch.

Example:

```text
main
 |
 ├── feature/order-discount
 |
 ├── feature/payment-validation
 |
 └── bugfix/order-status
```

SonarQube can provide PR-focused analysis where the edition/integration supports it.

The goal is:

```text
PR
 |
 v
Changed code
 |
 v
SonarQube
 |
 v
Quality result
```

---

# 25. PR Feedback

Developer creates:

```text
feature/order-discount
```

PR:

```text
feature/order-discount → main
```

CI runs:

```text
Build
Test
SonarQube
```

Suppose SonarQube finds:

```text
New vulnerability
```

Then:

```text
Quality Gate
      |
      X
      |
   PR blocked
```

The developer fixes the code.

---

# 26. Branch Protection

Now connect SonarQube to GitHub branch protection.

Our `main` branch requires:

```text
PR
 ↓
Required checks
 ↓
SonarQube PASS
 ↓
Tests PASS
 ↓
Security PASS
 ↓
Approval
 ↓
Merge
```

This means developers cannot simply ignore a failed quality gate.

---

# 27. GitLab Protected Branch

Similarly:

```text
main
 |
 v
Protected Branch
 |
 ├── Required pipeline
 ├── Security checks
 └── Approval
```

The exact approval configuration depends on GitLab edition and organization policy.

---

# 28. Jenkins Branch Protection

For Jenkins:

```text
GitHub PR
    |
    v
Jenkins
    |
    v
Quality Gate
    |
    v
GitHub status
```

The Jenkins result can become a required status check before merging.

---

# 29. SonarQube Failure Example

Suppose a developer introduces:

```java
String query =
    "SELECT * FROM orders WHERE id = " + id;
```

SonarQube may flag security-related issues depending on the rule set/context.

Pipeline:

```text
Build
 ↓
Test
 ↓
SonarQube
 ↓
Security issue
 ↓
Quality Gate FAIL
 ↓
CI FAIL
```

Therefore:

```text
No ECR
No deployment
```

---

# 30. Quality Gate Is Not a Magic Security Scanner

Important distinction:

```text
SonarQube
   |
   v
Static analysis
```

It cannot guarantee:

```text
"Application is completely secure."
```

For example, business logic can be wrong while passing static analysis.

Example:

```text
User has ₹1,000
Order costs ₹2,000
Application accidentally approves order
```

That may require:

```text
Unit tests
Integration tests
Security testing
Business validation
Code review
```

---

# 31. Quality Gate Design

We should avoid gates that are so strict that developers start bypassing them.

Bad:

```text
Any warning
    ↓
BLOCK
```

Better:

```text
Critical issue
    ↓
BLOCK

High-risk issue
    ↓
BLOCK/review

Minor issue
    ↓
Report
```

The exact policy should be agreed between:

```text
Architecture
Security
Development
DevOps
SRE
```

---

# 32. SonarQube Server Availability

Our CI pipeline now depends on SonarQube.

Therefore:

```text
GitHub Runner
      |
      v
SonarQube
```

If SonarQube is unavailable:

```text
CI
 |
 v
SonarQube
 |
 X
Unavailable
```

we need a defined policy.

For protected production paths, failing closed is often appropriate:

```text
SonarQube unavailable
       |
       v
Quality decision unavailable
       |
       v
Don't release
```

For non-production development workflows, organizations sometimes choose a different availability policy.

---

# 33. SonarQube High Availability

For a serious enterprise environment, SonarQube itself becomes infrastructure.

Conceptually:

```text
             Load Balancer
                   |
                   v
              SonarQube
                   |
             ┌─────┴─────┐
             ↓           ↓
         Sonar Node   Sonar Node
             |
             v
         PostgreSQL
```

The exact topology depends on SonarQube edition and supported architecture.

We don't need to build HA immediately for our learning environment.

---

# 34. SonarQube Database

SonarQube requires persistent database infrastructure.

For our AWS environment:

```text
SonarQube
    |
    v
PostgreSQL
```

This is separate from:

```text
E-Commerce Order DB
```

Do not mix them.

We have:

```text
SonarQube PostgreSQL
       ≠
Order Service PostgreSQL
```

---

# 35. SonarQube Data

SonarQube stores analysis-related data such as:

```text
Projects
Issues
Measures
Quality Gates
Quality Profiles
Users
Permissions
Analysis history
```

Therefore backups and persistence matter.

---

# 36. Authentication

CI needs to authenticate to SonarQube.

Possible approaches depend on the SonarQube setup, but our CI should use a dedicated token/service identity rather than a developer's personal credentials.

Conceptually:

```text
CI
 |
 v
Service Token
 |
 v
SonarQube
```

Avoid:

```text
Developer's personal token
       |
       v
Production CI
```

because employee departure or permission changes could break the pipeline.

---

# 37. Least Privilege

The CI identity should have only the permissions required for analysis.

Think:

```text
WHO
 ↓
CAN DO WHAT
 ↓
ON WHICH RESOURCE
```

For example:

```text
CI Order Service
    |
    └── Analyze company-ecommerce-order-service
```

rather than unnecessarily granting broad administrative access.

---

# 38. Separate Projects

For 21 services:

```text
company-ecommerce-auth-service
company-ecommerce-user-service
company-ecommerce-customer-service
...
company-ecommerce-order-service
...
company-ecommerce-reporting-service
```

Each project can have:

```text
Source
Issues
Metrics
Quality Gate
History
```

while organization-wide rules can be standardized through shared profiles and policies.

---

# 39. Monorepo vs Multi-Repo

Our project uses:

```text
21 application repositories
```

Therefore:

```text
1 Repository
   |
   v
1 Microservice
   |
   v
1 SonarQube Project
```

Conceptually:

```text
order-service repo
       |
       v
company-ecommerce-order-service
```

This is straightforward.

---

# 40. Quality Gate Across 21 Services

We want consistency:

```text
                    Organization Policy
                           |
        ┌──────────────────┼──────────────────┐
        ↓                  ↓                  ↓
       Auth              Order             Payment
        |                  |                  |
        v                  v                  v
    SonarQube          SonarQube          SonarQube
        |                  |                  |
        └──────────────────┼──────────────────┘
                           ↓
                    Common Standards
```

The actual code differs.

The quality standards should be broadly consistent.

---

# 41. Local Developer Workflow

Before pushing:

```bash
./mvnw clean verify
```

Then:

```bash
git add .
git commit -m "feat: add order validation"
git push
```

CI performs:

```text
Build
 ↓
Test
 ↓
SonarQube
 ↓
Security
```

This gives fast feedback.

---

# 42. Production CI Flow

Our final CI flow is becoming:

```text
Developer
   |
   v
PR
   |
   v
Build
   |
   v
Unit Tests
   |
   v
JaCoCo
   |
   v
SonarQube
   |
   v
Quality Gate
   |
   v
Dependency Scan
   |
   v
Gitleaks
   |
   v
Docker Build
   |
   v
Trivy
   |
   v
SBOM
   |
   v
Image Signing
   |
   v
ECR
```

---

# 43. Three CI Systems + SonarQube

The final relationship:

```text
                       SonarQube
                           ^
                           |
             ┌─────────────┼─────────────┐
             |             |             |
             |             |             |
        GitHub Actions  GitLab CI      Jenkins
             |             |             |
             └─────────────┼─────────────┘
                           |
                           v
                    Common Policy
```

This is exactly what we want.

---

# 44. Troubleshooting SonarQube

When analysis fails, don't immediately change the pipeline.

Follow the chain:

```text
CI Job
  |
  v
Maven
  |
  v
Sonar Scanner
  |
  v
Network
  |
  v
SonarQube
  |
  v
Authentication
  |
  v
Project
  |
  v
Analysis
  |
  v
Quality Gate
```

---

# 45. Troubleshooting Example: Authentication

Error conceptually:

```text
401 Unauthorized
```

Investigate:

```text
SONAR_TOKEN
     |
     ├── Exists?
     ├── Correct?
     ├── Expired?
     └── Has permission?
```

Don't immediately change:

```text
sonar.host.url
```

without checking the actual failure.

---

# 46. Troubleshooting Example: Network

Error:

```text
Connection refused
```

Check:

```text
CI Runner
    |
    v
DNS
    |
    v
Network
    |
    v
Firewall
    |
    v
Load Balancer
    |
    v
SonarQube
```

If GitHub-hosted runners cannot access an internal SonarQube endpoint, use an appropriate reachable runner/network architecture.

---

# 47. Troubleshooting Example: Quality Gate

Suppose:

```text
SonarQube analysis SUCCESS
```

but:

```text
Quality Gate FAILED
```

These are different.

Analysis succeeded:

```text
Scanner → Server
```

Quality gate failed:

```text
Policy evaluation
       |
       X
```

Inspect:

```text
New vulnerabilities
Coverage
Bugs
Duplications
Security hotspots
```

Then fix the actual condition.

---

# 48. Important Mental Model

Remember these four:

```text
SonarQube
   ↓
Analysis

Quality Profile
   ↓
Rules

Quality Gate
   ↓
Decision

CI
   ↓
Enforcement
```

That's the easiest way to remember the architecture.

---

# 49. Chapter Summary

We now understand:

```text
1. SonarQube architecture
2. SonarQube projects
3. Project keys
4. Quality Profiles
5. Quality Gates
6. New Code strategy
7. JaCoCo
8. Coverage
9. GitHub integration
10. GitLab integration
11. Jenkins integration
12. Tokens
13. Least privilege
14. PR analysis
15. Branch protection
16. Failure handling
17. Troubleshooting
18. 21-service standardization
```

Our CI security chain is now:

```text
             Source
                |
                v
             Build
                |
                v
              Test
                |
                v
           SonarQube
                |
                v
          Quality Gate
                |
                v
         Dependency Scan
                |
                v
            Gitleaks
                |
                v
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
               ECR
```

---

# Next: `15-Trivy.md`

We'll now go deep into **Trivy**, rather than treating it as one command in the CI YAML.

We'll cover:

```text
Trivy Architecture
       ↓
Filesystem Scanning
       ↓
Dependency Scanning
       ↓
Dockerfile Misconfiguration
       ↓
Container Image Scanning
       ↓
OS Packages
       ↓
Java Dependencies
       ↓
Severity
       ↓
Fixable vs Unfixable
       ↓
Ignore Policies
       ↓
CI Exit Codes
       ↓
GitHub Actions
       ↓
GitLab CI
       ↓
Jenkins
       ↓
ECR Scanning
       ↓
SBOM Generation
       ↓
Vulnerability Exceptions
       ↓
Production Security Policy
```

Then we'll have the complete **code → dependency → container security pipeline** before moving into ECR image versioning and artifact promotion.
