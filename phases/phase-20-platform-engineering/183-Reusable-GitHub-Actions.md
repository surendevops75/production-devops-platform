# PHASE 20 — PLATFORM ENGINEERING

# Chapter 183 — Reusable GitHub Actions

Production implementation guide · GitHub Actions · AWS ECR · Docker · Kubernetes · Helm · GitOps

Chapter 182 established the reusable Helm chart that standardizes how our 21 microservices are deployed. This chapter builds the reusable CI components that standardize how those services are tested, scanned, built, and published.

The central design principle is build once, verify the artifact, and promote the same image digest through environments.

We will not create 21 completely independent CI pipelines with duplicated logic. Instead, we will maintain reusable workflows and composite actions in a platform repository, while each application repository retains its own build configuration and business tests.

## 1. What we are building

By the end of this chapter, you will have a practical reusable CI architecture that supports:

- Java/Spring Boot applications built with Maven.
- Node.js applications.
- Python applications.
- Go applications.
- Unit and integration testing.
- Code-quality checks and SonarQube.
- Container image scanning with Trivy.
- Docker image building.
- Amazon ECR authentication and publishing.
- Immutable image digest output.
- Pull-request validation and main-branch releases.
- GitOps promotion without directly deploying from application CI.
- Centralized versioning, security controls, and pipeline maintenance.

The examples use GitHub Actions and AWS in `ap-south-1`. The workflows must be adapted to the actual repository configuration, application build commands, security requirements, and AWS identity setup before production use.

## 2. Architecture: reusable CI across 21 microservices

21 application repositories

Source code · tests · Dockerfile · service-specific workflow inputs

Reusable GitHub Actions platform

Build · test · code quality · security · image publishing

Amazon ECR

Versioned image tags · immutable image digest

GitOps repository

Reviewed chart version and image digest promotion

Argo CD → EKS

Deployment and reconciliation happen through GitOps

### 2.1 What is reusable?

There are two important reusable mechanisms in GitHub Actions.

| Mechanism         | Best use                                         | Example                                                              |
| ----------------- | ------------------------------------------------ | -------------------------------------------------------------------- |
| Reusable workflow | Standardize a complete job or multi-job pipeline | Build, test, scan, and publish an image                              |
| Composite action  | Package repeated steps within a job              | Set up a tool, run a common validation command, or generate metadata |

A reusable workflow is invoked with `jobs.<job_id>.uses`. A composite action is used as a step with `uses`.

They solve related but different problems. We will use both.

### 2.2 Why not one giant workflow?

A single enormous workflow that contains every language, every security tool, every build system, and every deployment rule becomes difficult to maintain.

Instead:

- Common controls belong in reusable platform components.
- Application-specific build and test commands remain configurable.
- Language-specific jobs use appropriate build tools.
- Publishing is separated from pull-request validation.
- Production deployment remains controlled by GitOps.

The result is centralized governance without forcing every application to use the same build command.

## 3. Repository design

We will use a dedicated platform repository for shared workflows.

```
company-ecommerce/
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
├── reporting-service/
├── platform-github-actions/
├── platform-helm-charts/
├── platform-terraform/
└── ecommerce-gitops/
```

Inside `platform-github-actions`:

```
platform-github-actions/
├── .github/
│   └── workflows/
│       ├── java-ci.yaml
│       ├── node-ci.yaml
│       ├── python-ci.yaml
│       ├── go-ci.yaml
│       ├── container-build-publish.yaml
│       └── helm-validation.yaml
├── actions/
│   ├── setup-java/
│   │   └── action.yaml
│   ├── setup-node/
│   │   └── action.yaml
│   ├── setup-python/
│   │   └── action.yaml
│   ├── setup-go/
│   │   └── action.yaml
│   ├── docker-metadata/
│   │   └── action.yaml
│   └── validate-image-reference/
│       └── action.yaml
├── scripts/
│   ├── validate-release-metadata.sh
│   └── verify-image-digest.sh
├── docs/
│   ├── workflow-contract.md
│   ├── security.md
│   ├── release-process.md
│   └── migration-guide.md
└── README.md
```

This is the shared pipeline repository. Each application repository still owns its application source, tests, Dockerfile, and service-specific configuration.

## 4. Establish the workflow contract

Before writing workflows, define what the reusable platform expects from application repositories.

### 4.1 Standard application repository contract

Each application repository should define:

| Item                     | Example for Order Service           |
| ------------------------ | ----------------------------------- |
| Application name         | `order-service`                     |
| Runtime                  | Java                                |
| Build system             | Maven                               |
| Source location          | `src/`                              |
| Unit test command        | `mvn test`                          |
| Package command          | `mvn package`                       |
| Dockerfile               | `Dockerfile`                        |
| Container port           | `8080`                              |
| Health endpoints         | `/actuator/health/...`              |
| Image repository         | ECR repository for Order Service    |
| Deployment configuration | Maintained in the GitOps repository |

Not every service uses Maven or port `8080`. The contract must describe the actual service rather than assume that all 21 services have identical internals.

### 4.2 Repository variables and secrets

Use GitHub configuration variables for non-secret configuration such as:

```
AWS_REGION
ECR_REPOSITORY
SONAR_PROJECT_KEY
```

Use GitHub environments and approved identity mechanisms for environment-specific access and deployment approvals.

Prefer GitHub OIDC federation to AWS over long-lived AWS access keys. With OIDC, GitHub obtains a short-lived identity token and AWS validates the configured trust relationship before issuing temporary credentials.

Never commit AWS access keys, SonarQube tokens, registry passwords, or production credentials to the repository.

### 4.3 Pin shared workflow versions

An application repository can reference a reusable workflow by a Git reference:

```
uses: company-ecommerce/platform-github-actions/.github/workflows/java-ci.yaml@v1.0.0
```

This is an illustrative reference. The organization, file path, and tag must match the actual repository.

For stronger supply-chain control, pin reusable workflows and third-party actions to reviewed immutable commit SHAs where required by organizational policy.

A shared workflow update should be released deliberately, not silently injected into all production pipelines.

## 5. Configure GitHub repository security

Reusable workflows are only as secure as the permissions and trust boundaries around them.

### 5.1 Set minimal default permissions

In each application repository, prefer a restrictive default:

```
permissions:
  contents: read
```

A job that publishes to ECR can explicitly request the permissions it needs:

```
permissions:
  contents: read
  id-token: write
```

`id-token: write` allows the job to request a GitHub OIDC token. It does not itself grant AWS access. AWS must also trust the correct GitHub identity and repository/environment claims.

Avoid granting `contents: write`, `packages: write`, or broad administrative permissions to every job when those permissions are not needed.

### 5.2 Protect the main branch

Recommended repository controls:

- Require pull requests for changes to the main branch.
- Require the relevant CI checks to pass.
- Require reviews for workflow and Dockerfile changes.
- Restrict who can approve or modify production release workflows.
- Protect release tags.
- Prevent unauthorized workflow changes from accessing production secrets.
- Enable appropriate secret scanning and dependency security controls.
- Review GitHub Actions permissions and allowed actions.

Exact controls depend on the organization's GitHub plan and repository settings.

### 5.3 Protect the shared workflow repository

The `platform-github-actions` repository is a high-impact dependency.

A change to its build or publishing workflow can affect all 21 services. Therefore:

1. Require code review.
2. Run tests on changes.
3. Publish versioned releases.
4. Maintain changelogs.
5. Test compatibility with representative application repositories.
6. Provide an upgrade path.
7. Avoid moving production users automatically to unreviewed changes.

A platform team should treat reusable workflows like versioned software products.

## 6. Design the reusable workflow interface

