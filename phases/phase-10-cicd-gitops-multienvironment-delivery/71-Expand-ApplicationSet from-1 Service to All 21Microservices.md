# 71 — Expand ApplicationSet from 1 Service to All 21 Microservices

Now that you understand and can test:

```text
ApplicationSet
      ↓
order-service-dev
      ↓
ArgoCD
      ↓
EKS
```

we will expand the same mechanism to **all 21 microservices**.

The important thing is: **we are not creating 84 Application YAML files manually.**

---

# 1. What We Have Now

Currently:

```text
DEV
└── order-service
    ├── app.yaml
    └── values.yaml
```

ApplicationSet watches:

```text
environments/dev/*/app.yaml
```

Therefore it finds:

```text
order-service/app.yaml
```

and creates:

```text
order-service-dev
```

---

# 2. What We Want

Eventually:

```text
DEV
├── auth-service
├── user-service
├── customer-service
├── catalog-service
├── product-service
├── inventory-service
├── pricing-service
├── promotion-service
├── cart-service
├── order-service
├── payment-service
├── fraud-service
├── shipping-service
├── delivery-service
├── notification-service
├── search-service
├── recommendation-service
├── review-service
├── tax-service
├── invoice-service
└── reporting-service
```

ApplicationSet automatically creates:

```text
auth-service-dev
user-service-dev
customer-service-dev
...
reporting-service-dev
```

---

# 3. The 21 Services

We will use this exact list:

```text
auth-service
user-service
customer-service
catalog-service
product-service
inventory-service
pricing-service
promotion-service
cart-service
order-service
payment-service
fraud-service
shipping-service
delivery-service
notification-service
search-service
recommendation-service
review-service
tax-service
invoice-service
reporting-service
```

---

# 4. Create the Directory Structure

From the GitOps repository:

```bash
cd gitops
```

Create all 21 directories:

```bash
mkdir -p environments/dev/{auth-service,user-service,customer-service,catalog-service,product-service,inventory-service,pricing-service,promotion-service,cart-service,order-service,payment-service,fraud-service,shipping-service,delivery-service,notification-service,search-service,recommendation-service,review-service,tax-service,invoice-service,reporting-service}
```

You already have:

```text
order-service/
```

so that directory already exists.

---

# 5. Don't Create Everything Manually

For a real project, don't sit and type 42 files manually.

Create a small scaffolding script.

Create:

```text
scripts/bootstrap-dev-services.sh
```

```bash
#!/bin/bash

set -e

SERVICES=(
  auth-service
  user-service
  customer-service
  catalog-service
  product-service
  inventory-service
  pricing-service
  promotion-service
  cart-service
  order-service
  payment-service
  fraud-service
  shipping-service
  delivery-service
  notification-service
  search-service
  recommendation-service
  review-service
  tax-service
  invoice-service
  reporting-service
)

for SERVICE in "${SERVICES[@]}"; do

    DIR="environments/dev/$SERVICE"

    mkdir -p "$DIR"

    if [ ! -f "$DIR/app.yaml" ]; then

        cat > "$DIR/app.yaml" <<EOF
service: $SERVICE

repository: https://github.com/company-ecommerce/$SERVICE.git

chartPath: helm/$SERVICE

chartRevision: main

enabled: true

team: backend-team
EOF

    fi

    if [ ! -f "$DIR/values.yaml" ]; then

        cat > "$DIR/values.yaml" <<EOF
replicaCount: 2

image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/$SERVICE
  tag: "REPLACE_ME"
  pullPolicy: IfNotPresent

resources:
  requests:
    cpu: 250m
    memory: 512Mi

  limits:
    cpu: 1000m
    memory: 1Gi

hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70

pdb:
  enabled: true
  minAvailable: 1
EOF

    fi

done

echo "DEV service configuration created."
```

Make executable:

```bash
chmod +x scripts/bootstrap-dev-services.sh
```

Run:

```bash
./scripts/bootstrap-dev-services.sh
```

---

# 6. What Did the Script Actually Do?

It created:

```text
environments/dev/
├── auth-service/
│   ├── app.yaml
│   └── values.yaml
│
├── user-service/
│   ├── app.yaml
│   └── values.yaml
│
├── customer-service/
│   ├── app.yaml
│   └── values.yaml
│
├── catalog-service/
│   ├── app.yaml
│   └── values.yaml
│
...
│
└── reporting-service/
    ├── app.yaml
    └── values.yaml
```

