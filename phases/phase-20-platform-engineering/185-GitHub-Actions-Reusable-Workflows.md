# Chapter 185 — GitHub Actions Reusable Workflows

Phase 20 — Platform Engineering

Project: Enterprise E-commerce Platform with 21 Microservices\
Cloud: AWS, Amazon EKS, Amazon ECR\
Source control: GitHub\
CI/CD: GitHub Actions, Jenkins and GitLab CI/CD where required\
Deployment model: GitOps with Argo CD and Helm\
Infrastructure: Terraform\
Security: SonarQube, Trivy, dependency scanning, secret scanning and software supply-chain controls

This is the expanded continuation of Chapter 185. It covers the reusable workflow catalog, implementation patterns, security controls, workflow contracts, service integration, release promotion, and production troubleshooting. The complete chapter is intended to be built in substantial parts rather than shortened into a high-level overview.

## 1. Objective

The objective is to build a reusable CI/CD platform that standardizes how all 21 microservices are built, tested, scanned, packaged, published and prepared for deployment.

Without reusable workflows, each service team may maintain its own pipeline. Over time, the pipelines become inconsistent.

For example:

- The Order Service scans its dependencies but the Payment Service does not.
- One service publishes images using immutable tags while another overwrites `latest`.
- One service runs unit tests before building while another skips them.
- One pipeline uses AWS access keys while another uses OIDC.
- One service scans an image and then rebuilds a different image for publishing.
- One service updates GitOps only after security checks, while another bypasses those checks.

The platform team should prevent this type of inconsistency by providing approved reusable workflows with documented contracts and enforceable controls.

The desired result is:

1. Developers focus on application code.
2. Service repositories call approved workflows.
3. Shared workflows enforce the common engineering standards.
4. Security failures block releases according to defined policy.
5. Approved container images are published to Amazon ECR.
6. GitOps repositories promote exact image digests between environments.
7. Argo CD deploys the approved state into EKS.
8. Deployment verification and release evidence remain available for audits and troubleshooting.

Important: A reusable workflow is a shared execution mechanism. It does not automatically become mandatory merely because it exists. Repository rules, protected environments, organization policy and access controls must enforce its use.

## 2. Production architecture