A reusable workflow receives inputs and secrets through the `workflow_call` trigger.

For example:

```
name: Reusable Java CI

on:
  workflow_call:
    inputs:
      java-version:
        description: Java version to use
        required: true
        type: string

      working-directory:
        description: Directory containing the Maven project
        required: false
        default: "."
        type: string

      run-sonar:
        description: Run the configured SonarQube analysis
        required: false
        default: false
        type: boolean

    secrets:
      SONAR_TOKEN:
        required: false
      SONAR_HOST_URL:
        required: false

    outputs:
      build-status:
        description: Result of the Java build job
        value: ${{ jobs.build.outputs.status }}
```

This defines a contract between the caller and the reusable workflow.

### 6.1 Input design rules

- Use `boolean` for true/false switches.
- Use `string` for version numbers and paths.
- Use defaults only when the default is safe and unambiguous.
- Validate unsupported values.
- Do not accept arbitrary shell commands from untrusted pull-request input.
- Avoid passing secrets unless the workflow needs them.
- Document each input and output.
- Keep the interface stable across minor updates.

A workflow should not accept dozens of arbitrary parameters if a smaller, well-defined contract would be safer and easier to use.

## 7. Composite action: standard Java setup

Create `actions/setup-java/action.yaml`:

```
name: Setup Java and Maven
description: Configure the approved Java runtime and Maven cache

inputs:
  java-version:
    description: Java version
    required: true

  distribution:
    description: Java distribution
    required: false
    default: temurin

  cache:
    description: Maven dependency caching
    required: false
    default: maven

runs:
  using: composite
  steps:
    - name: Set up Java
      uses: actions/setup-java@v4
      with:
        distribution: ${{ inputs.distribution }}
        java-version: ${{ inputs.java-version }}
        cache: ${{ inputs.cache }}
```

### 7.1 Why use a composite action here?

The action standardizes a repeated setup step. It does not need to orchestrate an entire build pipeline.

The platform team can later update the Java setup implementation in a reviewed release of the shared repository. Application teams then upgrade the reference when ready.

### 7.2 Important compatibility note

The example uses action version tags for readability. In a production organization, pin the external action to a reviewed commit SHA where policy requires immutable references. Maintain a documented update process for security patches.

## 8. Reusable Java CI workflow

Create `.github/workflows/java-ci.yaml` in `platform-github-actions`:

```
name: Reusable Java CI

on:
  workflow_call:
    inputs:
      java-version:
        required: true
        type: string

      working-directory:
        required: false
        type: string
        default: "."

      run-sonar:
        required: false
        type: boolean
        default: false

    secrets:
      SONAR_TOKEN:
        required: false
      SONAR_HOST_URL:
        required: false

    outputs:
      artifact-name:
        description: Name of the packaged application artifact
        value: ${{ jobs.build.outputs.artifact-name }}

jobs:
  build:
    name: Compile, Test and Package
    runs-on: ubuntu-latest

    permissions:
      contents: read

    outputs:
      artifact-name: ${{ steps.metadata.outputs.artifact-name }}

    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}

    steps:
      - name: Checkout application source
        uses: actions/checkout@v4

      - name: Set up Java and Maven
        uses: company-ecommerce/platform-github-actions/actions/setup-java@v1.0.0
        with:
          java-version: ${{ inputs.java-version }}

      - name: Verify Java version
        run: java -version

      - name: Verify Maven version
        run: mvn --version

      - name: Compile and run unit tests
        run: mvn --batch-mode --no-transfer-progress clean verify

      - name: Run SonarQube analysis
        if: inputs.run-sonar
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
        run: |
          set -euo pipefail

          if [[ -z "${SONAR_TOKEN:-}" ]]; then
            echo "SONAR_TOKEN is required when run-sonar is true" >&2
            exit 1
          fi

          if [[ -z "${SONAR_HOST_URL:-}" ]]; then
            echo "SONAR_HOST_URL is required when run-sonar is true" >&2
            exit 1
          fi

          mvn --batch-mode --no-transfer-progress \
            sonar:sonar \
            -Dsonar.host.url="$SONAR_HOST_URL"

      - name: Define artifact metadata
        id: metadata
        shell: bash
        run: |
          set -euo pipefail
          echo "artifact-name=java-build-${GITHUB_RUN_ID}-${GITHUB_RUN_ATTEMPT}" \
            >> "$GITHUB_OUTPUT"

      - name: Upload Maven build output
        uses: actions/upload-artifact@v4
        with:
          name: ${{ steps.metadata.outputs.artifact-name }}
          path: |
            ${{ inputs.working-directory }}/target/*.jar
          if-no-files-found: error
          retention-days: 7
```

### 8.1 Why run `mvn verify`?

Maven lifecycle phases include compilation, testing, packaging, and verification steps. Running `verify` allows configured checks and integration-test phases to run as part of the lifecycle.

However, the exact checks depend on the Maven project. Review `pom.xml` to determine whether integration tests, coverage, packaging, and quality gates are actually configured.

A successful Maven command is not proof that all expected tests ran. The pipeline must fail if the project has no tests when tests are required, or if the relevant test phase is skipped.

### 8.2 SonarQube configuration

SonarQube analysis requires a correctly configured project and server. The example assumes that the project's Maven configuration supports the `sonar:sonar` goal and that the server credentials are valid.

A production workflow should also define:

- The project key.
- Source and test directories.
- Coverage report paths.
- Quality gate enforcement.
- Supported scanner version.
- Network connectivity to the SonarQube server.
- Token rotation and access boundaries.

Do not assume that successfully uploading analysis means the quality gate has passed. Enforce the organization's quality-gate policy before allowing a release.

### 8.3 Shared action reference caveat

The example references a composite action using a version tag. Confirm that the referenced action exists at that tag and that the repository access permissions permit the caller to use it. GitHub Actions references to private repositories may require additional access configuration.

For a self-contained implementation, the Java setup steps can instead live directly inside the reusable workflow.

## 9. Call the reusable Java workflow from Order Service

Create `.github/workflows/ci.yaml` in the `order-service` repository:

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
    name: Java Build and Tests
    uses: company-ecommerce/platform-github-actions/.github/workflows/java-ci.yaml@v1.0.0

    with:
      java-version: "21"
      working-directory: "."
      run-sonar: true

    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
```

This is the application repository's caller workflow. The build logic remains in the shared workflow repository.

### 9.1 What happens when a pull request is opened?

1. GitHub starts the caller workflow.
2. The caller invokes the selected version of the reusable Java workflow.
3. The reusable workflow checks out the application source.
4. It configures Java.
5. It runs the Maven lifecycle.
6. It runs the configured SonarQube analysis.
7. It uploads the build artifact.
8. The job reports success or failure to the pull request.

This workflow does not publish an image to ECR or deploy to EKS. Those are separate responsibilities.

### 9.2 Why use `vars` for the SonarQube URL?

The SonarQube host URL is configuration, not a credential, so a repository or organization variable is appropriate.

The token is a secret and should be stored in GitHub Secrets or supplied through an approved secret-management integration.

### 9.3 Pull-request security

For pull requests from forks or untrusted contributors, do not expose secrets unnecessarily. GitHub restricts secret availability in many fork pull-request scenarios, but workflow design must still explicitly respect the trust boundary.

Do not use a privileged `pull_request_target` workflow to check out and execute untrusted pull-request code with access to write permissions or secrets.

## 10. Java build artifact strategy

The previous workflow uploads the packaged JAR as an artifact. This can be useful when another job consumes the result.

However, there are two different approaches to image building:

Approach A — Rebuild inside the Docker build stage.

The Docker build process compiles and packages the application. This is simple to reason about, but care is needed to avoid repeating expensive tests or using different build inputs.

Approach B — Build the JAR in CI, then package that artifact into an image.

The workflow uploads the JAR and a downstream image job downloads it. This separates application build from image packaging, but the artifact transfer and integrity need to be managed correctly.

For our project, we will make the build strategy explicit for each runtime. The key invariant is that the image that gets scanned and published must contain the application artifact that passed the required tests.

Do not compile a different commit or silently rebuild from a different branch during image publication.

## 11. Reusable Node.js CI workflow

Create `.github/workflows/node-ci.yaml`:

```
name: Reusable Node.js CI

