# CHAPTER 198 — ARTIFACT HANDOFF BETWEEN BUILD AND PUBLISH JOBS



Production implementation

GitHub Actions + AWS ECR



## 198.1 — Objective

In this chapter, we will implement a production-style artifact handoff between two separate CI/CD jobs.

Our requirement is simple:

Build the Docker image once, transfer the generated image artifact to another job, and publish that exact image to Amazon ECR without rebuilding it.

We will use the RoboShop microservices project as our example.

### What we are implementing

GitHub Repository

RoboShop application source code

JOB 1 — BUILD

- Checkout application source
- Build Docker image
- Add image metadata
- Export image as `image.tar`
- Generate SHA-256 checksum
- Upload artifact

GitHub Actions artifact storage

JOB 2 — PUBLISH

- Download `image.tar`
- Verify checksum
- Load the Docker image
- Authenticate with AWS using OIDC
- Tag the existing image for ECR
- Push image to ECR
- Capture the published image digest

Amazon ECR

Exact image built in Job 1

## 198.2 — Why do we need artifact handoff?

Consider a RoboShop application with 20+ microservices.

For example, the Product Catalog service has the following pipeline:

1. Developer pushes code to GitHub.
2. GitHub Actions starts the pipeline.
3. Docker image is built.
4. Security scanning is performed.
5. Image is published to ECR.
6. ArgoCD deploys the image to EKS.

A common mistake is to build the Docker image again in the publishing job.

This creates unnecessary work and can introduce inconsistencies.

### Without artifact handoff

```
BUILD JOB
   |
   | Build Docker image
   |
   | Image: product-catalog:abc123
   |
   X Job completed

PUBLISH JOB
   |
   | Checkout source code again
   |
   | Build Docker image AGAIN
   |
   | Image: product-catalog:abc123
   |
   | Push to ECR
```

The same commit is used, but the second build might not produce identical output if dependencies, base images, build arguments, or external inputs have changed.

### With artifact handoff

```
BUILD JOB
   |
   | Build Docker image ONCE
   |
   | Export image.tar
   |
   | Generate checksum
   |
   | Upload artifact
   |
   v
ARTIFACT STORAGE
   |
   | Download exact artifact
   |
   v
PUBLISH JOB
   |
   | Verify checksum
   |
   | Docker load
   |
   | Tag existing image
   |
   | Push to ECR
```

Important: `docker load` restores an existing image. It does not rebuild the image.

This gives us a clear separation between image creation and image publishing.

## 198.3 — Repository structure

We will organize this implementation inside the Product Catalog service repository.

```
roboshop-product-catalog/
│
├── src/
│   ├── package.json
│   ├── server.js
│   └── ...
│
├── Dockerfile
│
├── .dockerignore
│
├── scripts/
│   ├── build-image.sh
│   ├── export-image.sh
│   ├── verify-image.sh
│   └── publish-image.sh
│
├── .github/
│   └── workflows/
│       └── product-catalog-ci.yml
│
└── README.md
```

The four scripts separate the responsibilities of building, exporting, verifying, and publishing.

For this chapter, the GitHub Actions workflow will orchestrate the complete process.

## 198.4 — Define image naming conventions

Before implementing the workflow, we need a consistent naming convention.

| Attribute      | Example                                                            |
| -------------- | ------------------------------------------------------------------ |
| Service        | product-catalog                                                    |
| Docker image   | product-catalog                                                    |
| Commit SHA     | `a1b2c3d4...`                                                      |
| Build tag      | `a1b2c3d4...`                                                      |
| ECR repository | `roboshop/product-catalog`                                         |
| ECR image URI  | `ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/roboshop/product-catalog` |

We will use the Git commit SHA as the image tag.

Example:

```
product-catalog:a1b2c3d4e5f6
```

After publishing:

```
123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/product-catalog:a1b2c3d4e5f6
```

Why use commit SHA?