[End-to-end CI/CD on EKS: GitHub Actions, OIDC, and ArgoCD GitOps | by Ibrahim Cisse | Apr, 2026 | Medium](https://images.openai.com/static-rsc-4/NxYz2c538D_rG25agOI3EoNKo--RggNFf-BrR4KA29cjgDNAeZZo3Y-L1roVCZeaUio8FeBcEgDjgLbndf8LBeb8fuZgo4tA7wiA2gE4W3EGG-vsJqLY5QRiXVxCP3_-KMHSP2CHfl2M5UPjfuRu9yX0APekrACoMN8Z_Lj871s?purpose=inline)

[medium.com](https://medium.com/%40Ibraheemcisse/end-to-end-ci-cd-on-eks-github-actions-oidc-and-argocd-gitops-bdd1556cd463?utm_source=chatgpt.com)

```
Developer
   |
   v
Service Repository
   |
   v
Pull Request
   |
   v
GitHub Actions Caller Workflow
   |
   +--> Reusable Language CI
   |      +--> Compile
   |      +--> Unit Tests
   |      +--> Code Quality
   |
   +--> Reusable Security Workflows
   |      +--> Dependency Scan
   |      +--> Secret Scan
   |      +--> SAST
   |      +--> License Compliance
   |
   +--> Reusable Artifact Workflows
   |      +--> Build Container Image
   |      +--> Container Scan
   |      +--> Generate SBOM
   |
   v
Protected Release Workflow
   |
   +--> AWS OIDC
   +--> Publish Approved Image to ECR
   +--> Verify Exact Digest
   +--> Sign and Record Provenance
   |
   v
GitOps Repository Pull Request
   |
   +--> Update Image Digest
   +--> Validate Helm / Kubernetes Policies
   +--> Environment Approval
   |
   v
Argo CD
   |
   v
Amazon EKS
   |
   +--> Deployment Verification
   +--> Monitoring and Alerting
   +--> Rollback / Recovery Process
```

This architecture separates four responsibilities:

| Layer                        | Responsibility                                                           |
| ---------------------------- | ------------------------------------------------------------------------ |
| Service repository           | Application source, tests, Dockerfile and service-specific configuration |
| Shared workflow repository   | Reusable pipeline orchestration and common CI/CD logic                   |
| Security and platform policy | Defines mandatory controls, thresholds and exceptions                    |
| GitOps repository            | Desired deployment state for DEV, QA, STAGING and production             |

The deployment path is deliberately separated from application CI. A successful build does not itself authorize a production deployment.

## 3. Repository structure

Create a dedicated repository for reusable GitHub Actions workflows.

```
company-ecommerce/
|
+-- platform-github-actions/
|   |
|   +-- .github/
|   |   +-- workflows/
|   |       +-- reusable-java-ci.yml
|   |       +-- reusable-nodejs-ci.yml
|   |       +-- reusable-python-ci.yml
|   |       +-- reusable-go-ci.yml
|   |       +-- reusable-code-quality.yml
|   |       +-- reusable-dependency-scan.yml
|   |       +-- reusable-secret-scan.yml
|   |       +-- reusable-sast.yml
|   |       +-- reusable-container-build.yml
|   |       +-- reusable-container-scan.yml
|   |       +-- reusable-ecr-publish.yml
|   |       +-- reusable-image-verify.yml
|   |       +-- reusable-sbom.yml
|   |       +-- reusable-image-sign.yml
|   |       +-- reusable-provenance.yml
|   |       +-- reusable-terraform-validate.yml
|   |       +-- reusable-terraform-security.yml
|   |       +-- reusable-terraform-plan.yml
|   |       +-- reusable-helm-validate.yml
|   |       +-- reusable-kubernetes-policy.yml
|   |       +-- reusable-api-contract-test.yml
|   |       +-- reusable-integration-test.yml
|   |       +-- reusable-performance-test.yml
|   |       +-- reusable-release.yml
|   |       +-- reusable-gitops-promotion.yml
|   |       +-- reusable-deployment-verification.yml
|   |       +-- reusable-rollback-validation.yml
|   |       +-- reusable-license-compliance.yml
|   |       +-- reusable-owasp-dast.yml
|   |       +-- reusable-evidence-archive.yml
|   |
|   +-- actions/
|   |   +-- setup-java/
|   |   +-- setup-node/
|   |   +-- setup-python/
|   |   +-- setup-go/
|   |   +-- verify-image-digest/
|   |
|   +-- docs/
|   |   +-- workflow-catalog.md
|   |   +-- workflow-contracts.md
|   |   +-- onboarding.md
|   |   +-- versioning.md
|   |   +-- troubleshooting.md
|   |
|   +-- tests/
|       +-- workflow-contracts/
|       +-- security-fixtures/
|       +-- integration/
|
+-- order-service/
+-- payment-service/
+-- inventory-service/
+-- catalog-service/
+-- gitops/
+-- platform-infrastructure/
```

The directory names are an intended design. They must be created and tested in the actual repository; listing a file here does not mean it already exists.

## 4. Reusable workflow versus composite action

These two GitHub Actions features solve different problems.

### Reusable workflow

A reusable workflow can contain multiple jobs, dependencies between jobs, environment configuration and job-level permissions.

Example:

```
jobs:
  ci:
    uses: company-ecommerce/platform-github-actions/.github/workflows/reusable-java-ci.yml@v1
```

A reusable workflow is called at the job level, under `jobs.<job_id>.uses`.

### Composite action

A composite action groups multiple steps into a reusable action.

Example:

```
steps:
  - uses: company-ecommerce/platform-github-actions/actions/setup-java@v1
```

A composite action is called at the step level, under `steps[].uses`.

### Practical rule

Use reusable workflows when the platform needs to standardize a whole pipeline or job graph. Use composite actions for smaller operations that should be shared across workflows.

Do not turn every shell command into a separate reusable workflow. Excessive fragmentation makes debugging and ownership more difficult.

## 5. The workflow contract

Every shared workflow needs an explicit contract.

The contract defines what the caller supplies, what the workflow guarantees, what permissions it needs and what happens when it fails.

A workflow contract should document:

- Purpose and supported languages or artifact types.
- Required and optional inputs.
- Output names and meanings.
- Required permissions.
- Required secrets, if any.
- Runner requirements.
- Artifact naming and retention.
- Security gates and failure behavior.
- Supported workflow versions.
- Compatibility guarantees.
- Maintainer and escalation path.
- Test evidence and upgrade instructions.

Example contract:

```
name: Reusable Java CI

on:
  workflow_call:
    inputs:
      java-version:
        description: Java version required by the service
        required: false
        type: string
        default: "21"

      working-directory:
        description: Directory containing the Maven project
        required: false
        type: string
        default: "."

      run-tests:
        description: Whether unit tests must run
        required: false
        type: boolean
        default: true

    outputs:
      artifact-name:
        description: Name of the build artifact
        value: ${{ jobs.build.outputs.artifact-name }}
```

This is a contract illustration. The workflow must also define a `build` job with the matching output, and its steps must produce the named artifact before the contract is valid.

A required test must not be silently skipped because an input defaults to `false`. Production policy should explicitly distinguish optional tests from mandatory release gates.

## 6. Complete reusable workflow catalog

The following is the proposed catalog for the platform. The purpose of this section is to identify the capabilities that must be implemented and governed, not merely to list filenames.

### 6.1 Application CI workflows

| Workflow                         | Required behavior                                       | Typical failure                       |
| -------------------------------- | ------------------------------------------------------- | ------------------------------------- |
| `reusable-java-ci.yml`           | Compile, run Maven tests, package application           | Compilation or test failure           |
| `reusable-nodejs-ci.yml`         | Install locked dependencies, lint, test and build       | Dependency or test failure            |
| `reusable-python-ci.yml`         | Install locked dependencies, lint, test and package     | Test or static-check failure          |
| `reusable-go-ci.yml`             | Format, test, vet and compile                           | Test, vet or build failure            |
| `reusable-code-quality.yml`      | Run code analysis and enforce configured quality gate   | Quality gate fails                    |
| `reusable-api-contract-test.yml` | Validate consumer/provider API contracts                | Incompatible API change               |
| `reusable-integration-test.yml`  | Test service integration with required dependencies     | Integration test fails                |
| `reusable-performance-test.yml`  | Run authorized performance tests and compare thresholds | Latency or throughput threshold fails |

Not every service needs all four language workflows. Each service should use the workflow matching its actual runtime, confirmed by the service's repository and application manifest.

### 6.2 Security workflows

| Workflow                          | Required behavior                                                       |
| --------------------------------- | ----------------------------------------------------------------------- |
| `reusable-dependency-scan.yml`    | Identify vulnerable direct and transitive dependencies                  |
| `reusable-secret-scan.yml`        | Detect secrets in changed files and repository history where configured |
| `reusable-sast.yml`               | Analyze source code for supported security weaknesses                   |
| `reusable-license-compliance.yml` | Enforce the approved software-license policy                            |
| `reusable-terraform-security.yml` | Detect unsafe infrastructure configurations                             |
| `reusable-container-scan.yml`     | Scan the actual image intended for release                              |
| `reusable-kubernetes-policy.yml`  | Validate manifests and rendered Helm output                             |
| `reusable-owasp-dast.yml`         | Test a designated running application in an authorized environment      |

A scanner's exit code is only one part of the control. The workflow must interpret the results according to the approved policy, preserve the report and fail the appropriate gate.

### 6.3 Artifact and supply-chain workflows

| Workflow                        | Required behavior                                               |
| ------------------------------- | --------------------------------------------------------------- |
| `reusable-container-build.yml`  | Build an image from the reviewed source and build instructions  |
| `reusable-ecr-publish.yml`      | Publish an approved image to the correct ECR repository         |
| `reusable-image-verify.yml`     | Verify the exact registry image and digest                      |
| `reusable-sbom.yml`             | Generate a machine-readable software bill of materials          |
| `reusable-image-sign.yml`       | Sign the approved artifact using the selected signing mechanism |
| `reusable-provenance.yml`       | Record evidence linking the artifact to its source and build    |
| `reusable-evidence-archive.yml` | Retain test, scan, build and release evidence                   |

Critical invariant: the image scanned, published, signed and promoted must be the same image digest. A tag such as `latest` is not a substitute for a verified immutable digest.

### 6.4 Infrastructure and deployment workflows

| Workflow                               | Required behavior                                        |
| -------------------------------------- | -------------------------------------------------------- |
| `reusable-terraform-validate.yml`      | Format, initialize safely, validate and lint Terraform   |
| `reusable-terraform-plan.yml`          | Generate a plan for review before infrastructure changes |
| `reusable-helm-validate.yml`           | Lint charts and render supported environment values      |
| `reusable-release.yml`                 | Verify release eligibility and required evidence         |
| `reusable-gitops-promotion.yml`        | Propose an environment-specific digest update in GitOps  |
| `reusable-deployment-verification.yml` | Verify rollout status and application health             |
| `reusable-rollback-validation.yml`     | Validate the rollback target and recovery requirements   |

Terraform apply, production deployment and rollback must each have appropriate environment and authorization controls. A workflow existing in the repository does not grant it permission to perform those operations.

## 7. Establishing versioning and trust

The shared workflow repository is part of the production software supply chain. A change to a shared workflow can affect every service that calls it.

### 7.1 Versioning strategy

Use a controlled release strategy such as:

```
platform-github-actions/
|
+-- main
|
+-- release/v1
|
+-- release/v2
|
+-- .github/workflows/
```

For an illustrative workflow reference:

```
jobs:
  ci:
    uses: company-ecommerce/platform-github-actions/.github/workflows/reusable-java-ci.yml@v1
```

Here, `v1` is a readable release reference. It is safe only if the organization controls how that reference is moved and verifies the code it resolves to.

For higher assurance, pin reusable workflows and third-party actions to reviewed full commit SHAs. A full SHA is more resistant to a tag being moved unexpectedly.

```
jobs:
  ci:
    uses: company-ecommerce/platform-github-actions/.github/workflows/reusable-java-ci.yml@<REVIEWED_FULL_COMMIT_SHA>
```

Replace the placeholder with a verified commit SHA before using this in a real workflow.

### 7.2 Change management for shared workflows

Before releasing a new version:

1. Open a pull request in the shared workflow repository.
2. Require review from the platform owners.
3. Run workflow syntax and contract tests.
4. Test the change with representative Java, Node.js, Python and Go repositories where applicable.
5. Verify that permissions remain least-privilege.
6. Check that existing callers continue to satisfy the workflow contract.
7. Publish a controlled release reference.
8. Upgrade a small number of pilot services.
9. Review failures and security results.
10. Roll out the version to the remaining services.

Do not update every service to an untested workflow revision just because the shared repository's default branch is green.

### 7.3 Deprecation policy

A reusable workflow should not be removed while active callers still depend on it.

The platform team should maintain:

- Supported workflow versions.
- A list of callers using each version.
- A deprecation date.
- A migration guide.
- A replacement workflow.
- An owner for exceptions.
- A verification report showing which callers have migrated.

## 8. Java CI workflow implementation

This example shows the foundation of `reusable-java-ci.yml`.

```
name: Reusable Java CI

on:
  workflow_call:
    inputs:
      java-version:
        type: string
        required: false
        default: "21"

      working-directory:
        type: string
        required: false
        default: "."

      run-tests:
        type: boolean
        required: false
        default: true

    outputs:
      artifact-name:
        description: Built Java artifact name
        value: ${{ jobs.build.outputs.artifact-name }}

permissions:
  contents: read

jobs:
  build:
    name: Compile and test Java
    runs-on: ubuntu-latest

    outputs:
      artifact-name: ${{ steps.metadata.outputs.artifact-name }}

    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}

    steps:
      - name: Checkout source
        uses: actions/checkout@<REVIEWED_FULL_COMMIT_SHA>

      - name: Set up Java
        uses: actions/setup-java@<REVIEWED_FULL_COMMIT_SHA>
        with:
          distribution: temurin
          java-version: ${{ inputs.java-version }}
          cache: maven

      - name: Verify Maven wrapper
        run: |
          test -f ./mvnw
          test -x ./mvnw

      - name: Compile application
        run: ./mvnw --batch-mode --no-transfer-progress -DskipTests compile

      - name: Run unit tests
        if: ${{ inputs.run-tests }}
        run: ./mvnw --batch-mode --no-transfer-progress test

      - name: Package application
        run: ./mvnw --batch-mode --no-transfer-progress package -DskipTests

      - name: Determine artifact metadata
        id: metadata
        shell: bash
        run: |
          set -euo pipefail
          echo "artifact-name=java-build" >> "$GITHUB_OUTPUT"
```

### 8.1 Important implementation considerations

This example is a foundation, not a complete production workflow.

- Replace every placeholder action SHA with a reviewed full commit SHA.
- Validate that the Maven wrapper is trustworthy and that the project commits its wrapper files.
- Avoid running the same tests unnecessarily in both the `test` and `package` commands.
- Ensure that the build produces the expected JAR or other package.
- Upload the actual artifact if a later job needs to consume it; workflow outputs only communicate small metadata values.
- Define artifact retention and access controls.
- Add code-quality and security workflows as separate required gates or as explicitly governed jobs.
- Avoid passing secrets to workflows that do not need them.
- For untrusted pull requests, do not expose privileged publishing credentials.

The `-DskipTests` option is used here only after the test command has run. If a caller disables tests, a separate policy control must prevent an untested build from qualifying for a protected release.

## 9. Calling the Java workflow from Order Service

The service repository should contain a small caller workflow rather than a full duplicate of the shared implementation.

File: `order-service/.github/workflows/ci.yml`

```
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

jobs:
  java-ci:
    uses: company-ecommerce/platform-github-actions/.github/workflows/reusable-java-ci.yml@<REVIEWED_FULL_COMMIT_SHA>
    with:
      java-version: "21"
      working-directory: "."
      run-tests: true
```

This example uses a placeholder for the shared workflow commit. Replace it with the actual reviewed SHA.

### 9.1 Why this caller is useful

The service repository declares its requirements:

- Java 21.
- Maven project at the repository root.
- Unit tests enabled.

The shared repository owns the implementation.

If the platform team improves test reporting or adds a mandatory build validation step, it can release a new workflow version. Service teams can then migrate through reviewed pull requests rather than independently rebuilding the pipeline.

### 9.2 What this caller does not do

This CI caller does not:

- Authenticate to AWS.
- Publish to ECR.
- Update GitOps.
- Deploy to EKS.
- Approve production changes.

Those operations belong in separate workflows with their own permissions and environment protections.

## 10. Node.js reusable CI workflow

The Node.js workflow should use the lockfile committed by the service repository. Reproducible dependency installation is essential because changing dependency versions can alter test results or introduce vulnerable packages.

File: `.github/workflows/reusable-nodejs-ci.yml`

```
name: Reusable Node.js CI

on:
  workflow_call:
    inputs:
      node-version:
        type: string
        required: false
        default: "22"

      working-directory:
        type: string
        required: false
        default: "."

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}

    steps:
      - name: Checkout source
        uses: actions/checkout@<REVIEWED_FULL_COMMIT_SHA>

      - name: Set up Node.js
        uses: actions/setup-node@<REVIEWED_FULL_COMMIT_SHA>
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm
          cache-dependency-path: ${{ inputs.working-directory }}/package-lock.json

      - name: Validate lockfile
        run: test -f package-lock.json

      - name: Install locked dependencies
        run: npm ci

      - name: Run lint
        run: npm run lint --if-present

      - name: Run tests
        run: npm test --if-present

      - name: Build application
        run: npm run build --if-present
```

### Production improvements

The example intentionally uses `--if-present` to support different Node.js project scripts. However, this can allow a service with no test or build script to pass.

For a production release, the platform should maintain a service contract that identifies required scripts. The workflow must fail if a mandatory script is missing.

For example, the Order Service may require tests and a production build, while a small internal tooling package may have a different contract.

The platform should also:

- Require a lockfile.
- Use `npm ci` rather than an uncontrolled dependency update.
- Apply dependency scanning and license policy.
- Retain test results.
- Avoid exposing publishing credentials to pull requests from forks.
- Validate the resulting container image separately.

## 11. Python reusable CI workflow

File: `.github/workflows/reusable-python-ci.yml`

```
name: Reusable Python CI

on:
  workflow_call:
    inputs:
      python-version:
        type: string
        required: false
        default: "3.12"

      working-directory:
        type: string
        required: false
        default: "."

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}

    steps:
      - name: Checkout source
        uses: actions/checkout@<REVIEWED_FULL_COMMIT_SHA>

      - name: Set up Python
        uses: actions/setup-python@<REVIEWED_FULL_COMMIT_SHA>
        with:
          python-version: ${{ inputs.python-version }}

      - name: Install project dependencies
        run: |
          python -m pip install --upgrade pip
          test -f requirements.txt
          pip install -r requirements.txt

      - name: Run lint
        run: |
          pip install ruff
          ruff check .

      - name: Run tests
        run: |
          pip install pytest
          pytest -q
```

This is appropriate for a project using `requirements.txt` and pytest. If the service uses Poetry, uv, or another dependency manager, create a matching contract or an explicit supported mode rather than assuming all Python services share the same packaging structure.

In production, pin tooling versions and use the project's approved lockfile. Installing an unpinned linter or test runner during each build can introduce unexpected behavior.

## 12. Go reusable CI workflow

File: `.github/workflows/reusable-go-ci.yml`

```
name: Reusable Go CI

on:
  workflow_call:
    inputs:
      go-version:
        type: string
        required: false
        default: "1.24"

      working-directory:
        type: string
        required: false
        default: "."

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}

    steps:
      - name: Checkout source
        uses: actions/checkout@<REVIEWED_FULL_COMMIT_SHA>

      - name: Set up Go
        uses: actions/setup-go@<REVIEWED_FULL_COMMIT_SHA>
        with:
          go-version: ${{ inputs.go-version }}
          cache: true
          cache-dependency-path: ${{ inputs.working-directory }}/go.sum

      - name: Verify module files
        run: |
          test -f go.mod
          test -f go.sum

      - name: Verify formatting
        run: |
          test -z "$(gofmt -l .)"

      - name: Run tests
        run: go test ./...

      - name: Run vet
        run: go vet ./...

      - name: Build
        run: go build ./...
```

The Go version shown is illustrative. The platform must select versions supported by the project's actual toolchain and compatibility requirements.

## 13. Reusable code-quality workflow

The code-quality workflow standardizes static code analysis and the enforcement of the organization's configured quality gate.

It should not merely upload a report and then allow the pipeline to succeed regardless of the result.

File: `.github/workflows/reusable-code-quality.yml`

```
name: Reusable Code Quality

on:
  workflow_call:
    inputs:
      project-key:
        type: string
        required: true

      quality-gate-required:
        type: boolean
        required: false
        default: true

    secrets:
      SONAR_TOKEN:
        required: true

      SONAR_HOST_URL:
        required: true

permissions:
  contents: read

jobs:
  quality:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@<REVIEWED_FULL_COMMIT_SHA>
        with:
          fetch-depth: 0

      - name: Run SonarQube scanner
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
        run: |
          set -euo pipefail
          sonar-scanner \
            -Dsonar.projectKey="${{ inputs.project-key }}" \
            -Dsonar.host.url="$SONAR_HOST_URL"
```

### Limitations and required additions

This example invokes the scanner, but it does not independently prove that the server-side quality gate has passed.

A complete implementation must:

1. Verify the scan was accepted.
2. Obtain the quality-gate result from the approved SonarQube integration or API.
3. Fail if the gate fails or cannot be verified.
4. Fail on a timeout rather than waiting indefinitely.
5. Preserve a link to the analysis report.
6. Ensure the project key maps to the correct service.
7. Use a supported, controlled scanner installation method.
8. Avoid logging tokens.

The secret names are illustrative. The real deployment must match the configured SonarQube authentication and secret-delivery approach.

A workflow should not report “quality passed” merely because the scanner process exited successfully.

## 14. Reusable dependency scanning workflow

Dependency scanning identifies known vulnerabilities in libraries used by an application.

It should cover the package managers actually used by each service, including transitive dependencies where supported.

File: `.github/workflows/reusable-dependency-scan.yml`

```
name: Reusable Dependency Scan

on:
  workflow_call:
    inputs:
      working-directory:
        type: string
        required: false
        default: "."

      severity-threshold:
        type: string
        required: false
        default: "HIGH"

permissions:
  contents: read

jobs:
  scan:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}

    steps:
      - name: Checkout source
        uses: actions/checkout@<REVIEWED_FULL_COMMIT_SHA>

      - name: Check npm dependencies
        if: ${{ hashFiles(format('{0}/package-lock.json', inputs.working-directory)) != '' }}
        run: npm audit --audit-level="${{ inputs.severity-threshold }}"

      - name: Check Python dependency inventory
        if: ${{ hashFiles(format('{0}/requirements.txt', inputs.working-directory)) != '' }}
        run: |
          python -m pip install pip-audit
          pip-audit -r requirements.txt
```

This is an illustrative starting point, not a universal multi-language scanner. The caller must have the relevant runtime and package-manager prerequisites. Production implementations should separate language-specific scanning when that makes dependencies and reporting easier to manage.

The threshold must come from an approved policy. For example, a `HIGH` threshold is not automatically appropriate for every organization, and the exact interpretation of a tool's exit code must be verified.

### Handling a vulnerable dependency

When a scan fails:

1. Identify the package and affected version.
2. Confirm whether the finding affects the application.
3. Check the available fixed version or approved mitigation.
4. Update the dependency and lockfile.
5. Run unit and integration tests.
6. Rerun the security scan.
7. Record an exception only when policy allows one.
8. Ensure the exception has an owner and expiration date.

Do not suppress an entire scanner because one dependency has a difficult upgrade path.

## 15. Reusable secret scanning workflow

Secret scanning is different from dependency scanning. It looks for credentials or sensitive tokens exposed in code or files.

A secret scan should run on pull requests and on protected branches. Repository-level secret scanning may additionally inspect historical content, depending on the chosen product and configuration.

Example using Gitleaks:

```
name: Reusable Secret Scan

on:
  workflow_call:

permissions:
  contents: read

jobs:
  scan:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository history
        uses: actions/checkout@<REVIEWED_FULL_COMMIT_SHA>
        with:
          fetch-depth: 0

      - name: Scan for secrets
        uses: gitleaks/gitleaks-action@<REVIEWED_FULL_COMMIT_SHA>
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

This requires a reviewed Gitleaks action revision and verification of the action's current configuration requirements. The example is not a guarantee that every token pattern or historical secret will be detected.

### Secret scanning operational rules

- Never place real tokens in workflow YAML.
- Never echo secret values into logs.
- Never expose privileged credentials to untrusted pull-request code.
- Treat a confirmed leaked credential as compromised.
- Revoke or rotate the credential.
- Investigate its permissions and usage.
- Review logs for potential misuse.
- Add a narrowly scoped suppression only after review.
- Keep an audit record of the remediation.

Removing a secret from the latest commit does not revoke it. Credential rotation is still required.

## 16. Reusable SAST workflow

Static Application Security Testing (SAST) inspects source code for potential security weaknesses without requiring the deployed application to be running.

For example, it may detect unsafe input handling, injection risks, insecure cryptographic patterns or other language-specific issues, depending on the scanner and its rules.

The reusable SAST workflow should standardize:

- Scanner selection and approved versions.
- Language and repository detection.
- Configuration and rule sets.
- Report generation.
- Severity thresholds.
- Suppression review.
- Pull-request annotations where supported.
- Failure behavior.
- Evidence retention.

Example using a CLI-based scanner contract:

```
name: Reusable SAST

on:
  workflow_call:
    inputs:
      scan-config:
        type: string
        required: false
        default: ".security/sast-config.yml"

permissions:
  contents: read
  security-events: write

jobs:
  sast:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@<REVIEWED_FULL_COMMIT_SHA>

      - name: Verify approved scanner configuration
        run: |
          test -f "${{ inputs.scan-config }}"

      - name: Run approved SAST scanner
        run: |
          echo "Invoke the organization's approved SAST scanner here"
          echo "Configure a non-zero exit code for policy failures"
```

The scanner invocation is intentionally a placeholder. The actual command depends on the selected scanner, its authentication method and its report format. A placeholder cannot be treated as a functioning security gate.

For production, replace it with a pinned, tested scanner integration. Validate that:

1. The expected languages are analyzed.
2. Scan failures cause the intended workflow result.
3. Findings are not silently discarded.
4. Suppressions have reviewers.
5. Reports are available to authorized reviewers.
6. Forked pull requests cannot access privileged credentials.

## 17. Reusable container build workflow

The build workflow creates a container image from the approved source revision.

The container build should have a clear input contract:

- Source revision.
- Dockerfile path.
- Build context.
- Build arguments that are explicitly approved.
- Target platform or platforms.
- Image metadata.
- Build cache configuration.
- Output image or artifact reference.

Example:

```
name: Reusable Container Build

on:
  workflow_call:
    inputs:
      image-name:
        type: string
        required: true

      dockerfile:
        type: string
        required: false
        default: Dockerfile

      build-context:
        type: string
        required: false
        default: "."

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@<REVIEWED_FULL_COMMIT_SHA>

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@<REVIEWED_FULL_COMMIT_SHA>

      - name: Build image for validation
        uses: docker/build-push-action@<REVIEWED_FULL_COMMIT_SHA>
        with:
          context: ${{ inputs.build-context }}
          file: ${{ inputs.dockerfile }}
          push: false
          tags: ${{ inputs.image-name }}:${{ github.sha }}
```

The image name and build context must be validated against the service's approved configuration. Production workflows should also define how the built artifact is retained and handed to the scanning and publishing stages.

### Why rebuilding matters

Suppose a pipeline performs these operations:

1. Build image A.
2. Scan image A.
3. Build image B in another job.
4. Publish image B.

Even if both builds use the same source commit, image B is not automatically proven identical to image A. Different base-image resolution, dependency downloads, build inputs or timestamps may change the resulting artifact.

A production design should transfer the exact built image or use a controlled, reproducible build process with verification that proves the published digest matches the scanned digest.

## 18. Reusable container scan workflow

File: `.github/workflows/reusable-container-scan.yml`

```
name: Reusable Container Scan

on:
  workflow_call:
    inputs:
      image-reference:
        type: string
        required: true

      severity-threshold:
        type: string
        required: false
        default: HIGH

permissions:
  contents: read

jobs:
  scan:
    runs-on: ubuntu-latest

    steps:
      - name: Scan the supplied image reference
        uses: aquasecurity/trivy-action@<REVIEWED_FULL_COMMIT_SHA>
        with:
          image-ref: ${{ inputs.image-reference }}
          severity: ${{ inputs.severity-threshold }},CRITICAL
          ignore-unfixed: false
          exit-code: "1"
```

This example demonstrates the general contract; the action version, supported inputs and policy semantics must be verified against the reviewed action revision before deployment.

The workflow should scan an immutable reference where possible:

```
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:<VERIFIED_DIGEST>
```

The account ID and digest above are placeholders.

### Container scanning rules

A container scan should consider:

- OS packages.
- Application dependencies embedded in the image.
- Known vulnerabilities.
- Whether a fix is available.
- Approved exceptions.
- The actual runtime base image.
- Scanner database freshness.
- The image digest under review.

A scan against a mutable tag can produce ambiguity if that tag changes between the scan and release stages.

### Exception process

A vulnerability exception should record:

```
exception:
  id: "SEC-EXCEPTION-EXAMPLE"
  service: "order-service"
  finding: "EXAMPLE-CVE"
  severity: "HIGH"
  owner: "service-team"
  approver: "application-security"
  justification: "Document the verified business and technical reason"
  mitigation: "Document the compensating control"
  expires-on: "YYYY-MM-DD"
```

This is an example data structure, not a complete exception-management system. The actual process must validate expiry, verify approval and prevent an expired exception from continuing to authorize a release.

## 19. Reusable SBOM workflow

An SBOM, or Software Bill of Materials, provides a structured inventory of software components included in an artifact.

It can help answer questions such as:

- Which services contain an affected library?
- Which image versions contain a vulnerable OS package?
- Which release was produced from a particular dependency set?
- Which components need to be upgraded following a security advisory?

A practical SBOM workflow can use a scanner such as Syft, provided the organization approves and pins the selected tool version.

Example command:

```
syft "$IMAGE_REFERENCE" \
  -o cyclonedx-json=sbom.cdx.json
```

The workflow should then:

1. Verify the command succeeded.
2. Confirm the output file exists and is valid.
3. Associate the SBOM with the image digest.
4. Upload it as a build artifact or store it in an approved artifact system.
5. Apply retention and access controls.
6. Record its location in release evidence.
7. Regenerate or verify the SBOM when the artifact changes.

Do not attach an SBOM from one build to a different image merely because both have the same service name.

## 20. Reusable image verification workflow

This workflow checks that the image published to ECR is the same image that passed the required checks.

Its contract should require:

- Expected ECR repository.
- Expected image digest.
- Approved AWS account and region.
- Successful security scan evidence.
- Successful build status.
- Expected source revision.
- Optional signature and provenance verification.

Example verification logic:

```
set -euo pipefail

test -n "${EXPECTED_DIGEST:-}"
test -n "${EXPECTED_REPOSITORY:-}"

case "$EXPECTED_DIGEST" in
  sha256:*) ;;
  *)
    echo "Expected a SHA-256 image digest"
    exit 1
    ;;