on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string

      working-directory:
        required: false
        type: string
        default: "."

      package-manager:
        required: false
        type: string
        default: "npm"

jobs:
  build:
    name: Install, Test and Build
    runs-on: ubuntu-latest

    permissions:
      contents: read

    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: ${{ inputs.package-manager }}
          cache-dependency-path: ${{ inputs.working-directory }}/package-lock.json

      - name: Verify runtime
        run: |
          node --version
          npm --version

      - name: Install locked dependencies
        run: npm ci

      - name: Run lint
        run: npm run lint --if-present

      - name: Run unit tests
        run: npm test --if-present

      - name: Build application
        run: npm run build --if-present

      - name: Verify expected build output
        run: |
          set -euo pipefail
          test -f package.json
```

### 11.1 Make test requirements explicit

The example uses `--if-present` to allow reuse across projects with different scripts. This is convenient but may allow a repository without a test or lint script to pass without running those checks.

For a production service, require the appropriate scripts explicitly. For example:

```
{
  "scripts": {
    "lint": "eslint .",
    "test": "jest --ci",
    "build": "tsc"
  }
}
```

This is illustrative; actual scripts depend on the project's framework and test tools.

If tests are mandatory, the workflow should verify that the script exists and fail if it is missing.

### 11.2 Package manager consistency

Do not use `npm ci` for a repository that uses Yarn or pnpm.

The shared workflow can either:

- Provide separate npm, Yarn, and pnpm workflows, or
- Use a validated package-manager input with explicit branches for supported package managers.

Avoid dynamically executing an arbitrary shell command supplied by a caller.

### 11.3 Caller workflow for a Node.js service

```
name: Cart Service CI

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
  node-ci:
    uses: company-ecommerce/platform-github-actions/.github/workflows/node-ci.yaml@v1.0.0

    with:
      node-version: "22"
      working-directory: "."
      package-manager: "npm"
```

The Node.js version should match the version supported by the application and its dependencies. Pin the selected version in the repository contract and test upgrades before changing it.

## 12. Reusable Python CI workflow

Create `.github/workflows/python-ci.yaml`:

```
name: Reusable Python CI

on:
  workflow_call:
    inputs:
      python-version:
        required: true
        type: string

      working-directory:
        required: false
        type: string
        default: "."

      requirements-file:
        required: false
        type: string
        default: "requirements.txt"

jobs:
  build:
    name: Install, Lint and Test
    runs-on: ubuntu-latest

    permissions:
      contents: read

    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
          cache: pip
          cache-dependency-path: ${{ inputs.working-directory }}/${{ inputs.requirements-file }}

      - name: Verify Python
        run: python --version

      - name: Install dependencies
        run: |
          set -euo pipefail
          python -m pip install --upgrade pip
          python -m pip install -r "${{ inputs.requirements-file }}"

      - name: Run static checks
        run: |
          python -m compileall .

      - name: Run tests
        run: |
          python -m pytest -q
```

### 12.1 Python dependency management

The example assumes the project uses a requirements file and pytest. Projects using Poetry, uv, or another supported tool should have a workflow that respects their lockfile and installation process.

For reproducible builds:

- Maintain a dependency lock strategy.
- Pin or constrain dependency versions appropriately.
- Scan dependencies for known vulnerabilities.
- Avoid installing untrusted packages or executing arbitrary install scripts without review.
- Record the Python runtime version.
- Ensure tests actually run in CI.

### 12.2 Caller workflow for a Python service

```
name: Recommendation Service CI

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
  python-ci:
    uses: company-ecommerce/platform-github-actions/.github/workflows/python-ci.yaml@v1.0.0

    with:
      python-version: "3.12"
      working-directory: "."
      requirements-file: "requirements.txt"
```

The Python version and dependency installation approach must match the actual service.

## 13. Reusable Go CI workflow

Create `.github/workflows/go-ci.yaml`:

```
name: Reusable Go CI

on:
  workflow_call:
    inputs:
      go-version:
        required: true
        type: string

      working-directory:
        required: false
        type: string
        default: "."

jobs:
  build:
    name: Test and Build Go Application
    runs-on: ubuntu-latest

    permissions:
      contents: read

    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ inputs.go-version }}
          cache: true
          cache-dependency-path: ${{ inputs.working-directory }}/go.sum

      - name: Verify Go
        run: go version

      - name: Download dependencies
        run: go mod download

      - name: Verify dependency metadata
        run: go mod verify

      - name: Format check
        run: |
          set -euo pipefail
          test -z "$(gofmt -l .)"

      - name: Run tests
        run: go test ./...

      - name: Build application
        run: go build ./...
```

### 13.1 What these checks do

- `go mod download` downloads the dependencies declared by the module.
- `go mod verify` verifies downloaded module content against recorded checksums.
- `gofmt` checks formatting.
- `go test ./...` runs tests across the module's packages.
- `go build ./...` verifies that packages compile.

The application may need additional integration tests, race detection, linting, or cross-compilation checks.

### 13.2 Caller workflow

```
name: Fraud Service CI

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
  go-ci:
    uses: company-ecommerce/platform-github-actions/.github/workflows/go-ci.yaml@v1.0.0

    with:
      go-version: "1.24"
      working-directory: "."
```

Use a Go version that is supported by the application and available in the approved build environment. The version above is an example input, not a requirement to upgrade an existing service.

## 14. Standardize container image metadata

After an application passes its language-specific CI, the publishing pipeline needs to determine:

- The target ECR repository.
- The source commit.
- The image tags.
- The build context.
- The Dockerfile location.
- The image digest.
- The scan results.

The commit SHA is useful for traceability. The immutable image digest identifies the actual image artifact.

### 14.1 Recommended image naming

Example repository:

```
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
```

Example traceability tag:

```
order-service:7f2a1b6c...
```

The tag shown here represents a source commit SHA, not an actual published image.

Prefer immutable release references. A tag alone is not a cryptographic identity of the image unless the registry enforces immutability and the pipeline verifies the relationship.

### 14.2 Why the digest matters

A tag such as `main` can point to different images over time. A digest identifies a specific manifest or image index.

Example format:

```
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service@sha256:<64-hex-character-digest>
```

The placeholder is illustrative. The pipeline must obtain the actual digest from the image build/push operation or registry and verify that it matches the expected artifact.

The GitOps repository should use this immutable reference for environment promotion.

## 15. AWS OIDC authentication for GitHub Actions

We will use GitHub OIDC to authenticate the publishing job to AWS rather than storing long-lived AWS access keys in repository secrets.

### 15.1 Identity flow

GitHub Actions job

Requests an OIDC identity token

AWS IAM OIDC trust configuration

Validates issuer, audience, and trusted subject claims

Temporary AWS credentials

Only the permissions granted by the assumed role

Amazon ECR

Authenticate and publish the approved image

### 15.2 GitHub workflow permissions

The publishing job needs:

```
permissions:
  contents: read
  id-token: write