- Every image can be traced to its source commit.
- We avoid depending on a mutable `latest` tag.
- Rollbacks become easier.
- We can identify which application version is deployed.

For production, deploy using an image digest wherever possible.

## 198.5 — Configure the Dockerfile

We need a Dockerfile for the Product Catalog service.

Example Node.js Dockerfile:

```
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci --omit=dev

COPY . .

ENV NODE_ENV=production

EXPOSE 8080

USER node

CMD ["node", "server.js"]
```

Example `.dockerignore`:

```
node_modules
.git
.github
.env
*.log
coverage
Dockerfile*
README.md
```

Production note: Use a trusted, pinned base image version or digest, and update it through a controlled dependency process.

## 198.6 — Create the build script

File:

```
scripts/build-image.sh
```

Implementation:

```
#!/bin/bash

set -euo pipefail

SERVICE="product-catalog"
IMAGE_TAG="${GITHUB_SHA:?GITHUB_SHA is required}"

LOCAL_IMAGE="${SERVICE}:${IMAGE_TAG}"

echo "======================================"
echo "Starting Docker image build"
echo "======================================"

echo "Service: ${SERVICE}"
echo "Image: ${LOCAL_IMAGE}"

docker build \
  --pull \
  --label "org.opencontainers.image.title=${SERVICE}" \
  --label "org.opencontainers.image.revision=${GITHUB_SHA}" \
  --label "org.opencontainers.image.source=${GITHUB_SERVER_URL}/${GITHUB_REPOSITORY}" \
  --tag "${LOCAL_IMAGE}" \
  .

echo "Docker image built successfully"

docker image inspect "${LOCAL_IMAGE}"

echo "Build completed"
```

### Explanation

`set -euo pipefail` ensures that the script stops when a command fails, an unset variable is referenced, or a pipeline command fails.

`GITHUB_SHA` contains the commit SHA provided by GitHub Actions.

`docker build` creates the Docker image.

The labels attach useful metadata to the image:

- Image title.
- Source commit.
- Source repository.

We can later inspect this metadata to verify the image's origin.

## 198.7 — Export the Docker image

File:

```
scripts/export-image.sh
```

Implementation:

```
#!/bin/bash

set -euo pipefail

SERVICE="product-catalog"
IMAGE_TAG="${GITHUB_SHA:?GITHUB_SHA is required}"

IMAGE="${SERVICE}:${IMAGE_TAG}"

ARTIFACT_DIR="image-artifact"

mkdir -p "${ARTIFACT_DIR}"

echo "Exporting Docker image..."

docker save \
  --output "${ARTIFACT_DIR}/image.tar" \
  "${IMAGE}"

echo "Generating SHA256 checksum..."

cd "${ARTIFACT_DIR}"

sha256sum image.tar > image.tar.sha256

echo "Generating image metadata..."

cat > metadata.txt <<EOF
SERVICE=${SERVICE}
IMAGE=${IMAGE}
COMMIT_SHA=${GITHUB_SHA}
REPOSITORY=${GITHUB_REPOSITORY}
EOF

echo "Artifact export completed"

ls -lh
```

The resulting directory will look like this:

```
image-artifact/
│
├── image.tar
├── image.tar.sha256
└── metadata.txt
```

### What does docker save do?

```
docker save \
  --output image.tar \
  product-catalog:a1b2c3d4
```

It exports the Docker image, including its layers and configuration, into a tar archive.

This archive can be transferred to another machine or GitHub Actions job.

### What does sha256sum do?

```
sha256sum image.tar > image.tar.sha256
```

It calculates a SHA-256 checksum for the archive.

We will use this checksum in the publishing job to detect accidental corruption during artifact transfer.

Security note: A checksum stored alongside the archive is an integrity check against accidental changes, not proof against a malicious actor who can modify both files. For stronger provenance, use signed artifacts or image attestations.

## 198.8 — Configure AWS ECR using Terraform