esac

printf 'Expected repository: %s\n' "$EXPECTED_REPOSITORY"
printf 'Expected digest: %s\n' "$EXPECTED_DIGEST"
```

This is only the input-validation part of the workflow. It does not query ECR or verify the published artifact by itself.

A complete implementation must retrieve the registry manifest, verify the repository and digest, and compare them with the recorded build and scan evidence. If signing or provenance is required by policy, those checks must also succeed before release eligibility is granted.

### Example of the intended release record

```
service: order-service
source_commit: "<REVIEWED_SOURCE_COMMIT>"
image_repository: "order-service"
image_digest: "sha256:<VERIFIED_DIGEST>"
scan_status: "passed"
sbom_status: "generated"
signature_status: "verified"
provenance_status: "verified"
release_eligible: true
```

The values are illustrative placeholders. The record should be generated from verified job outputs and reports, not written manually to claim that checks passed.

## 21. Reusable ECR publishing workflow

The ECR publishing workflow authenticates to AWS and publishes the already approved image.

It should not rebuild the image if the platform's policy requires promotion of the exact scanned artifact.

### 21.1 AWS authentication using OIDC

Use GitHub's OpenID Connect federation with an AWS IAM role instead of storing long-lived AWS access keys in repository secrets.

The caller's permissions should be explicit:

```
permissions:
  contents: read
  id-token: write