```

These permissions do not grant the job unrestricted AWS access. The IAM role trust policy and permissions policy determine what the job can do.

### 15.3 Trust policy design

The IAM role should trust the GitHub OIDC provider and constrain the subject to the intended repository and, where applicable, environment or branch.

Conceptually, a trust condition for a protected branch may restrict the subject to:

```
repo:company-ecommerce/order-service:ref:refs/heads/main
```

The actual IAM policy must use the correct OIDC provider, audience, and condition keys for the AWS account configuration. Do not copy a conceptual condition into production without validating the exact claims emitted for the chosen GitHub workflow and environment.

For production publishing, consider requiring a protected GitHub environment or a protected release tag so that untrusted pull requests cannot assume the publishing role.

### 15.4 Least-privilege ECR permissions

The publishing role should have only the permissions required for the intended repository operations, such as:

- Obtaining an authorization token.
- Checking image upload prerequisites.
- Uploading image layers.
- Completing image uploads.
- Publishing the image manifest.

The precise policy depends on the ECR workflow and whether the pipeline also creates or describes repositories. Separate repository provisioning from image publishing so routine CI does not need infrastructure-administration permissions.

## 16. Reusable image build and ECR publishing workflow

Create `.github/workflows/container-build-publish.yaml` in the platform repository.

This workflow is deliberately separate from the language-specific workflows. It packages a previously tested source revision into a container image, scans it, publishes it to ECR, and returns the immutable image digest.

```
name: Reusable Container Build and Publish

on:
  workflow_call:
    inputs:
      aws-region:
        description: AWS region containing ECR
        required: true
        type: string

      aws-role-arn:
        description: IAM role ARN for the trusted publishing workflow
        required: true
        type: string

      ecr-repository:
        description: ECR repository name
        required: true
        type: string

      dockerfile:
        description: Dockerfile path relative to the build context
        required: false
        type: string
        default: Dockerfile

      build-context:
        description: Docker build context
        required: false
        type: string
        default: "."

      image-tag:
        description: Unique image tag for this build
        required: true
        type: string

      publish:
        description: Whether to publish the image
        required: false
        type: boolean
        default: false

    outputs:
      image-uri:
        description: Published image URI without a tag
        value: ${{ jobs.image.outputs.image-uri }}

      image-digest:
        description: Digest returned by the image publishing step
        value: ${{ jobs.image.outputs.image-digest }}

jobs:
  image:
    name: Build, Scan and Publish
    runs-on: ubuntu-latest

    permissions:
      contents: read
      id-token: write

    outputs:
      image-uri: ${{ steps.metadata.outputs.image-uri }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Validate image inputs
        env:
          ECR_REPOSITORY: ${{ inputs.ecr-repository }}
          IMAGE_TAG: ${{ inputs.image-tag }}
          DOCKERFILE: ${{ inputs.dockerfile }}
          BUILD_CONTEXT: ${{ inputs.build-context }}
        run: |
          set -euo pipefail

          [[ "$ECR_REPOSITORY" =~ ^[a-z0-9]+([._/-][a-z0-9]+)*$ ]] || {
            echo "Invalid ECR repository name" >&2
            exit 1
          }

          [[ "$IMAGE_TAG" =~ ^[A-Za-z0-9_][A-Za-z0-9_.-]{0,127}$ ]] || {
            echo "Invalid image tag" >&2
            exit 1
          }

          test -f "$BUILD_CONTEXT/$DOCKERFILE" || {
            echo "Dockerfile not found: $BUILD_CONTEXT/$DOCKERFILE" >&2
            exit 1
          }

      - name: Configure AWS credentials
        if: inputs.publish
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ inputs.aws-role-arn }}
          aws-region: ${{ inputs.aws-region }}

      - name: Log in to Amazon ECR
        if: inputs.publish
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      - name: Define image metadata
        id: metadata
        env:
          REGISTRY: ${{ steps.ecr-login.outputs.registry }}
          ECR_REPOSITORY: ${{ inputs.ecr-repository }}
          IMAGE_TAG: ${{ inputs.image-tag }}
        run: |
          set -euo pipefail

          if [[ "${{ inputs.publish }}" == "true" ]]; then
            test -n "$REGISTRY"
            IMAGE_URI="${REGISTRY}/${ECR_REPOSITORY}"
          else
            IMAGE_URI="local-validation/${ECR_REPOSITORY}"
          fi

          echo "image-uri=$IMAGE_URI" >> "$GITHUB_OUTPUT"

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and scan image
        id: build
        uses: docker/build-push-action@v6
        with:
          context: ${{ inputs.build-context }}
          file: ${{ inputs.build-context }}/${{ inputs.dockerfile }}
          push: ${{ inputs.publish }}
          tags: ${{ steps.metadata.outputs.image-uri }}:${{ inputs.image-tag }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          outputs: type=docker,dest=/tmp/application-image.tar

      - name: Load image into Docker
        if: ${{ !inputs.publish }}
        run: docker load --input /tmp/application-image.tar

      - name: Run Trivy image scan
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: ${{ steps.metadata.outputs.image-uri }}:${{ inputs.image-tag }}
          format: table
          exit-code: "1"
          ignore-unfixed: true
          vuln-type: os,library
          severity: CRITICAL,HIGH
```

### 16.1 Critical correction to this starter workflow

The example illustrates the inputs and job boundary, but its build and scan arrangement must be finalized before production use.

In particular, the `outputs` configuration requests a Docker archive even when publishing is enabled. A single build invocation must not be configured with conflicting output/export modes. For a production workflow, use separate build paths:

- Validation path: build locally, export the image, load it, then scan the local image.
- Publishing path: build and push to ECR, obtain the pushed digest, then scan the exact pushed image by digest.

Do not rely on a starter workflow with incompatible build outputs. Section 17 provides a corrected implementation pattern.

### 16.2 Why scan the published artifact?

Scanning a locally built image before publishing is useful. Scanning the exact pushed digest is stronger evidence that the artifact entering the release process was evaluated.

The production gate should tie the scan result to the image digest and fail when the required severity policy is violated.

### 16.3 Do not silently allow failed scans

If the policy blocks unresolved critical or high vulnerabilities, the scan must fail the release job when those findings are present, according to the approved exception policy.

`ignore-unfixed: true` excludes findings without a fix available under the scanner's interpretation. This is a policy choice, not proof that the remaining vulnerabilities are safe. Document the choice and consider additional controls for exploitable or otherwise high-risk findings.

## 17. Corrected container build and publish implementation

For a production pipeline, make validation and publishing explicit rather than mixing incompatible Docker Buildx outputs.

### 17.1 Recommended job flow

1. Validate source and application tests.
2. Build the container image.
3. Scan the image.
4. Publish only after the required checks pass.
5. Capture the resulting digest.
6. Verify that the digest is available in ECR.
7. Pass the digest to the GitOps promotion workflow.

A stronger workflow separates the build-validation path from the publishing path, or uses a tested build-and-push design that can scan the exact pushed artifact before permitting promotion. If policy requires scanning before publication, build and scan locally first, then publish the approved image and verify the resulting digest.

### 17.2 Example: local build and scan

The following steps assume the image URI and tag have already been validated and that the job has checked out the correct source revision.

```
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build local image for validation
  uses: docker/build-push-action@v6
  with:
    context: .
    file: ./Dockerfile
    push: false
    load: true
    tags: local-validation/order-service:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max

- name: Scan local image
  uses: aquasecurity/trivy-action@0.28.0
  with:
    image-ref: local-validation/order-service:${{ github.sha }}
    format: table
    exit-code: "1"
    ignore-unfixed: true
    vuln-type: os,library
    severity: CRITICAL,HIGH
```

This path creates an image available to the local Docker daemon and scans that image. The action versions shown are examples; production pipelines should pin external actions to reviewed immutable SHAs where required.

### 17.3 Example: publish the approved image

After validation succeeds, authenticate to AWS and publish the image:

```
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ vars.AWS_ECR_PUBLISH_ROLE_ARN }}
    aws-region: ap-south-1