Before publishing images, we need an ECR repository.

We will create it using Terraform.

Repository structure:

```
terraform/
│
├── modules/
│   └── ecr/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
└── environments/
    └── production/
        └── ecr.tf
```

### 198.8.1 — ECR module

File:

```
terraform/modules/ecr/main.tf
```

```
resource "aws_ecr_repository" "this" {
  name                 = var.repository_name
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "AES256"
  }

  tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Application = "RoboShop"
  }
}
```

File:

```
terraform/modules/ecr/variables.tf
```

```
variable "repository_name" {
  description = "ECR repository name"
  type        = string
}

variable "environment" {
  description = "Deployment environment"
  type        = string
}
```

File:

```
terraform/modules/ecr/outputs.tf
```

```
output "repository_url" {
  value = aws_ecr_repository.this.repository_url
}

output "repository_arn" {
  value = aws_ecr_repository.this.arn
}
```

### 198.8.2 — Create the production ECR repository

File:

```
terraform/environments/production/ecr.tf
```

```
module "product_catalog_ecr" {
  source = "../../modules/ecr"

  repository_name = "roboshop/product-catalog"
  environment     = "production"
}
```

Run:

```
terraform init

terraform plan

terraform apply
```

Verify:

```
aws ecr describe-repositories \
  --repository-names roboshop/product-catalog \
  --region ap-south-1
```

### Why immutable tags?

We configured:

```
image_tag_mutability = "IMMUTABLE"
```

This prevents an existing tag from being overwritten with another image.

For example, if the tag is:

```
product-catalog:a1b2c3d4
```

A subsequent push attempting to overwrite that tag should be rejected by ECR.

This is useful for maintaining reliable image history.

## 198.9 — Configure GitHub Actions authentication with AWS OIDC

Our publishing job needs permission to push images into ECR.

There are two common approaches:

1. Store AWS access keys in GitHub Secrets.
2. Use GitHub Actions OIDC to assume an AWS IAM role.

For production, we will use OIDC.

### Why OIDC?

With OIDC:

- GitHub Actions requests a short-lived identity token.
- AWS validates the token.
- AWS allows the workflow to assume a specific IAM role.
- Temporary AWS credentials are provided to the job.

We do not need to store permanent AWS access keys in GitHub.

### 198.9.1 — Create the GitHub OIDC provider

In AWS IAM, configure the GitHub Actions OIDC identity provider:

```
Provider URL:
https://token.actions.githubusercontent.com
```

Audience:

```
sts.amazonaws.com
```

If the provider already exists in your AWS account, reuse it instead of creating another.

### 198.9.2 — Create the IAM trust policy

Create a trust policy for the GitHub Actions publishing role.

Example:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/roboshop-product-catalog:environment:ecr-publish"
        }
      }
    }
  ]
}
```

Replace:

- `123456789012` with your AWS account ID.
- `YOUR_ORG` with your GitHub organization or username.

The subject condition restricts this role to the specified repository and GitHub environment.

Create a GitHub environment named:

```
ecr-publish
```

You can also configure environment protection rules, such as required approvals, depending on your production release policy.

### 198.9.3 — Create the ECR push policy

Create an IAM permissions policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRAuthentication",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ECRPushAndInspect",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage",
        "ecr:BatchGetImage",
        "ecr:DescribeImages"
      ],
      "Resource": "arn:aws:ecr:ap-south-1:123456789012:repository/roboshop/product-catalog"
    }
  ]
}
```

Attach this policy to the IAM role.

The role should have only the permissions required to publish and inspect images in the specified ECR repository.

## 198.10 — Create the verification script

File:

```
scripts/verify-image.sh
```