Therefore our ApplicationSet now sees:

```text
environments/dev/*/app.yaml
```

and finds all 21 services.

---

# 7. But There Is an Important Problem

Our script creates:

```yaml
image:
  tag: "REPLACE_ME"
```

That means those services cannot actually deploy yet.

Why?

Because we haven't built their Docker images.

For example:

```text
auth-service
```

needs an actual ECR image:

```text
auth-service:<version>
```

Similarly:

```text
payment-service
```

needs:

```text
payment-service:<version>
```

So **ApplicationSet does not magically create the applications' Docker images.**

CI must do that.

---

# 8. This Gives Us an Important Architecture

Each service has its own lifecycle:

```text
auth-service
   ↓
CI
   ↓
ECR
   ↓
GitOps
   ↓
ApplicationSet
   ↓
ArgoCD
   ↓
EKS
```

Same for:

```text
order-service
user-service
payment-service
...
```

---

# 9. ApplicationSet Doesn't Care About the Image

ApplicationSet sees:

```yaml
service: order-service
```

and creates:

```text
order-service-dev
```

It doesn't care whether the image is:

```text
order-service:1.0.0
```

or:

```text
order-service:8f31a2c
```

That is Helm's responsibility.

The values file tells Helm which image to deploy.

---

# 10. Verify the ApplicationSet

After pushing the new directories:

```bash
git add environments/dev scripts/
```

Commit:

```bash
git commit -m "feat: add dev configuration for microservices"
```

Push:

```bash
git push origin main
```

Wait for ApplicationSet reconciliation.

Then:

```bash
kubectl get applications -n argocd
```

You should start seeing:

```text
auth-service-dev
user-service-dev
customer-service-dev
catalog-service-dev
product-service-dev
...
order-service-dev
...
reporting-service-dev
```

---

# 11. Important: Don't Expect All 21 to Become Healthy

This is very important.

At this stage, we may see:

```text
auth-service-dev       Missing
user-service-dev       Missing
customer-service-dev   Missing
...
order-service-dev      Healthy
```

Why?

Because only `order-service` has actually been implemented.

We created the **GitOps metadata** for the other services, but we haven't necessarily created:

```text
auth-service repository
user-service repository
customer-service repository
...
```

with their Helm charts and images.

---

# 12. This Is Actually Good

It proves something important:

ApplicationSet is doing its job.

It created:

```text
auth-service-dev
```

even though the actual application isn't ready.

The problem is now with the application repository/image/chart—not ApplicationSet.

---

# 13. Application Lifecycle

For each new service, we need to complete these steps:

```text
1. Create application repository
          ↓
2. Write application code
          ↓
3. Create Dockerfile
          ↓
4. Create Helm chart
          ↓
5. Create ECR repository
          ↓
6. Create CI pipeline
          ↓
7. Build image
          ↓
8. Push image to ECR
          ↓
9. Update GitOps values
          ↓
10. ApplicationSet generates Application
          ↓
11. ArgoCD deploys
          ↓
12. Verify Healthy
```

This is the real production workflow.

---

# 14. One Important Improvement

We should not allow an incomplete service configuration to create an Application.

For example:

```yaml
enabled: false
```

could mean:

> Don't deploy this service yet.

So:

```yaml
service: payment-service

repository: https://github.com/company-ecommerce/payment-service.git

chartPath: helm/payment-service

chartRevision: main

enabled: false

team: backend-team
```

Then our ApplicationSet should eventually filter on:

```text
enabled == true
```

This is useful while we're building the 21 services gradually.

---

# 15. Why Do We Need This?

Imagine we have:

```text
21 services
```

but only:

```text
3 services ready
```

We don't want ArgoCD trying to deploy:

```text
18 unfinished services
```

So we can have:

```text
auth-service        enabled: true
user-service        enabled: true
order-service       enabled: true

payment-service     enabled: false
fraud-service       enabled: false
...
```

This lets us gradually onboard services.

---

# 16. A More Production-Friendly Structure

I would use:

```text
environments/dev/order-service/
├── app.yaml
└── values.yaml
```

where:

```yaml
enabled: true
```

means:

```text
This service should exist in DEV.
```

And:

```yaml
enabled: false
```

means:

```text
This service is not onboarded yet.
```

---

# 17. But There Is a Catch

The basic Git file generator:

```yaml
files:
  - path: environments/dev/*/app.yaml
```

finds the file.

Filtering based on `enabled` requires the ApplicationSet generator/template strategy to be designed carefully.