- name: Log in to Amazon ECR
  id: ecr-login
  uses: aws-actions/amazon-ecr-login@v2

- name: Build and push approved image
  id: publish
  uses: docker/build-push-action@v6
  with:
    context: .
    file: ./Dockerfile
    push: true
    tags: |
      ${{ steps.ecr-login.outputs.registry }}/order-service:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

This second build must use the same source revision and controlled build inputs as the validated build. However, rebuilding can produce a different image if the build is not reproducible or depends on mutable external inputs.

For stronger artifact identity, use a pipeline architecture that preserves and publishes the exact artifact that was scanned, or scan and verify the pushed digest before allowing that digest to be promoted. Record the relationship between the tested source commit and the final published digest.

### 17.4 Capture the image digest

The publishing action exposes a digest output. Pass it through the job outputs:

```
jobs:
  publish:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      id-token: write

    outputs:
      image-digest: ${{ steps.publish.outputs.digest }}

    steps:
      # Checkout, authenticate and publish steps go here.
      # The build-push-action step must have id: publish.

      - name: Validate digest output
        env:
          IMAGE_DIGEST: ${{ steps.publish.outputs.digest }}
        run: |
          set -euo pipefail
          [[ "$IMAGE_DIGEST" =~ ^sha256:[a-f0-9]{64}$ ]] || {
            echo "The image build did not return a valid digest" >&2
            exit 1
          }
```

The snippet illustrates digest validation and output wiring. The complete job must contain the checkout, AWS authentication, ECR login, and image build steps before the digest is referenced.

### 17.5 Verify the pushed artifact

After publishing, verify the image manifest in ECR:

```
aws ecr describe-images \
  --region ap-south-1 \
  --repository-name order-service \
  --image-ids imageDigest="$IMAGE_DIGEST"
```

Set `IMAGE_DIGEST` to the actual digest output from the build step.

If the repository has immutable tags enabled, a commit-SHA tag can also provide useful traceability. Nevertheless, the digest is the authoritative artifact reference used by the GitOps release process.

### 17.6 Production artifact rule

The release pipeline should fail if:

- The image digest is missing or malformed.
- The expected ECR repository does not match the service.
- The published artifact cannot be found.
- The required scan has not passed.
- The build used an unauthorized source revision.
- The required attestations or provenance checks are absent.
- The artifact does not meet the organization's release policy.

## 18. Scan the exact ECR image digest

A scan of the local image is useful, but the final release record should identify the artifact actually published.

After ECR publishing, a workflow can pull the image by digest and scan it.

Example shell pattern:

```
set -euo pipefail

IMAGE_URI="${ECR_REGISTRY}/order-service"
IMAGE_DIGEST="${IMAGE_DIGEST:?IMAGE_DIGEST is required}"
IMAGE_REF="${IMAGE_URI}@${IMAGE_DIGEST}"

[[ "$IMAGE_DIGEST" =~ ^sha256:[a-f0-9]{64}$ ]] || {
  echo "Invalid image digest" >&2
  exit 1
}

docker pull "$IMAGE_REF"

trivy image \
  --exit-code 1 \
  --ignore-unfixed \
  --severity HIGH,CRITICAL \
  "$IMAGE_REF"
```

This assumes the runner has Docker and Trivy installed and has already authenticated to ECR. The actual scan policy should be centrally maintained and versioned.

### 18.1 Why scan by digest?

The tag could be changed or moved. The digest identifies the image artifact being evaluated.

The release record should associate the following:

```
Source commit
    ↓
Build run
    ↓
Image digest
    ↓
Scan result
    ↓
Approval record
    ↓
GitOps promotion commit
```

This chain makes it easier to answer questions such as:

- Which source commit produced this production image?
- Which security scan was performed on it?
- Which image digest was approved?
- Which GitOps commit promoted it?
- Which environments currently run that digest?

### 18.2 Vulnerability exceptions

If a scan fails, the workflow should not simply rerun the same scan until it passes.

Instead:

1. Identify the affected package and vulnerability.
2. Determine whether an approved fixed version is available.
3. Update the dependency or base image when appropriate.
4. Rebuild and retest.
5. If a formal exception is justified, record its scope, reason, compensating controls, approver, and expiry.
6. Reassess the artifact under the organization's policy.

An exception should not silently become a permanent global bypass.

## 19. Dockerfile standards for the microservices

The shared workflow depends on application repositories following a predictable Docker build contract.

### 19.1 Example multi-stage Dockerfile for Spring Boot

```
FROM maven:3.9-eclipse-temurin-21 AS build

WORKDIR /workspace

COPY pom.xml .
COPY src ./src

RUN mvn --batch-mode --no-transfer-progress clean package -DskipTests

FROM eclipse-temurin:21-jre

WORKDIR /app

RUN groupadd --system --gid 10001 app \
    && useradd --system --uid 10001 --gid 10001 app

COPY --from=build --chown=10001:10001 \
    /workspace/target/*.jar /app/application.jar

USER 10001:10001

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app/application.jar"]
```

This is a teaching example, not a complete production Dockerfile.

Important considerations:

- Pin base images to reviewed versions and preferably immutable digests.
- Ensure the artifact copy pattern resolves to exactly the intended JAR.
- Keep credentials out of image layers and build arguments.
- Use a suitable non-root runtime image.
- Ensure the application supports the chart's security context.
- Consider whether Maven should run tests in the image build or whether CI should build and transfer a tested artifact.
- Verify the base image's package-maintenance and vulnerability-management process.

The `-DskipTests` option skips test execution in this Dockerfile. It is only defensible when the pipeline has already run the required tests against the same source and the final image is verified to contain the corresponding artifact. The Dockerfile should not be the only place where testing occurs.

### 19.2 Example Node.js Dockerfile

```
FROM node:22-alpine AS build

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM node:22-alpine

WORKDIR /app

ENV NODE_ENV=production

COPY package*.json ./
RUN npm ci --omit=dev

COPY --from=build /app/dist ./dist

USER node

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

This example assumes the application builds into `dist/` and starts from `dist/index.js`. Adjust the paths and runtime command to match the actual application.

For production, consider:

- Lockfile integrity.
- Native dependency compatibility between build and runtime stages.
- Base-image updates.
- Runtime file permissions.
- Whether development dependencies are needed at runtime.
- Whether the process handles termination signals correctly.

### 19.3 Example Python Dockerfile

```
FROM python:3.12-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app

RUN useradd --system --uid 10001 app \
    && chown -R 10001:10001 /app

USER 10001:10001

EXPOSE 8000

CMD ["python", "-m", "app"]
```

This assumes the project can be started with `python -m app`. Use the real application entry point and ensure required native dependencies are installed.

### 19.4 Example Go Dockerfile

```
FROM golang:1.24-alpine AS build

WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -trimpath \
    -o /out/application \
    ./cmd/server

FROM scratch

COPY --from=build /out/application /application

USER 10001:10001

EXPOSE 8080