```
#!/bin/bash

set -euo pipefail

ARTIFACT_DIR="image-artifact"

cd "${ARTIFACT_DIR}"

echo "Verifying image artifact checksum..."

sha256sum --check image.tar.sha256

echo "Checksum verification successful"

echo "Loading Docker image..."

docker load --input image.tar

echo "Docker image loaded"

echo "Reading metadata..."

cat metadata.txt

IMAGE="product-catalog:${GITHUB_SHA:?GITHUB_SHA is required}"

echo "Verifying image exists..."

docker image inspect "${IMAGE}" >/dev/null

echo "Verifying source commit label..."

IMAGE_COMMIT=$(docker image inspect \
  --format '{{ index .Config.Labels "org.opencontainers.image.revision" }}' \
  "${IMAGE}")

if [ "${IMAGE_COMMIT}" != "${GITHUB_SHA}" ]; then
    echo "ERROR: Image commit does not match workflow commit"
    exit 1
fi

echo "Image verification successful"
```

### What are we verifying?

We are checking three things:

1. The downloaded archive has the expected checksum.
2. Docker can successfully load the image.
3. The image contains the expected Git commit SHA.

This helps catch accidental artifact corruption or mismatched build artifacts.

## 198.11 — Create the publishing script

File:

```
scripts/publish-image.sh
```

```
#!/bin/bash

set -euo pipefail

AWS_REGION="${AWS_REGION:?AWS_REGION is required}"
AWS_ACCOUNT_ID="${AWS_ACCOUNT_ID:?AWS_ACCOUNT_ID is required}"

SERVICE="product-catalog"

IMAGE_TAG="${GITHUB_SHA:?GITHUB_SHA is required}"

ECR_REPOSITORY="roboshop/product-catalog"

ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

LOCAL_IMAGE="${SERVICE}:${IMAGE_TAG}"

ECR_IMAGE="${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"

echo "Authenticating with Amazon ECR..."

aws ecr get-login-password \
  --region "${AWS_REGION}" |
  docker login \
    --username AWS \
    --password-stdin "${ECR_REGISTRY}"

echo "Tagging existing Docker image..."

docker tag "${LOCAL_IMAGE}" "${ECR_IMAGE}"

echo "Publishing image..."

docker push "${ECR_IMAGE}"

echo "Retrieving published image digest..."

IMAGE_DIGEST=$(aws ecr describe-images \
  --repository-name "${ECR_REPOSITORY}" \
  --image-ids imageTag="${IMAGE_TAG}" \
  --region "${AWS_REGION}" \
  --query 'imageDetails[0].imageDigest' \
  --output text)

if [[ ! "${IMAGE_DIGEST}" =~ ^sha256:[a-f0-9]{64}$ ]]; then
    echo "ERROR: Unable to retrieve a valid image digest"
    exit 1
fi

echo "======================================"
echo "Image published successfully"
echo "======================================"

echo "Image: ${ECR_IMAGE}"
echo "Digest: ${IMAGE_DIGEST}"

echo "IMAGE_URI=${ECR_REGISTRY}/${ECR_REPOSITORY}@${IMAGE_DIGEST}" \
  >> "${GITHUB_OUTPUT:-/dev/null}"
```

Important: This script assumes the image has already been loaded and verified. It does not run `docker build`.

The ECR digest identifies the published image manifest. We will use it later as a deployment reference.

## 198.12 — Implement the complete GitHub Actions workflow

Now we connect everything.

File:

```
.github/workflows/product-catalog-ci.yml
```