```

The `id-token: write` permission allows the job to request a GitHub OIDC token. It does not, by itself, grant access to AWS. AWS must trust the intended identity and authorize the assumed role.

The IAM trust policy should restrict access using appropriate conditions for the GitHub organization, repository, branch or protected environment. Avoid a trust policy that accepts arbitrary repositories or untrusted branches.

### 21.2 ECR publish example

The following is an illustrative publishing workflow. It assumes the approved image has already been transferred into the runner or made available through a controlled artifact mechanism.

```
name: Reusable ECR Publish

on:
  workflow_call:
    inputs:
      aws-region:
        type: string
        required: true

      ecr-repository:
        type: string
        required: true

      image-tag:
        type: string
        required: true

    outputs:
      image-digest:
        description: Published image digest
        value: ${{ jobs.publish.outputs.image-digest }}

permissions:
  contents: read
  id-token: write

jobs:
  publish:
    runs-on: ubuntu-latest

    environment: ecr-publish

    outputs:
      image-digest: ${{ steps.publish.outputs.image-digest }}

    steps:
      - name: Checkout approved source
        uses: actions/checkout@<REVIEWED_FULL_COMMIT_SHA>

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@<REVIEWED_FULL_COMMIT_SHA>
        with:
          role-to-assume: ${{ vars.AWS_ECR_PUBLISH_ROLE_ARN }}
          aws-region: ${{ inputs.aws-region }}

      - name: Log in to ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@<REVIEWED_FULL_COMMIT_SHA>

      - name: Publish approved image
        id: publish
        env:
          REGISTRY: ${{ steps.ecr-login.outputs.registry }}
          REPOSITORY: ${{ inputs.ecr-repository }}
          IMAGE_TAG: ${{ inputs.image-tag }}
        run: |
          set -euo pipefail

          test -n "$REGISTRY"
          test -n "$REPOSITORY"
          test -n "$IMAGE_TAG"

          docker image inspect "$REPOSITORY:$IMAGE_TAG" >/dev/null

          docker tag \
            "$REPOSITORY:$IMAGE_TAG" \
            "$REGISTRY/$REPOSITORY:$IMAGE_TAG"

          docker push "$REGISTRY/$REPOSITORY:$IMAGE_TAG"

          DIGEST="$(aws ecr describe-images \
            --repository-name "$REPOSITORY" \
            --image-ids imageTag="$IMAGE_TAG" \
            --query 'imageDetails[0].imageDigest' \
            --output text)"

          case "$DIGEST" in
            sha256:*) ;;
            *)
              echo "Unable to obtain a valid ECR digest"
              exit 1
              ;;
          esac

          echo "image-digest=$DIGEST" >> "$GITHUB_OUTPUT"