So **don't modify our working ApplicationSet yet**.

First prove all 21 Applications can be generated.

Then we'll add service enable/disable control safely.

---

# 18. What Should We Do With the 21 Applications?

We should not deploy all 21 immediately.

Our project implementation should progress like this:

```text
Phase 1
order-service
      ↓
DONE

Phase 2
user-service
      ↓
DONE

Phase 3
catalog-service
      ↓
product-service
      ↓
inventory-service
      ↓
...

Phase 4
remaining services
```

The infrastructure pattern remains identical.

---

# 19. Environment Promotion

Now let's understand something very important.

Suppose:

```text
order-service
```

has:

```text
image: 8f31a2c
```

in DEV.

After testing, we want the **same image** in QA.

Not:

```text
DEV → build image A
QA → build image B
```

Instead:

```text
Build once

8f31a2c
   ↓
DEV
   ↓
QA
   ↓
STAGING
   ↓
PROD
```

This is the principle:

> **Build once, promote the same immutable artifact.**

---

# 20. Example

DEV:

```yaml
image:
  tag: "8f31a2c"
```

After testing:

```text
DEV
 ↓
QA
```

QA:

```yaml
image:
  tag: "8f31a2c"
```

After staging:

```text
STAGING
```

```yaml
image:
  tag: "8f31a2c"
```

Production:

```yaml
image:
  tag: "8f31a2c"
```

Same image.

---

# 21. Why Is This Important?

Suppose we do:

```text
DEV → build image A
QA → build image B
PROD → build image C
```

Then production isn't necessarily running the exact artifact that was tested in DEV.

Instead:

```text
Source
  ↓
Build
  ↓
Image A
  ↓
Scan
  ↓
Test
  ↓
Promote
```

is much easier to trace.

---

# 22. Where ApplicationSet Fits

ApplicationSet doesn't decide:

```text
Should this image go to QA?
```

Our promotion process decides that.

ApplicationSet simply ensures:

```text
DEV has an Application for every enabled service.

QA has an Application for every enabled service.

STAGING has an Application for every enabled service.

PROD has an Application for every enabled service.
```

---

# 23. Four ApplicationSets

Eventually:

```text
                    GitOps
                       |
       +---------------+---------------+
       |               |               |
       v               v               v
  ecommerce-dev   ecommerce-qa   ecommerce-staging
       |               |               |
       +---------------+---------------+
                       |
                       v
                 ecommerce-prod
```

Each ApplicationSet watches its environment.

---

# 24. Why Not One Giant ApplicationSet?

You technically can build more sophisticated generators.

For learning and production maintainability, I prefer:

```text
ecommerce-dev
ecommerce-qa
ecommerce-staging
ecommerce-prod
```

because each environment has different:

```text
sync policy
namespace
AppProject
approval model
chart revision strategy
risk level
```

It's much easier to troubleshoot.

---

# 25. What We Have Achieved

At this point our architecture has progressed from:

```text
One service
      ↓
One ArgoCD Application
```

to:

```text
21 services
      ↓
ApplicationSet
      ↓
21 generated DEV Applications
```

And eventually:

```text
21 services
×
4 environments
      ↓
ApplicationSets
      ↓
84 ArgoCD Applications
```

---

# 26. The Most Important Mental Model

Think about the system in layers:

```text
LAYER 1 — APPLICATION

order-service
user-service
payment-service
...


LAYER 2 — CONTAINER

Docker image
      ↓
ECR


LAYER 3 — CONFIGURATION

GitOps
      ↓
DEV / QA / STAGING / PROD


LAYER 4 — APPLICATION CREATION

ApplicationSet
      ↓
ArgoCD Applications


LAYER 5 — DEPLOYMENT

ArgoCD
      ↓
Kubernetes


LAYER 6 — RUNTIME

EKS
      ↓
Pods
```

Each layer has one job.

---

# 27. Where We Go Next

Before moving into more advanced ApplicationSet features, the next practical step should be:

```text
72 — Multi-Environment ApplicationSets
```

We will create the actual:

```text
ecommerce-dev
ecommerce-qa
ecommerce-staging
ecommerce-prod
```

and you'll see **exactly how the same `order-service` moves from DEV → QA → STAGING → PROD**, including:

```text
GitOps directories
values files
ApplicationSets
AppProjects
sync policies
image promotion
production approval
rollback
```

After that, we'll return to the 21 services and onboard them systematically.