```

name: Product Catalog CI - Build and Publish

on:
  push:
    branches:
      - main

  workflow_dispatch:

permissions:
  contents: read

env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: roboshop/product-catalog
  SERVICE_NAME: product-catalog

jobs:

  # =====================================
  # JOB 1 - BUILD AND EXPORT IMAGE
  # =====================================

  build:
    name: Build and Export Docker Image

    runs-on: ubuntu-latest

    outputs:
      image_tag: ${{ steps.metadata.outputs.image_tag }}

    steps:

      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Set image metadata
        id: metadata
        run: |
          echo "image_tag=${GITHUB_SHA}" >> "$GITHUB_OUTPUT"

      - name: Build Docker image
        run: |
          chmod +x scripts/*.sh
          ./scripts/build-image.sh

      - name: Export Docker image
        run: |
          ./scripts/export-image.sh

      - name: Upload Docker image artifact
        uses: actions/upload-artifact@v4
        with:
          name: product-catalog-image
          path: image-artifact/
          if-no-files-found: error
          retention-days: 1

  # =====================================
  # JOB 2 - VERIFY AND PUBLISH TO ECR
  # =====================================

  publish:
    name: Verify and Publish to ECR

    needs: build

    runs-on: ubuntu-latest

    environment: ecr-publish

    permissions:
      contents: read
      actions: read
      id-token: write

    outputs:
      image_uri: ${{ steps.publish.outputs.image_uri }}
      image_digest: ${{ steps.publish.outputs.image_digest }}

    steps:

      - name: Download Docker image artifact
        uses: actions/download-artifact@v4
        with:
          name: product-catalog-image
          path: image-artifact

      - name: Verify and load image
        run: |
          chmod +x scripts/*.sh
          ./scripts/verify-image.sh

      - name: Configure AWS credentials using OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ECR_PUBLISH_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Verify AWS identity
        run: |
          aws sts get-caller-identity

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Tag and publish Docker image
        id: publish
        env:
          AWS_ACCOUNT_ID: ${{ vars.AWS_ACCOUNT_ID }}
          IMAGE_TAG: ${{ needs.build.outputs.image_tag }}
        run: |
          set -euo pipefail

          ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

          LOCAL_IMAGE="${SERVICE_NAME}:${IMAGE_TAG}"

          ECR_IMAGE="${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"

          docker tag "${LOCAL_IMAGE}" "${ECR_IMAGE}"

          docker push "${ECR_IMAGE}"

          IMAGE_DIGEST=$(aws ecr describe-images \
            --repository-name "${ECR_REPOSITORY}" \
            --image-ids imageTag="${IMAGE_TAG}" \
            --region "${AWS_REGION}" \
            --query 'imageDetails[0].imageDigest' \
            --output text)

          if [[ ! "${IMAGE_DIGEST}" =~ ^sha256:[a-f0-9]{64}$ ]]; then
            echo "Invalid ECR image digest"
            exit 1
          fi

          IMAGE_URI="${ECR_REGISTRY}/${ECR_REPOSITORY}@${IMAGE_DIGEST}"

          echo "image_uri=${IMAGE_URI}" >> "$GITHUB_OUTPUT"
          echo "image_digest=${IMAGE_DIGEST}" >> "$GITHUB_OUTPUT"

          echo "Published image: ${IMAGE_URI}"

      - name: Display published image
        run: |
          echo "Image URI: ${{ steps.publish.outputs.image_uri }}"
          echo "Image Digest: ${{ steps.publish.outputs.image_digest }}"

```

### Important workflow correction for script-based verification

In the example above, the publish job uses `scripts/verify-image.sh`, but it has not checked out the repository containing that script.

There are two ways to solve this:

- Check out the repository in the publish job.
- Package the verification script into the artifact.

For this implementation, use checkout because the scripts are maintained in the Git repository.

Add this step at the beginning of the `publish` job, before downloading the artifact:

```
- name: Checkout repository scripts
  uses: actions/checkout@v4
```

The final publishing job will therefore have both:

- The scripts from the Git repository.
- The image artifact downloaded from the build job.

In a production repository, pin third-party actions to verified full commit SHAs rather than relying only on version tags.

## 198.13 — Configure GitHub repository variables

Go to:

```
GitHub Repository
    |
    Settings
    |
    Secrets and variables
    |
    Actions
    |
    Variables
```

Add the following repository variables:

| Variable                   | Example                                             |
| -------------------------- | --------------------------------------------------- |
| `AWS_ACCOUNT_ID`           | `123456789012`                                      |
| `AWS_ECR_PUBLISH_ROLE_ARN` | `arn:aws:iam::123456789012:role/github-ecr-publish` |