```

Implementation warning: this example illustrates the OIDC, publishing and output contract. The image-tagging assumption must match the actual artifact-transfer design. It must not be interpreted as a complete solution for transferring an image between isolated jobs.

The production implementation must also ensure that:

- The image tag is unique to the approved build or otherwise protected against replacement.
- The image being pushed is the image that was scanned.
- The ECR role can publish only to approved repositories.
- The release identity cannot delete repositories or alter unrelated AWS infrastructure.
- The digest returned by ECR is independently verified.
- Required signing, provenance and SBOM evidence are associated with that digest.

### 21.3 Correct output chaining

GitHub reusable workflow outputs have three layers:

1. A step writes a value to `$GITHUB_OUTPUT`.
2. The job exposes that step output under `jobs.<job_id>.outputs`.
3. The reusable workflow exposes the job output under `on.workflow_call.outputs`.

The caller can then access the value through `needs.<job_id>.outputs.<output_name>`.

If any layer is missing, the caller may receive an empty value. Never promote an image when the required digest output is empty or invalid.

## 22. Terraform reusable workflows

Infrastructure changes need a different risk model from application CI.

An application build failure usually blocks a release. A Terraform mistake can modify IAM permissions, expose a network path, replace an EKS cluster or destroy persistent infrastructure.

### 22.1 Terraform validation workflow

File: `.github/workflows/reusable-terraform-validate.yml`

```
name: Reusable Terraform Validate

on:
  workflow_call:
    inputs:
      working-directory:
        type: string
        required: false
        default: "."

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}

    steps:
      - name: Checkout source
        uses: actions/checkout@<REVIEWED_FULL_COMMIT_SHA>

      - name: Set up Terraform
        uses: hashicorp/setup-terraform@<REVIEWED_FULL_COMMIT_SHA>
        with:
          terraform_wrapper: false

      - name: Check formatting
        run: terraform fmt -check -recursive

      - name: Initialize without a backend
        run: terraform init -backend=false -input=false

      - name: Validate configuration
        run: terraform validate
```

The backend-free initialization is useful for validation that does not need to access the real remote state. It is not a replacement for a real plan against the intended backend.

### 22.2 Terraform security workflow

The security workflow should scan for risks such as:

- Publicly exposed storage.
- Unrestricted ingress.
- Overly broad IAM policies.
- Missing encryption.
- Missing or inappropriate logging settings.
- Unsafe EKS configuration.
- Unprotected state storage.
- Public database exposure.

A tool such as Checkov can be used when approved by the platform team. Its rules and exit behavior must be tested against the organization's actual Terraform patterns.

### 22.3 Terraform plan workflow

A plan workflow should:

1. Authenticate using a restricted identity.
2. Access the intended backend and workspace or state context.
3. Use a pinned Terraform version.
4. Generate a plan for the intended environment.
5. Retain the plan and a readable summary.
6. Identify destructive or replacement changes.
7. Require review and approval according to risk.
8. Ensure the apply process uses the reviewed plan where applicable.
9. Record the resulting infrastructure change.

A plan generated for one commit or state must not be assumed valid for a later commit or changed infrastructure state.

Never automatically apply arbitrary Terraform changes from untrusted pull requests.

## 23. Reusable Helm validation workflow

Helm charts are the delivery interface between the platform and Kubernetes workloads. The validation workflow should check both the chart structure and the manifests produced from its values.

Example:

```
name: Reusable Helm Validate

on:
  workflow_call:
    inputs:
      chart-directory:
        type: string
        required: true

      values-file:
        type: string
        required: false
        default: "values.yaml"

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@<REVIEWED_FULL_COMMIT_SHA>

      - name: Lint Helm chart
        run: |
          helm lint \
            "${{ inputs.chart-directory }}" \
            -f "${{ inputs.chart-directory }}/${{ inputs.values-file }}"

      - name: Render manifests
        run: |
          helm template validation \
            "${{ inputs.chart-directory }}" \
            -f "${{ inputs.chart-directory }}/${{ inputs.values-file }}" \
            > rendered.yaml

      - name: Verify rendered output exists
        run: test -s rendered.yaml
```

A complete workflow must use an approved Helm installation method and pin the tool version.

It should also validate environment-specific values and rendered output against the organization's Kubernetes policy. A chart can pass `helm lint` while still rendering a privileged container or a workload without resource limits.

### Example workload policy checks

The rendered manifests should be evaluated for:

- Non-root execution where compatible.
- Read-only root filesystem where practical.
- Dropped Linux capabilities.
- Appropriate seccomp settings.
- Resource requests and limits.
- Approved image registries and immutable image references.
- Appropriate ServiceAccount usage.
- Required health probes.
- Approved ingress and network exposure.
- Required PodDisruptionBudget or autoscaling settings where applicable.

These controls should be introduced through the relevant platform templates and policy engine, with explicit handling for legitimate exceptions.

## 24. Reusable Kubernetes policy workflow

The Kubernetes policy workflow should validate rendered Helm manifests before promotion.

For example, a Conftest/OPA policy suite can check whether a deployment uses an approved registry and has resource limits. An admission controller can enforce related policies in the cluster at deployment time.

These are separate controls:

- CI policy check: catches violations before the release is promoted.
- Admission policy: rejects noncompliant resources when they are submitted to the Kubernetes API.
- Runtime monitoring: detects conditions that occur after deployment.

No single layer replaces the others.

A simple conceptual policy is:

```
package kubernetes.resources

deny contains msg if {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  not container.resources.limits
  msg := sprintf(
    "Container %s must define resource limits",
    [container.name]
  )
}
```

This is a policy example, not a complete policy library. It must be tested with valid and invalid fixtures, integrated with the selected policy engine and adapted for the exact resource formats being checked.

A production workflow should:

1. Render all relevant Helm values.
2. Parse the generated YAML.
3. Run the approved policy bundle.
4. Fail on unapproved violations.
5. Publish a report.
6. Record any valid exception.
7. Ensure cluster admission controls remain enabled for required environments.

## 25. API contract and integration test workflows

Microservices can pass their own unit tests but still fail when integrated with another service.

The Order Service may compile successfully while calling a Payment Service endpoint that has changed its response schema. The Cart Service may produce an event that the Order Service no longer understands.

### API contract testing

A contract workflow should validate:

- Request and response schemas.
- Required and optional fields.
- Status codes and error responses.
- Authentication expectations.
- Compatibility with supported consumers.
- Event schemas, where applicable.

For a breaking API change, the workflow should fail until the affected consumer and provider changes are coordinated or a reviewed compatibility exception is approved.

### Integration testing

Integration tests should run against controlled dependencies. These may include:

- A database instance.
- A message broker.
- A dependent service.
- A test identity provider.
- A local or isolated infrastructure environment.

Do not connect pull-request tests to production databases or production message brokers.

Integration environments must use synthetic or approved test data, restricted credentials and predictable cleanup.

### Event-driven contracts

For Kafka or RabbitMQ integrations, tests should validate:

- Event schema and version.
- Producer and consumer compatibility.
- Retry behavior.
- Idempotency.
- Dead-letter behavior where configured.
- Duplicate delivery handling.
- Timeout and failure behavior.

Passing a schema test does not prove end-to-end delivery or business correctness, so retain integration tests for the critical transaction paths.

## 26. Reusable performance testing workflow

Performance testing should be run against an explicitly designated test environment, not automatically against production.

The workflow should accept controlled parameters such as:

- Target environment.
- Authorized base URL.
- Test scenario.
- Maximum duration.
- Virtual-user or arrival-rate configuration.
- Latency thresholds.
- Error-rate thresholds.
- Approved test window.

For example, a k6 test might be executed as follows:

```
k6 run \
  --summary-export=performance-summary.json \
  tests/performance/order-flow.js