ENTRYPOINT ["/application"]
```

This example assumes the main package is in `./cmd/server` and that a static binary is suitable for the application. Some Go applications need certificates, timezone data, shared libraries, or other runtime files, in which case `scratch` may not be appropriate.

### 19.5 Docker build context and `.dockerignore`

Each application repository should maintain a `.dockerignore` file.

Example:

```
.git
.github
node_modules
target
dist
.venv
__pycache__
*.pyc
.env
.env.*
*.pem
*.key
*.p12
*.pfx
```

Adjust this list for the runtime. Do not exclude required build artifacts or files used by the Dockerfile.

A smaller build context improves efficiency and reduces the risk of accidentally sending secrets or unnecessary files into the build process.

## 20. Security scanning stages

The CI pipeline should use multiple complementary checks rather than treating a single scanner as a complete security solution.

| Stage                 | Tool example                                  | Purpose                                               |
| --------------------- | --------------------------------------------- | ----------------------------------------------------- |
| Source code           | SonarQube                                     | Code quality and configured static analysis           |
| Dependencies          | Dependabot or an approved dependency scanner  | Known vulnerable dependencies                         |
| Secrets               | GitHub secret scanning or an approved scanner | Accidentally committed credentials                    |
| Container image       | Trivy                                         | OS package and application dependency vulnerabilities |
| IaC and Kubernetes    | Trivy, Checkov, or policy-as-code             | Configuration and infrastructure risks                |
| Artifact verification | Approved signature/provenance tooling         | Establish artifact identity and trusted build origin  |

The selected tools and their policies should be centrally managed, while application teams own remediation of findings in their code and dependencies.

### 20.1 SonarQube stage

For Java, the analysis commonly integrates with Maven. For other runtimes, use the scanner appropriate to the language and repository.

The quality gate should be checked as a release condition. A pipeline that uploads results but never enforces the required gate is not enforcing the policy.

### 20.2 Trivy stage

A basic image scan:

```
trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  "$IMAGE_REF"
```

A production policy must define:

- Which severities block release.
- Whether unfixed findings are excluded.
- How exceptions are approved.
- How scanner database freshness is managed.
- Whether the scan runs before publication, after publication, or both.
- How scan evidence is retained.
- Whether license, misconfiguration, or secret scanning is required.

Do not assume that every service should have exactly the same exception policy. Some organizations apply stricter rules to public-facing or payment-related workloads.

### 20.3 Dependency scanning

Dependency scanning should run early enough to provide developers with actionable feedback. The image scan complements it by examining the packages present in the resulting container.

A Java dependency may be vulnerable even if the operating-system package scan is clean. Conversely, a vulnerable operating-system package may exist in a base image even when the application's dependencies are up to date.

These checks are related but not interchangeable.

## 21. Reusable security scanning workflow

A central security workflow can collect the mandatory checks and their evidence.

Create `.github/workflows/security-checks.yaml`:

```
name: Reusable Security Checks

on:
  workflow_call:
    inputs:
      run-trivy-fs:
        required: false
        type: boolean
        default: true

      severity:
        required: false
        type: string
        default: "HIGH,CRITICAL"

jobs:
  filesystem-scan:
    name: Scan Repository Filesystem
    if: inputs.run-trivy-fs
    runs-on: ubuntu-latest

    permissions:
      contents: read

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Scan repository with Trivy
        uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: fs
          scan-ref: .
          format: table
          exit-code: "1"
          ignore-unfixed: true
          severity: ${{ inputs.severity }}
```

This is a reusable filesystem scan, not a complete security pipeline. Add the required secret scanning, dependency analysis, IaC checks, and image scanning according to the platform's policy.

### 21.1 Avoid arbitrary severity inputs

The example accepts a severity string. In a mature platform, validate the allowed values or define a small set of approved profiles, such as:

- Pull-request advisory profile.
- Required merge gate.
- Production release gate.

This prevents callers from weakening a mandatory release check by supplying an empty or overly permissive value.

### 21.2 Keep policy ownership centralized

Application teams may add extra scans, but they should not be able to bypass the required platform checks by changing a local workflow.

Use repository rules, protected workflow references, code-owner reviews, and restricted publishing credentials to enforce the intended boundary.

## 22. Reusable workflow outputs

A downstream workflow often needs information from a reusable workflow.

For example, the publishing workflow can return:

```
image-uri
image-digest
source-commit
scan-result
```

GitHub reusable workflow outputs are exposed through the `workflow_call` outputs and mapped from job outputs.

Example:

```
on:
  workflow_call:
    outputs:
      image-digest:
        description: Immutable digest of the published image
        value: ${{ jobs.publish.outputs.image-digest }}
```

The `publish` job must define the corresponding output:

```
jobs:
  publish:
    runs-on: ubuntu-latest

    outputs:
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - name: Build and push
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: example.invalid/order-service:example
```

The example registry and tag are placeholders. A real workflow must authenticate to the intended registry, use the correct repository and source revision, and enforce the scan and publication requirements.

### 22.1 Why pass the digest as an output?

It avoids trying to reconstruct the digest from a tag or rerun the build to determine which image was published.

The next job can use the output directly, validate it, and record it in the release metadata.

### 22.2 Outputs are not release approvals

A valid digest output means the build step returned a digest. It does not prove that the release is approved.

Release eligibility should also depend on the required tests, security results, source trust, approvals, and artifact verification.

## 23. Pull request versus release workflow

Not every workflow event should have the same permissions.

### Pull-request validation

Typical behavior:

- Read the source.
- Install dependencies.
- Run unit tests and static checks.
- Build locally if needed.
- Run security scans.
- Publish test reports.
- Do not receive production AWS credentials.
- Do not publish a production image.
- Do not modify production GitOps state.

### Main-branch or release workflow

Typical behavior:

- Validate the protected source revision.
- Run required tests and scans.
- Authenticate to the approved publishing role.
- Publish the container image.
- Record the immutable digest.
- Create or update a GitOps change.
- Require the appropriate review before production promotion.

The distinction is important because pull-request code may be untrusted, even if the repository itself belongs to the organization.

### 23.1 Example event conditions

```
on:
  pull_request:
    branches:
      - main

  push:
    branches:
      - main
```

This triggers the workflow on pull requests targeting `main` and on pushes to `main`. It does not by itself ensure that only reviewed code reaches `main`; branch protection and required reviews must enforce that.

### 23.2 Do not publish from every pull request

Publishing an image for a trusted internal pull request can be useful in some workflows, but it should use a separate non-production repository or isolated credentials where appropriate.

Do not expose production publishing permissions to untrusted fork pull requests. Use a deliberate trust model rather than a broad `if` condition that is difficult to audit.

## 24. Connect CI to GitOps without deploying directly

After the image has been built, scanned, and published, the pipeline needs to communicate the new artifact to the GitOps repository.

The preferred approach is a pull request that changes the service's image digest in the environment values.

Example GitOps values:

```
image:
  repository: "123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service"
  digest: "sha256:REPLACE_WITH_APPROVED_DIGEST"
  tag: ""
```

The digest is a placeholder. The release automation should write the actual verified digest, not this literal value.

### 24.1 Why use a GitOps pull request?

A pull request provides:

- Reviewable changes.
- An auditable promotion record.
- Clear separation between artifact creation and deployment.
- Environment-specific approval controls.
- A rollback path through Git history.
- A traceable relationship between the image and the deployment configuration.

The pipeline should not commit an unreviewed production change simply because an image was built successfully.

### 24.2 Promotion through environments

A typical progression is:

```
Build and scan
    ↓
Publish immutable image
    ↓
DEV GitOps change
    ↓
QA verification
    ↓
STAGING verification
    ↓
Production approval
    ↓