These are configuration values, not permanent AWS access keys.

Do not add AWS access keys when using OIDC.

## 198.14 — Execute the pipeline

Push your changes:

```
git add .

git commit -m "Implement Docker artifact handoff"

git push origin main
```

Open:

```
GitHub Repository
    |
    Actions
    |
    Product Catalog CI - Build and Publish
```

You should see two jobs.

Build and Export Docker Image

Job 1

- Checkout source code
- Build Docker image
- Export `image.tar`
- Generate checksum
- Upload artifact

Verify and Publish to ECR

Job 2

- Checkout verification scripts
- Download artifact
- Verify checksum
- Load Docker image
- Assume AWS IAM role
- Push image to ECR
- Capture image digest

## 198.15 — Verify the artifact handoff

We need to confirm that the image published to ECR is the same image built in Job 1.

### Step 1: Check the GitHub Actions artifact

Open the workflow run and inspect the uploaded artifact:

```
product-catalog-image
```

Expected contents:

```
image.tar
image.tar.sha256
metadata.txt
```

### Step 2: Check the downloaded artifact

In the publish job logs, confirm:

```
Checksum verification successful
Docker image loaded
Image verification successful
```

### Step 3: Check the local Docker image

Run:

```
docker images
```

Expected example:

```
REPOSITORY          TAG
product-catalog     a1b2c3d4e5f6
```

### Step 4: Verify image metadata

Run:

```
docker image inspect \
  product-catalog:a1b2c3d4e5f6 \
  --format '{{ index .Config.Labels "org.opencontainers.image.revision" }}'
```

Expected:

```
a1b2c3d4e5f6...
```

### Step 5: Verify the image in ECR

Run:

```
aws ecr describe-images \
  --repository-name roboshop/product-catalog \
  --region ap-south-1
```

You should see the commit SHA tag and the corresponding image digest.

### Step 6: Pull the published image

```
docker pull \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/product-catalog:a1b2c3d4e5f6
```

Inspect the image:

```
docker image inspect \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/product-catalog:a1b2c3d4e5f6
```

We have now confirmed that the image can be retrieved from ECR.

## 198.16 — Production failure scenarios

A production DevOps engineer must understand not only the successful pipeline but also how to recover when individual stages fail.

### Scenario 1 — Docker build fails

Possible reasons:

- Dockerfile syntax issue.
- Missing package dependencies.
- Incorrect build context.
- Network issue while downloading dependencies.
- Base image unavailable.

Troubleshooting:

```
docker build --no-cache -t product-catalog:test .
```

Check the Dockerfile and build logs.

Do not proceed to artifact export if the build fails.

### Scenario 2 — Artifact upload fails

Possible reasons:

- Artifact path is incorrect.
- Image archive was not created.
- Artifact upload action failed.
- Runner encountered storage or network problems.

Check:

```
ls -lh image-artifact/
```

Expected:

```
image.tar
image.tar.sha256
metadata.txt
```

If files are missing, fix the export stage before retrying.

### Scenario 3 — Artifact download fails

Possible reasons:

- Incorrect artifact name.
- Build job failed.
- Publish job did not depend on the correct build job.
- Artifact expired or was unavailable.

Check:

```
needs: build
```

And:

```
with:
  name: product-catalog-image
```

The upload and download names must match.

### Scenario 4 — Checksum verification fails

Example:

```
image.tar: FAILED
```

This means the archive does not match the checksum file.

Actions:

1. Stop the publishing job.
2. Do not push the image.
3. Investigate artifact handling.
4. Re-run the build and artifact transfer.
5. Preserve logs for investigation.

Never skip checksum verification just to make the pipeline green.

### Scenario 5 — Docker load fails

Run:

```
docker load --input image-artifact/image.tar
```

Check whether:

- The archive is incomplete.
- The downloaded file is actually a tar archive.
- The Docker daemon is available.
- The archive was generated successfully.

### Scenario 6 — AWS OIDC authentication fails

Common errors:

```
Not authorized to perform sts:AssumeRoleWithWebIdentity
```

Check:

- OIDC provider exists in AWS IAM.
- Audience is `sts.amazonaws.com`.
- GitHub repository name matches the trust policy.
- GitHub environment name matches the trust policy.
- Workflow has `id-token: write`.
- IAM role ARN is correct.

The most common issue is a mismatch between the IAM trust policy's `sub` condition and the actual GitHub Actions identity.

### Scenario 7 — ECR push fails

Example:

```
AccessDeniedException
```

Check:

- IAM role permissions.
- AWS account ID.
- Region.
- Repository name.
- ECR authentication.
- Repository tag immutability.

If an immutable tag already exists, do not attempt to overwrite it. Investigate whether this is an intentional retry or a conflicting build.

### Scenario 8 — Image digest retrieval fails

Check:

```
aws ecr describe-images \
  --repository-name roboshop/product-catalog \
  --region ap-south-1
```

Verify that the push completed and the expected tag exists.

Do not trigger deployment until the published digest is confirmed.

## 198.17 — Production optimization and security considerations

The implementation above is a working foundation. For a real production environment, consider these improvements.

| Area                  | Production approach                             |
| --------------------- | ----------------------------------------------- |
| GitHub Actions        | Pin actions to verified commit SHAs             |
| Authentication        | AWS OIDC with restricted trust policy           |
| IAM                   | Repository-specific least-privilege permissions |
| Image tags            | Commit SHA and immutable tags                   |
| Image scanning        | ECR scanning and CI security gates              |
| Artifact retention    | Keep artifacts for the required release window  |
| Image provenance      | Signed attestations and provenance              |
| Deployment            | Use ECR image digest                            |
| Build reproducibility | Pin base images and dependencies                |
| Secrets               | Use approved secret-management solutions        |
| Concurrency           | Prevent conflicting production publishing runs  |

### Important: Artifact storage versus container registry

We now have two separate storage locations.

| GitHub Actions artifact      | Amazon ECR                   |
| ---------------------------- | ---------------------------- |
| Temporary build handoff      | Container image registry     |
| Transfers image between jobs | Stores published images      |
| Has configurable retention   | Has image lifecycle policies |
| Used by CI/CD workflow       | Used by deployment systems   |

We should not use GitHub Actions artifacts as a replacement for ECR.

The artifact is used to transfer the image between jobs. ECR is where the deployment platform retrieves the published image.

## 198.18 — How this connects to Kubernetes and ArgoCD

After publishing the image, our next stage will be deployment.

For example, the published image might be:

```
123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/product-catalog@sha256:abc123...
```

We can update the Kubernetes deployment to use that digest.

Example:

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: product-catalog
  namespace: roboshop

spec:
  replicas: 3

  selector:
    matchLabels:
      app: product-catalog

  template:
    metadata:
      labels:
        app: product-catalog

    spec:
      containers:
        - name: product-catalog

          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/roboshop/product-catalog@sha256:abc123...

          ports:
            - containerPort: 8080
```

The digest in this example is illustrative. In the real pipeline, we use the digest returned by ECR.

### GitOps deployment flow

```
Developer pushes code
        |
        v
GitHub Actions
        |
        v
Build Docker image
        |
        v
Export image artifact
        |
        v
Publish exact image to ECR
        |
        v
Capture image digest
        |
        v
Update Kubernetes deployment manifest
        |
        v
Push manifest changes to GitOps repository
        |
        v
ArgoCD detects Git changes
        |
        v
ArgoCD syncs the application
        |
        v
Amazon EKS
        |
        v