```

The test script must contain thresholds and checks appropriate for the application. The workflow should fail when a required threshold is breached.

The platform team should not allow a pull request to provide an arbitrary URL and then execute unrestricted load against that target. Target allowlists, environment approval and bounded test configuration are necessary.

### Example performance policy

An initial policy might require:

- No unexpected increase in HTTP 5xx responses.
- Error rate below an agreed threshold.
- P95 latency within the service's approved SLO.
- P99 latency within a separately defined limit.
- No unexpected resource saturation.
- A documented baseline for comparison.

The numeric thresholds must come from service requirements and observed baselines. They should not be invented once and applied to all 21 microservices.

## 27. Reusable release eligibility workflow

The release workflow decides whether a particular build is eligible for promotion. It does not itself prove that a production deployment is healthy.

A release eligibility contract should require evidence for:

```
[ ] Source revision identified
[ ] Required unit tests passed
[ ] Required integration tests passed
[ ] Code-quality gate passed
[ ] Required dependency scans passed
[ ] Required secret and SAST scans passed
[ ] Container scan passed
[ ] Exact image digest verified
[ ] SBOM generated
[ ] Signature verified, if required
[ ] Provenance verified, if required
[ ] Required approvals present
[ ] No expired security exceptions
[ ] Release metadata complete
```

The implementation should obtain these statuses from verified workflow results or artifact attestations. A caller should not be able to set an input such as `security-passed: true` and thereby bypass the actual security workflow.

A useful release record includes the service name, source commit, image digest, workflow run, scan evidence, approval references and destination environment.

## 28. Reusable GitOps promotion workflow

The GitOps promotion workflow changes the declared deployment state. It should not use direct production `kubectl apply` as a shortcut around the GitOps process.

The intended sequence is:

1. Verify release eligibility.
2. Retrieve the verified image digest.
3. Identify the target GitOps application and environment.
4. Confirm the service is allowed to deploy to that environment.
5. Update the appropriate image digest in a branch.
6. Render and validate the resulting manifests.
7. Open a pull request.
8. Obtain required approvals.
9. Merge according to repository protection rules.
10. Allow Argo CD to reconcile the desired state.
11. Verify the resulting deployment.

Example of the desired GitOps configuration:

```
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  digest: "sha256:<VERIFIED_DIGEST>"
```

The account ID and digest are placeholders. The real digest must come from the verified publishing workflow output.

### Promotion between environments

A typical path is:

```
Verified artifact
       |
       v
DEV GitOps revision
       |
       v
DEV verification
       |
       v
QA GitOps revision
       |
       v
QA verification
       |
       v
STAGING GitOps revision
       |
       v
Production change approval
       |
       v
Production GitOps revision
```

The exact same image digest should be promoted unless an explicitly governed process produces a new artifact and repeats the required verification. Environment-specific configuration can change, but the application artifact should not silently change between stages.

### Permissions

The workflow should have only the permissions needed to create or update the approved GitOps change. Production approval should be governed through protected branches, protected environments or equivalent controls.

The application CI workflow should not have unrestricted write access to the production GitOps branch.

## 29. Reusable deployment verification workflow

Deployment verification checks the state of a deployment after Argo CD reconciles the GitOps revision.

The workflow may check:

- Expected GitOps revision.
- Argo CD application sync and health state.
- Kubernetes rollout status.
- Desired and available replicas.
- Readiness and liveness probe behavior.
- Service endpoint health.
- Critical synthetic transaction results.
- Error-rate and latency signals.
- Deployment-specific alerts.

Example rollout command:

```
kubectl rollout status \
  deployment/order-service \
  --namespace=ecommerce \
  --timeout=180s
```

The workflow must authenticate to the intended cluster through an approved identity and should only be able to inspect the required resources.

### Rollout status is not application health

A successful Kubernetes rollout indicates that Kubernetes observed the deployment reaching its rollout completion criteria. It does not guarantee that the application's business logic is functioning correctly.

For example, Order Service pods may be ready while every checkout request fails because Payment Service is unavailable.

Therefore, deployment verification should include application-level checks for critical services.

## 30. Reusable rollback validation workflow

Rollback validation verifies that a known-good release can be restored safely.

It should establish:

1. Which previous release is considered known-good.
2. Whether its image digest remains available.
3. Whether the old configuration remains compatible.
4. Whether database changes are backward compatible.
5. Whether event schemas remain compatible.
6. Whether the GitOps revision can be reverted.
7. Whether the rollback has the required approval.
8. How health will be verified after restoration.

### Database rollback warning

Rolling an application image backward does not automatically reverse a database schema migration.

For example, if a new version deletes or renames a database column, an older version may fail after the image is rolled back.

Use backward-compatible schema evolution, such as expand-and-contract migrations, for changes that need safe rolling deployment and rollback.

A rollback workflow should not automatically execute destructive database operations just because the application deployment failed.

## 31. Reusable OWASP DAST workflow

Dynamic Application Security Testing (DAST) checks a running application from the outside. It complements SAST, dependency scanning and container scanning.

It should run against a controlled QA or staging endpoint that has explicit authorization for testing.

Example command using an approved OWASP ZAP baseline configuration:

```
docker run --rm \
  -t \
  -v "$PWD:/zap/wrk/:rw" \
  ghcr.io/zaproxy/zaproxy:<APPROVED_PINNED_VERSION> \
  zap-baseline.py \
  -t "$TARGET_URL" \
  -r zap-report.html
```

This command is illustrative. The container image should be pinned to a reviewed version or digest, and the organization should verify the scanner's current command-line behavior before adopting it.

### DAST safety controls

- Allow only approved target hosts.
- Restrict testing to designated environments.
- Avoid destructive test profiles against shared environments.
- Use test accounts and synthetic data.
- Limit test duration and request rates.
- Protect authentication tokens.
- Preserve reports for authorized reviewers.
- Define which findings block a release.
- Require approval for more intrusive testing.

A baseline scan is not proof that an application is free from vulnerabilities. It covers only the checks performed under the configured profile.

## 32. Reusable license compliance workflow

Open-source components may introduce licensing obligations as well as security risks.

The workflow should:

1. Detect dependencies and licenses using an approved tool.
2. Compare findings against the organization's license policy.
3. Identify unknown or unrecognized licenses.
4. Produce a report.
5. Fail on prohibited licenses where policy requires.
6. Route ambiguous findings to the designated legal or compliance owner.
7. Support documented, time-limited exceptions.
8. Retain evidence with the release.

Do not automatically declare a dependency prohibited solely because a scanner cannot identify its license. Unknown findings should be handled through the approved review process.

## 33. Reusable evidence archive workflow

The evidence workflow collects records that help the team prove what was built, checked and released.

Evidence can include:

- Source commit SHA.
- Workflow run ID and revision.
- Unit and integration test reports.
- Code-quality results.
- Dependency and secret scan reports.
- Container scan results.
- Image digest.
- SBOM.
- Signature verification results.
- Provenance or build attestations.
- Release eligibility decision.
- GitOps pull request and merged commit.
- Environment approvals.
- Deployment verification results.

### Evidence integrity

Evidence should be tied to immutable identifiers wherever possible.

For example, a report labelled only `order-service-latest-scan.json` can become ambiguous if it is overwritten. A report associated with the source revision, workflow run and exact image digest is easier to audit.

Evidence storage must also enforce appropriate access controls and retention. Logs and reports may contain sensitive repository details or vulnerability information, so broad public access is not appropriate.

## 34. Standardizing workflow inputs across 21 microservices

The platform should maintain a service inventory that records each service's actual build and release requirements.

An illustrative inventory:

| Service                | Runtime                | CI workflow                | Release artifact |
| ---------------------- | ---------------------- | -------------------------- | ---------------- |
| Auth Service           | Confirm actual runtime | Matching language workflow | Container image  |
| User Service           | Confirm actual runtime | Matching language workflow | Container image  |
| Customer Service       | Confirm actual runtime | Matching language workflow | Container image  |
| Catalog Service        | Confirm actual runtime | Matching language workflow | Container image  |
| Product Service        | Confirm actual runtime | Matching language workflow | Container image  |
| Inventory Service      | Confirm actual runtime | Matching language workflow | Container image  |
| Pricing Service        | Confirm actual runtime | Matching language workflow | Container image  |
| Promotion Service      | Confirm actual runtime | Matching language workflow | Container image  |
| Cart Service           | Confirm actual runtime | Matching language workflow | Container image  |
| Order Service          | Confirm actual runtime | Matching language workflow | Container image  |
| Payment Service        | Confirm actual runtime | Matching language workflow | Container image  |
| Fraud Service          | Confirm actual runtime | Matching language workflow | Container image  |
| Shipping Service       | Confirm actual runtime | Matching language workflow | Container image  |
| Delivery Service       | Confirm actual runtime | Matching language workflow | Container image  |
| Notification Service   | Confirm actual runtime | Matching language workflow | Container image  |
| Search Service         | Confirm actual runtime | Matching language workflow | Container image  |
| Recommendation Service | Confirm actual runtime | Matching language workflow | Container image  |
| Review Service         | Confirm actual runtime | Matching language workflow | Container image  |
| Tax Service            | Confirm actual runtime | Matching language workflow | Container image  |
| Invoice Service        | Confirm actual runtime | Matching language workflow | Container image  |
| Reporting Service      | Confirm actual runtime | Matching language workflow | Container image  |

Do not assume every service uses Java, Node.js, Python or Go simply because those language workflows exist. Populate the inventory from the real repositories and verify the mapping during onboarding.

### Service contract metadata

A service may publish metadata describing its requirements:

```
service:
  name: order-service
  owner: commerce-orders
  runtime: java
  runtime-version: "21"
  package-manager: maven
  container-dockerfile: Dockerfile
  helm-chart: deploy/chart
  gitops-path: apps/order-service
  required-gates:
    - unit-tests
    - code-quality
    - dependency-scan
    - secret-scan
    - container-scan