PROD GitOps change
```

The exact automation can vary. For example, promotion may automatically open the next environment's pull request after tests pass, while production always requires explicit approval.

The important point is that the same image digest is promoted. Avoid rebuilding the source separately for each environment if the objective is to promote the exact tested artifact.

### 24.3 GitOps writer permissions

The identity that updates the GitOps repository should have the minimum required permissions.

Prefer a dedicated GitHub App or another approved identity with narrowly scoped access. Avoid using a developer's personal access token as a permanent pipeline credential.

The identity should not be able to bypass branch protection or merge production changes without the required approvals.

## 25. Application-specific caller workflows

The following examples show how the 21 services can select their language workflow.

These are illustrative mappings. Confirm each service's actual runtime before adopting them.

| Service        | Example CI workflow |
| -------------- | ------------------- |
| Auth           | Java CI             |
| User           | Java CI             |
| Customer       | Java CI             |
| Catalog        | Java CI             |
| Product        | Java CI             |
| Inventory      | Java CI             |
| Pricing        | Java CI             |
| Promotion      | Java CI             |
| Cart           | Node.js CI          |
| Order          | Java CI             |
| Payment        | Java CI             |
| Fraud          | Go CI               |
| Shipping       | Java CI             |
| Delivery       | Java CI             |
| Notification   | Python CI           |
| Search         | Java CI             |
| Recommendation | Python CI           |
| Review         | Java CI             |
| Tax            | Java CI             |
| Invoice        | Java CI             |
| Reporting      | Java CI             |

This mapping is only a sample architecture. The earlier project context does not establish the exact implementation language for every service, so the mapping must be confirmed against the actual repositories.

### 25.1 Keep service-specific differences explicit

A service may need:

- Integration tests against a database.
- Contract tests against another service.
- Schema validation for event payloads.
- A generated client.
- A native library.
- A frontend build step.
- A specialized packaging command.
- Additional security tests.

Do not hide these requirements in undocumented conditionals inside a shared workflow. Add explicit inputs or a dedicated service workflow where needed.

## 26. Build efficiency and caching

Caching can reduce build time, but a cache is not a substitute for dependency integrity or artifact verification.

### 26.1 Maven caching

The Java setup action can cache Maven dependencies. Ensure that the cache key is sensitive to the relevant dependency files and that builds remain correct when the cache is empty.

A clean build must still succeed.

### 26.2 Node.js caching

Use the lockfile as the cache dependency path. Do not assume a cache makes an unpinned dependency reproducible.

### 26.3 Docker layer caching

The Docker Buildx GitHub Actions cache can improve repeated builds.

Example:

```
cache-from: type=gha
cache-to: type=gha,mode=max
```

Review cache scope and trust boundaries. Avoid allowing untrusted builds to poison caches that privileged release builds depend on.

### 26.4 Measure the effect

Track:

- Build duration.
- Cache hit rate.
- Test duration.
- Queue time.
- Runner utilization.
- Failed build rate.
- Repeated dependency downloads.
- Image build time.

Do not optimize caching at the expense of reproducibility or supply-chain security.

## 27. Production workflow troubleshooting

### Scenario 1 — Reusable workflow not found

Example error:

```
workflow was not found
```

Check:

1. Organization and repository names.
2. The path under `.github/workflows/`.
3. Whether the referenced Git tag or commit exists.
4. Whether the workflow contains `on: workflow_call`.
5. Whether the caller is permitted to access the repository.
6. Whether the workflow file is present at the referenced revision.

A reusable workflow is not the same as a composite action. A composite action must have an `action.yaml` file and be referenced as a step.

### Scenario 2 — Invalid workflow syntax

Validate the YAML and workflow structure using an approved GitHub Actions linter or CI validator.

Check for:

- Incorrect indentation.
- Incorrect placement of `uses`.
- Unsupported input types.
- Invalid expressions.
- Incorrect output references.
- Missing `workflow_call`.
- Incorrect secret declarations.
- Job dependencies referencing nonexistent jobs.

Remember that GitHub Actions YAML is interpreted by GitHub's workflow engine. A generic YAML parser can validate syntax but cannot verify all workflow semantics.

### Scenario 3 — Maven build fails

Start with the actual Maven output:

```
mvn --batch-mode --no-transfer-progress clean verify
```

Investigate:

- Dependency resolution failures.
- Java version mismatch.
- Incorrect Maven configuration.
- Compilation errors.
- Failing unit tests.
- Failing integration tests.
- Missing environment variables.
- Incorrect test profiles.
- Network restrictions on dependency repositories.

Reproduce the failure using the same Java version and Maven command as CI.

### Scenario 4 — SonarQube analysis fails

Check:

- The configured server URL.
- Token validity and permissions.
- Project key.
- Network connectivity.
- Scanner compatibility.
- Coverage report paths.
- Quality gate status.
- Whether the analysis actually reached the intended project.

Do not print tokens to logs while debugging.

### Scenario 5 — AWS OIDC role assumption fails

Check:

- The workflow has `id-token: write`.
- The IAM OIDC provider exists.
- The AWS role trust policy references the correct provider.
- The audience matches the expected value.
- The repository, branch, or environment subject condition matches the actual workflow identity.
- The workflow is running from an authorized ref.
- The role ARN is correct.

Do not fix a trust-policy mismatch by granting unrestricted AWS access.

### Scenario 6 — ECR login fails

Check:

```
aws sts get-caller-identity
aws ecr describe-repositories --region ap-south-1
```

The second command requires the relevant ECR permissions.

Verify that the assumed identity is the intended publishing role, that the repository exists in the correct account and region, and that the role has the necessary ECR permissions.

### Scenario 7 — Docker build fails

Inspect the Docker build output.

Common causes:

- Wrong build context.
- Incorrect Dockerfile path.
- Missing files due to `.dockerignore`.
- Failed dependency downloads.
- Invalid base image.
- Incorrect architecture.
- Incorrect file permissions.
- Native dependency incompatibility.
- Application build failures.

Do not fix a failed build by adding the entire repository to the build context without reviewing what files will be included.

### Scenario 8 — Trivy reports vulnerabilities

Identify the package and affected version, then determine whether a fix exists.

Possible actions:

- Upgrade the application dependency.
- Upgrade the base image.
- Remove an unnecessary package.
- Rebuild and retest the image.
- Request a time-bound exception through the approved process.

Do not disable the scanner or globally ignore a vulnerability solely to make the pipeline green.

### Scenario 9 — Digest output is empty

Check:

- The build step has an `id`.
- The publishing action actually pushed the image.
- The job output maps to the correct step output.
- The reusable workflow output maps to the correct job output.
- The caller references the output through the correct job identifier.
- The publishing path did not use a local-only build configuration.

Validate the digest format and verify the digest in ECR before passing it to GitOps.

### Scenario 10 — GitOps pull request was created with the wrong digest

Stop promotion and identify the incorrect source.

Check the relationship between:

- Source commit.
- CI run.
- ECR repository.
- Image tag.
- Image digest.
- GitOps pull request.

Correct the GitOps change and rerun verification. Do not infer a digest from a tag or silently rebuild the image without recording the new artifact identity.

### Scenario 11 — Production pipeline can access secrets on an untrusted branch

Treat this as a security incident until the exposure is understood.

1. Disable the affected workflow or publishing path if necessary.
2. Determine which identities and credentials were available.
3. Revoke or rotate exposed credentials as appropriate.
4. Review the workflow event type and permissions.
5. Review OIDC trust conditions and GitHub environment protections.
6. Inspect logs and audit records.
7. Add regression controls and required reviews.

OIDC reduces reliance on long-lived credentials, but a workflow with an overly broad trust policy can still assume a privileged role.

## 28. Reusable workflow versioning and upgrades

### 28.1 Use release tags

Example:

```
v1.0.0
v1.1.0
v1.1.1
v2.0.0
```

Use a documented versioning policy.

- Patch: compatible bug fix.
- Minor: compatible new capability.
- Major: breaking input, output, behavior, or security-contract change.

### 28.2 Avoid silently changing behavior

A reusable workflow can affect many application repositories. A change to a default test command, scanner severity, build runtime, or AWS role behavior can break multiple services.

For every shared workflow update:

1. Add tests.
2. Test against representative applications.
3. Document the behavior change.
4. Publish a version.
5. Update callers deliberately.
6. Monitor failures after rollout.
7. Retain a rollback path.

### 28.3 Test compatibility across the 21 services

Use a representative test matrix covering different runtimes, build systems, and special requirements.

For example:

- One Spring Boot service.
- One Node.js service.
- One Python service.
- One Go service.
- One service with integration tests.
- One service with a specialized Docker build.
- One service that publishes a worker image.
- One service with additional security checks.

After representative testing passes, roll out the shared workflow update in controlled groups rather than changing every repository simultaneously.

## 29. Migration from independent pipelines to reusable workflows

If the 21 repositories already have separate pipelines, migrate incrementally.

### Phase A — Inventory existing pipelines

Record for every service:

- Runtime and version.
- Build command.
- Test commands.
- Dockerfile path.
- Image repository.
- Security checks.
- Required secrets.
- Existing deployment behavior.
- Special build requirements.
- Known pipeline failures.

Do not assume two Java services use the same Maven lifecycle simply because they share a language.

### Phase B — Establish a baseline

Record the existing pipeline's behavior:

- Build success rate.
- Typical build duration.
- Test coverage or test execution evidence.
- Scanner results.
- Image publication process.
- Deployment handoff.
- Manual maintenance effort.

This baseline allows the platform team to measure whether the reusable workflows improve consistency and efficiency.

### Phase C — Pilot one service

Start with a representative service, such as Order Service.

1. Create the caller workflow.
2. Invoke the reusable Java workflow.
3. Compare its results with the previous pipeline.
4. Verify test and scanner behavior.
5. Validate the container build.
6. Publish a test image.
7. Verify the digest.
8. Confirm that no unintended deployment occurs.

Do not delete the previous pipeline until the replacement has passed the agreed acceptance criteria.

### Phase D — Expand by runtime

Migrate groups of services after the pilot succeeds.

Use the same process for Java, Node.js, Python, and Go. Maintain explicit exceptions for services that need specialized behavior.

### Phase E — Enforce the platform contract

After the migration is stable:

- Require approved shared workflows.
- Restrict unauthorized production publishing paths.
- Require security gates.
- Standardize artifact naming and digest recording.
- Document supported versions.
- Monitor workflow failures.
- Maintain an exception register.

## 30. Operational metrics for reusable CI

The platform team should measure the outcome rather than assume reusable workflows automatically improve delivery.

Recommended metrics:

| Metric                                | What it indicates                                |
| ------------------------------------- | ------------------------------------------------ |
| Pipeline success rate                 | Overall workflow reliability                     |
| Median build duration                 | Developer feedback speed                         |
| Pull-request time to first result     | How quickly developers receive CI feedback       |
| Test failure rate                     | Application or test-suite instability            |
| Security gate failure rate            | Frequency and nature of detected security issues |
| Image publication failure rate        | Registry or publishing reliability               |
| Digest verification failures          | Artifact integrity or workflow wiring problems   |
| Shared workflow adoption              | Consistency across repositories                  |
| Mean time to repair pipeline failures | Maintainability                                  |
| Rollback frequency after releases     | Possible release-quality issues                  |

Interpret metrics in context. A high security-gate failure rate may indicate improved detection rather than worsening application quality. A fast pipeline is not beneficial if it skips required tests.

## 31. Production release runbook

Before promoting a new application image:

### Source and tests

- The source commit is reviewed and authorized.
- Required unit and integration tests pass.
- Static analysis and dependency checks pass.
- Build inputs are controlled and recorded.
- The Dockerfile and build context are reviewed.

### Image and security

- The image is built from the intended source revision.
- Required image scans pass.
- Approved exceptions are documented.
- The image is published to the correct ECR repository.
- The image digest is captured.
- The digest is verified in ECR.
- The artifact has required provenance or attestation evidence where applicable.

### GitOps and deployment

- The GitOps change references the correct image digest.
- The chart version is explicit.
- Environment values are correct.
- Required reviewers approve the change.
- Argo CD can retrieve the chart and image.
- Rollout and application-level verification are ready.
- Rollback instructions are available.

## 32. Practical exercise

Implement this chapter in the platform repository.

### Exercise A — Create the shared workflows

- Create the Java reusable workflow.
- Create the Node.js reusable workflow.
- Create the Python reusable workflow.
- Create the Go reusable workflow.
- Create the security workflow.
- Create the container build and publish workflow.
- Define workflow inputs and outputs.
- Document the supported contract.

### Exercise B — Integrate Order Service

- Create the caller workflow.
- Configure the correct Java version.
- Run Maven tests.
- Configure SonarQube if required.
- Build the container image.
- Run Trivy according to the approved policy.
- Configure OIDC authentication.
- Publish the image to ECR.
- Capture and verify the digest.

### Exercise C — Validate the security boundary

- Confirm pull requests do not receive production publishing credentials.
- Confirm the publishing role trusts only authorized GitHub identities.
- Confirm IAM permissions are limited to the required ECR operations.
- Confirm secrets are not printed in logs.
- Confirm untrusted code cannot bypass mandatory checks.
- Confirm the GitOps writer cannot bypass required production approval.

### Exercise D — Test GitOps handoff

- Create a GitOps pull request with the actual digest.
- Verify the correct environment values.
- Verify the chart version.
- Verify the artifact identity against CI output.
- Confirm that the workflow does not directly deploy to production.
- Test rollback through the approved GitOps process.

## 33. Chapter 183 completion criteria

Chapter 183 is complete when:

1. At least one application successfully invokes the shared workflow.
2. The required language-specific tests run.
3. Mandatory security checks are enforced.
4. The container build is reproducible enough for the organization's requirements and its inputs are traceable.
5. The publishing identity uses approved authentication and least-privilege permissions.
6. The image is published to the correct ECR repository.
7. The actual image digest is captured and verified.
8. Pull-request workflows cannot publish unauthorized production artifacts.
9. The GitOps change references the verified digest.
10. Workflow versions are controlled and documented.
11. Rollback and pipeline-failure procedures are documented.
12. The platform can onboard additional services without copying and maintaining an independent pipeline implementation for each one.

## 34. Key lessons

- Reusable workflows standardize complete CI processes.
- Composite actions standardize repeated steps.
- Application repositories retain responsibility for their own build and test requirements.
- Security gates must block releases when mandatory policies fail.
- OIDC provides short-lived AWS credentials when trust is configured correctly.
- ECR stores the container artifact.
- Image digests identify the specific artifact being promoted.
- GitOps controls the transition from a published artifact to an environment deployment.
- Versioned workflows allow controlled upgrades across all 21 microservices.

The key production principle is that a green pipeline must represent verified work, not merely successful execution of a few commands. The pipeline should prove that the intended source was tested, the resulting image met the required checks, and the exact published artifact is the one being promoted.

# Next: Chapter 184 — Terraform Modules

The next chapter will establish reusable Terraform modules for the AWS infrastructure supporting our 21 microservices, including:

- VPC and subnet modules.
- EKS cluster and node-group modules.
- ECR repository modules.
- IAM and workload identity modules.
- Remote state and locking.
- Module versioning and environment composition.
- Validation, security scanning, and CI.
- Infrastructure promotion and change approval.
- State recovery, drift detection, and troubleshooting.

This will connect the infrastructure foundation to the reusable Helm and GitHub Actions components already defined.