New Product Catalog pods
```

Important separation of responsibilities:

- GitHub Actions builds and publishes the image.
- The GitOps repository stores the desired Kubernetes configuration.
- ArgoCD reconciles the cluster with the GitOps repository.
- EKS runs the application.

The publish job should not directly modify the live Kubernetes cluster when we are following a GitOps deployment model.

## 198.19 — Scaling this implementation to 20+ microservices

We should not create an entirely different pipeline implementation for every microservice.

Instead, we can reuse the same workflow structure with service-specific configuration.

Example:

| Microservice    | ECR repository             |
| --------------- | -------------------------- |
| User            | `roboshop/user`            |
| Product Catalog | `roboshop/product-catalog` |
| Cart            | `roboshop/cart`            |
| Orders          | `roboshop/orders`          |
| Payment         | `roboshop/payment`         |
| Inventory       | `roboshop/inventory`       |
| Notification    | `roboshop/notification`    |

For 20+ services, we can introduce:

- Reusable GitHub Actions workflows.
- Service-specific build configurations.
- Separate ECR repositories.
- Shared artifact verification logic.
- Consistent image tagging.
- Independent release pipelines.

This avoids maintaining 20 different copies of the same CI/CD logic.

## 198.20 — Interview preparation

These are the questions you should be able to answer after completing this chapter.

1\. Why do we use artifact handoff between build and publish jobs?

We build the Docker image once and transfer the exact image to another job. This avoids rebuilding and reduces inconsistencies between build and publish stages.

2\. What is the difference between `docker save` and `docker load`?

`docker save` exports a Docker image into an archive. `docker load` imports that archive and restores the image.

3\. Why don't we build the image again in the publishing job?

Because the publishing job should publish the image already built and tested in the previous job. Rebuilding could produce a different image.

4\. Why do we use SHA-256 checksums?

To detect accidental corruption or changes to the transferred image archive.

5\. How does GitHub Actions authenticate with AWS without access keys?

Using OIDC. GitHub Actions obtains an identity token, AWS validates it against the configured trust policy, and the workflow assumes an IAM role with temporary credentials.

6\. Why do we use commit SHA as the image tag?

It helps identify the exact source-code revision used to create the image and makes image tracking and rollback easier.

7\. What happens if the artifact is corrupted?

The checksum verification fails, and the publishing job must stop before pushing the image to ECR.

8\. What is the difference between an image tag and an image digest?

A tag is a human-readable reference to an image. A digest identifies a specific image manifest by its content hash.

9\. Why do we use immutable ECR tags?

To prevent an existing image tag from being overwritten by another image.

10\. How do you deploy the exact image that was published?

Capture the ECR digest and use the digest in the Kubernetes deployment manifest.

## 198.21 — Final implementation checklist

Chapter 198 progress

# 0%

0 of 19 tasks completed

Create the Product Catalog Dockerfile

Create build-image.sh

Create export-image.sh

Generate SHA-256 checksum

Create ECR Terraform module

Provision the Product Catalog ECR repository

Configure AWS GitHub OIDC provider

Create restricted IAM trust policy

Create ECR push permissions policy

Configure GitHub environment and variables

Create verify-image.sh

Create publish-image.sh

Implement two-job GitHub Actions workflow

Upload and download image artifact

Verify checksum and image metadata

Publish image to ECR

Capture image digest

Verify the image in ECR

Prepare the digest for GitOps deployment

## 198.22 — Expected outcome

After completing this chapter, you should be able to explain and implement the following production workflow:

- Build a Docker image once.
- Export the image as an artifact.
- Transfer the artifact between separate GitHub Actions jobs.
- Verify artifact integrity.
- Authenticate with AWS through OIDC.
- Publish the existing image to Amazon ECR.
- Capture the image digest.
- Prepare the exact image for a GitOps deployment.

The main production principle: Build once, verify once, publish the same artifact, and deploy the exact image digest.

This completes CHAPTER 198 — ARTIFACT HANDOFF BETWEEN BUILD AND PUBLISH JOBS.