```

This is an example contract. The actual service inventory and approved policy should validate allowed values and mandatory fields before a release.

## 35. Required versus optional gates

A shared workflow must distinguish between a gate that is mandatory and a check that is informational.

| Check                  | Pull request                                    | Release candidate                      | Production promotion                      |
| ---------------------- | ----------------------------------------------- | -------------------------------------- | ----------------------------------------- |
| Compile/build          | Required                                        | Required                               | Required evidence                         |
| Unit tests             | Required                                        | Required                               | Required evidence                         |
| Integration tests      | Based on service policy                         | Required for designated critical paths | Required evidence                         |
| Code-quality gate      | Organization policy                             | Required where mandated                | Must have passed                          |
| Secret scanning        | Required                                        | Required                               | Must have passed                          |
| Dependency scanning    | Required                                        | Required                               | Must satisfy release policy               |
| Container scanning     | May scan PR image                               | Required for release image             | Must match promoted digest                |
| SBOM                   | Optional early in development if policy permits | Required under release policy          | Must match artifact                       |
| Image signing          | Usually release-stage                           | Required if policy mandates            | Verify before promotion                   |
| Terraform plan review  | For infrastructure PRs                          | Required for infrastructure changes    | Required before apply                     |
| Helm/Kubernetes policy | Required before promotion                       | Required                               | Required evidence                         |
| DAST                   | Usually not every PR                            | Required for designated releases       | Must satisfy policy                       |
| Production approval    | Not applicable                                  | Not applicable                         | Required for protected production changes |

This table is a proposed baseline, not a claim that every check is mandatory in every organization. The security owner must define the actual gates based on risk and regulatory requirements.

### Avoiding the optional-gate bypass

Consider this workflow input:

```
with:
  run-tests: false
```

If the workflow permits this input, a caller can potentially disable tests.

For a release workflow, the platform must either remove that input, validate it against a trusted service policy or prevent the resulting build from being marked release-eligible.

The caller must not be allowed to downgrade a mandatory control by changing a YAML value in an ordinary service pull request.

## 36. GitHub permissions and workflow security

Every workflow should declare its required permissions explicitly.

For example:

```
permissions:
  contents: read
```

This is a reasonable baseline for a workflow that checks out source code but does not publish packages or change repositories.

A publishing workflow may require:

```
permissions:
  contents: read
  id-token: write
```

A workflow that publishes release evidence or updates another repository may need additional permissions, but those should be granted only to the job that needs them.

### Security rules

1. Do not give every workflow write access to the repository.
2. Do not expose AWS publishing credentials to untrusted pull-request code.
3. Do not use `pull_request_target` to run untrusted code with privileged secrets.
4. Pin third-party actions to reviewed full commit SHAs.
5. Restrict reusable workflow references to trusted revisions.
6. Protect release branches and production environments.
7. Keep AWS IAM trust conditions narrow.
8. Validate all external inputs.
9. Avoid executing shell commands constructed from untrusted input.
10. Prevent build logs from exposing secrets.
11. Limit artifacts and caches to appropriate trust boundaries.
12. Review workflow changes as security-sensitive code.

### Why `pull_request_target` needs special care

`pull_request_target` runs in the context of the base repository. A workflow that checks out and executes code from an untrusted pull request can create a dangerous combination of privileged context and attacker-controlled code.

For ordinary CI, prefer the appropriate pull-request trigger and do not provide privileged secrets to untrusted code. If a privileged workflow genuinely requires a special trigger, isolate its actions and ensure it does not execute untrusted content.

## 37. Workflow concurrency and duplicate runs

When a developer pushes multiple commits to the same pull request, GitHub Actions may start multiple workflow runs.

For expensive CI jobs, a concurrency group can cancel outdated runs:

```
concurrency:
  group: ci-${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
```

This is useful for ordinary pull-request validation, where a newer commit usually supersedes an older commit.

However, do not use the same cancellation policy indiscriminately for production deployment jobs. Cancelling a deployment during a critical transition may leave an environment in an unexpected state.

Use separate concurrency policies for:

- Pull-request CI.
- Release artifact publication.
- GitOps promotion.
- Infrastructure changes.
- Production deployment.
- Recovery and rollback operations.

For production deployment, use an explicit environment-level concurrency policy and a recovery procedure for interrupted runs.

## 38. Cache management

Caching can improve build performance, but it introduces correctness and trust considerations.

### Appropriate cache candidates

- Maven dependencies.
- npm package cache.
- Python package downloads.
- Go module downloads.
- Docker build layers, when configured safely.

### Cache rules

- Use lockfiles or dependency definitions to scope cache keys.
- Do not treat a cache as an authoritative artifact.
- Do not share privileged build outputs across incompatible trust boundaries.
- Do not rely on a cache to prove which dependencies were used.
- Keep credentials out of cached content.
- Rebuild when a cache is invalid or untrusted.
- Monitor cache size and restore performance.

A cache hit should make a build faster, not change what the build is supposed to produce.

## 39. Testing reusable workflows before release

Reusable workflows are code. They need their own tests and release criteria.

### 39.1 Static validation

Validate workflow YAML, required input definitions, expressions, permissions and output declarations.

Use a maintained workflow linter or equivalent automated validation. The chosen tool should be pinned and tested in the platform repository.

### 39.2 Contract tests

For every workflow, test:

- Valid required inputs.
- Missing required inputs.
- Invalid input values.
- Expected output names.
- Missing output behavior.
- Correct failure propagation.
- Permission declarations.
- Artifact naming.
- Error reporting.

### 39.3 Integration tests

Run representative workflows in a test repository or isolated environment.

For example, test the Java workflow against:

1. A valid Maven project.
2. A project with a compilation error.
3. A project with a failing unit test.
4. A project missing its required wrapper.
5. A project with an invalid workflow configuration.

Expected outcomes should be explicit. A failing test fixture must cause the correct job and overall workflow to fail.

### 39.4 Negative security tests

A security workflow needs negative tests proving that it actually blocks the intended violations.

Examples:

- A fixture with a deliberately invalid manifest must fail the policy check.
- A test image containing a controlled vulnerable package must produce the expected finding when the scanner database and rule set support it.
- A Terraform fixture with a forbidden configuration must fail.
- An expired exception must no longer authorize a release.
- A workflow with missing required evidence must fail release eligibility.
- A publishing job with no valid digest must not publish or promote an artifact.

Never commit a real secret to test secret scanning. Use harmless synthetic fixtures recognized by the scanner.

## 40. Troubleshooting runbooks

### 40.1 Reusable workflow not found

Symptoms

- GitHub reports that a workflow cannot be found.
- The calling job cannot resolve its `uses` reference.

Checks

1. Confirm the repository owner and repository name.
2. Confirm the workflow path begins with `.github/workflows/`.
3. Confirm the workflow file exists on the referenced revision.
4. Confirm the target workflow uses `workflow_call`.
5. Confirm the caller is authorized to access the shared repository.
6. Confirm the branch, tag or commit reference is correct.

Resolution

Correct the workflow path or reference and rerun CI. Do not solve the problem by granting broad repository access without reviewing the intended trust boundary.

### 40.2 Required input missing

Symptoms

The reusable workflow fails validation or receives an empty value.

Checks

- Is the input defined under `on.workflow_call.inputs`?
- Is it marked `required: true`?
- Does the caller supply the correct input name?
- Is the type correct?
- Is the value valid for the target service?

Resolution

Fix the workflow contract or caller. Do not invent a fallback that disables a mandatory check.

### 40.3 Workflow output is empty

Symptoms

The publishing job completes, but the caller cannot obtain the image digest.

Checks

- Did the producing step write to `$GITHUB_OUTPUT`?
- Does the job expose that step output?
- Does `workflow_call.outputs` reference the job output?
- Is the caller referencing the correct job ID?
- Did the producer actually obtain the digest?

Resolution

Repair the output chain and add a validation step that fails when the digest is missing. Never substitute a mutable tag to make the pipeline continue.

### 40.4 AWS OIDC authentication fails

Symptoms

The AWS credentials action cannot assume the configured role.

Checks

- Does the workflow request `id-token: write`?
- Is the IAM role ARN correct?
- Does the IAM trust policy trust the intended GitHub OIDC identity?
- Do the repository and environment claims match the trust conditions?
- Is the workflow running in the expected branch or protected environment?
- Does the role have the required ECR permissions?

Resolution

Correct the identity or policy mismatch. Do not replace OIDC with a permanent administrator access key as a shortcut.

### 40.5 ECR publishing succeeds but release verification fails

Symptoms

The image appears in ECR, but the digest does not match the expected artifact.

Checks

- Was the exact scanned artifact transferred to the publishing job?
- Was the image rebuilt?
- Did a mutable tag get overwritten?
- Is the correct ECR repository being queried?
- Is the digest being extracted from the expected image manifest?
- Does the evidence reference the same source revision and digest?

Resolution

Stop promotion. Re-establish the artifact chain and verify the digest. A successful push does not override a failed identity check.

### 40.6 Security scan fails unexpectedly

Symptoms

The workflow fails on a finding the service team believes is irrelevant.

Checks

- Is the scanner version and vulnerability database current?
- Is the affected component actually present?
- Does the policy threshold apply to this stage?
- Is the finding a confirmed false positive?
- Is there an approved exception?
- Is the exception still valid?

Resolution

Investigate and document the finding. Update the dependency or create a narrowly scoped, approved, time-limited exception where policy allows. Never disable all security checks to release one service.

### 40.7 Terraform plan is unexpectedly destructive

Symptoms

The plan shows resource deletion or replacement that was not expected.

Response

1. Stop the apply process.
2. Verify the account, region, backend and workspace.
3. Inspect the plan output.
4. Compare the resource addresses and lifecycle changes.
5. Confirm provider and module versions.
6. Review recent configuration and state changes.
7. Involve the infrastructure owner.
8. Regenerate the plan after the cause is understood.

Never approve a destructive plan merely because CI completed successfully.

### 40.8 GitOps promotion does not deploy

Symptoms

The GitOps pull request is merged, but the EKS workload remains on the old version.

Checks

- Did the intended GitOps commit merge?
- Does Argo CD observe the correct repository and revision?
- Is the image digest present in the rendered manifests?
- Can the cluster pull the image?
- Is the application synchronized and healthy?
- Did admission policy reject the resource?
- Are the pods failing to start or become ready?

Resolution

Identify the failed reconciliation or deployment stage. Fix the actual cause and allow the approved GitOps process to reconcile again. Do not bypass the workflow by manually applying a different image to production.

## 41. Production governance and ownership

Reusable workflows need clear ownership because a failure in one shared workflow can affect many services simultaneously.

| Responsibility                                            | Primary owner                                 |
| --------------------------------------------------------- | --------------------------------------------- |
| Application unit tests and service-specific test coverage | Application team                              |
| Shared CI workflow implementation                         | Platform engineering                          |
| Security thresholds and approved exceptions               | Application security / security engineering   |
| IAM roles and AWS trust policies                          | Cloud platform / cloud security               |
| Terraform modules and infrastructure plans                | Infrastructure platform team                  |
| Helm chart and Kubernetes policy validation               | Kubernetes platform team                      |
| GitOps repository protection                              | Platform engineering and release governance   |
| Production change approval                                | Authorized release approver                   |
| Post-deployment verification                              | Service owner and platform/SRE functions      |
| Root-cause analysis                                       | Incident owner and relevant engineering teams |

The same person may hold multiple responsibilities in a smaller organization, but the controls should still distinguish implementation, approval and verification where separation of duties is required.

## 42. Workflow observability and operational metrics

The platform team should measure whether reusable workflows are reliable and useful.

Recommended metrics include:

- Workflow success rate.
- Failure rate by workflow version.
- Median and P95 workflow duration.
- Time spent waiting for runners.
- Cache hit rate.
- Frequency of flaky tests.
- Security findings by severity and age.
- Number of expired exceptions.
- Image build and scan duration.
- Number of releases blocked by missing evidence.
- Number of failed GitOps promotions.
- Number of production deployments requiring rollback.

### Example operational review

Suppose a new version of `reusable-container-scan.yml` causes 12 of the 21 microservices to fail.

The platform team should:

1. Identify the shared workflow version used by the failing services.
2. Compare results with the previous version.
3. Determine whether the change exposed real vulnerabilities or introduced a configuration error.
4. Stop broad rollout if the release risk warrants it.
5. Test the fix against representative services.
6. Release a corrected revision.
7. Upgrade affected callers through the approved process.
8. Document the incident and any necessary preventive action.

A shared workflow should not be changed in production merely to turn red checks green. The cause must be understood.

## 43. Workflow migration strategy

If services currently use Jenkins or GitLab CI/CD, migration should happen incrementally.

### Stage 1 — Inventory

Record:

- Existing pipeline files.
- Build tools and runtime versions.
- Required secrets and integrations.
- Existing test and security stages.
- Artifact destinations.
- Deployment permissions.
- Manual approvals.
- Service owners.

### Stage 2 — Map current controls

For each existing pipeline stage, identify its equivalent reusable workflow or document the reason for a difference.

Do not remove a security check just because the new platform uses a different tool.

### Stage 3 — Pilot migration

Choose a representative service and migrate its validation pipeline first.

Verify that:

- The new pipeline builds the same source.
- Tests produce comparable results.
- Required security checks still run.
- Artifacts are identified consistently.
- Permissions are restricted.
- Failures are reported clearly.

### Stage 4 — Parallel verification

Where feasible, compare the old and new pipelines for a controlled period. Confirm that discrepancies are understood before relying on the new pipeline.

### Stage 5 — Release and deployment migration

Migrate artifact publication and GitOps promotion only after the artifact identity, security gates and permissions have been validated.

### Stage 6 — Retire old pipelines

Remove obsolete jobs, credentials and runners only after confirming that the replacement workflow is operational and the rollback plan is adequate.

## 44. Practical implementation exercise

Implement the shared workflow platform for the Order Service first, then reuse it for the other services.

### Exercise A — Shared repository

- Create `company-ecommerce/platform-github-actions`.
- Protect its default branch.
- Add required code owners and review rules.
- Create the workflow catalog and contracts.
- Add `reusable-java-ci.yml`.
- Add `reusable-code-quality.yml`.
- Add `reusable-dependency-scan.yml`.
- Add `reusable-secret-scan.yml`.
- Add `reusable-container-build.yml`.
- Add `reusable-container-scan.yml`.
- Add `reusable-ecr-publish.yml`.
- Add `reusable-image-verify.yml`.
- Add the required test fixtures.

### Exercise B — Order Service integration

- Create the caller workflow in the Order Service repository.
- Pin shared workflow references to reviewed revisions.
- Confirm the correct Java version.
- Run unit tests.
- Verify the code-quality gate.
- Run dependency and secret scans.
- Build the container image.
- Scan the exact image.
- Publish the approved image to ECR using OIDC.
- Verify the ECR digest.
- Generate and associate the SBOM.
- Record the release evidence.

### Exercise C — GitOps integration

- Update the image digest through a GitOps pull request.
- Validate the Helm chart and rendered manifests.
- Run Kubernetes policy checks.
- Merge the approved environment change.
- Verify Argo CD reconciliation.
- Verify Kubernetes rollout status.
- Verify an application-level health check.
- Record the deployed digest and GitOps revision.

### Exercise D — Failure simulations

Test these scenarios in a safe development environment:

1. A unit test fails.
2. A required dependency scan fails.
3. The container scan fails.
4. The image digest is missing.
5. AWS OIDC authentication fails.
6. An unauthorized repository attempts to publish.
7. A Helm manifest violates a policy.
8. The GitOps promotion fails.
9. A deployment does not become ready.
10. A rollback target cannot be verified.

For each scenario, record the expected result, observed result, diagnostic evidence and corrective action.

## 45. Final production readiness checklist

### Shared workflow engineering

- Every workflow has an owner.
- Inputs and outputs are documented.
- Required permissions are explicit.
- Third-party actions are pinned to reviewed revisions.
- Shared workflow versions are controlled.
- Contract tests cover successful and failing paths.
- Breaking changes follow a migration process.
- Failures provide actionable diagnostic information.

### Security and supply chain

- Source and dependency checks run according to policy.
- Secrets are not exposed to untrusted code.
- Container scans evaluate the intended artifact.
- Vulnerability exceptions are scoped and time-limited.
- SBOMs are associated with the correct digest.
- Signing and provenance controls are verified where required.
- AWS authentication uses restricted identities.
- ECR permissions are limited to approved repositories.
- Production approval cannot be bypassed by a caller input.

### Artifact and deployment integrity

- The scanned image is the published image.
- The published digest is verified.
- GitOps promotion uses the verified digest.
- Helm and Kubernetes policies run before promotion.
- Argo CD performs the approved reconciliation.
- Deployment verification includes application health.
- Rollback procedures account for database compatibility.
- Release evidence is retained.

### Platform operations

- Workflow duration and failure rates are monitored.
- Shared workflow upgrades are tested before rollout.
- There is a procedure for a broken shared workflow.
- The service inventory identifies each service's actual runtime.
- The team can identify every service using a given workflow version.
- Migration and deprecation plans are documented.

## 46. Definition of Done

Chapter 185's implementation is complete when the team can demonstrate the following end-to-end path for a real service:

```
Source change
    |
    v
Reusable CI workflow
    |
    v
Tests and quality gates
    |
    v
Security workflows
    |
    v
Container build and scan
    |
    v
Verified image digest
    |
    v
ECR publication
    |
    v
Release evidence
    |
    v
GitOps pull request
    |
    v
Approved environment promotion
    |
    v
Argo CD reconciliation
    |
    v
Deployment and application verification
```

The team must also demonstrate that a failed required security check, a missing digest or an unauthorized publication attempt blocks the release.

Chapter 185's central principle: reusable workflows standardize how engineering work is executed. The security policies in Chapter 186 will define which controls are mandatory, how violations are handled and how those controls are enforced consistently across all 21 microservices.

Next: Chapter 186 — Security Policy Templates.

Length note: This is the expanded Chapter 185 implementation material, but I cannot honestly claim it has reached 2,000 Markdown lines. The remaining gap should be filled with additional implementation and test sections rather than artificial repetition.