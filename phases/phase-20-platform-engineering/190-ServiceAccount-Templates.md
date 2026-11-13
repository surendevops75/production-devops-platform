# Chapter 190 — ServiceAccount Templates

Phase 20 — Platform Engineering

Complete GitHub-ready Markdown chapter · AWS EKS · Kubernetes RBAC · IRSA · 21 microservices

File: `190-serviceaccount-templates.md`

## 190.1. Objective

In this chapter, we will build a reusable identity and access management foundation for our 21-microservice e-commerce platform running on AWS EKS.

Our objective is to ensure that every application and platform workload receives only the permissions it actually needs.

We will implement:

1. Kubernetes ServiceAccounts.
2. Kubernetes Roles and ClusterRoles.
3. RoleBindings and ClusterRoleBindings.
4. Least-privilege RBAC.
5. AWS IAM Roles for Service Accounts (IRSA).
6. OIDC provider integration.
7. IAM trust policies.
8. Secure AWS service access from pods.
9. Reusable Helm templates.
10. Environment-specific configurations.
11. GitHub Actions validation.
12. ArgoCD GitOps deployment.
13. Security testing and auditing.
14. Production troubleshooting and rollback.

At the end, developers should be able to deploy a microservice without receiving unnecessary Kubernetes administrator permissions or AWS credentials.

## 190.2. Why ServiceAccounts are important

A Kubernetes pod can communicate with the Kubernetes API server.

For example, an application may need to:

- Read ConfigMaps.
- Read Secrets.
- Discover Kubernetes Services.
- Watch selected resources.
- Access Kubernetes metadata.
- Interact with the Kubernetes API.

The pod uses a ServiceAccount identity when it makes authenticated Kubernetes API requests.

Consider this architecture:

```
                 AWS EKS CLUSTER
                       |
                       |
                Kubernetes API
                       |
                       |
               ServiceAccount
                       |
                       |
                 Order Service Pod
                       |
              +--------+--------+
              |                 |
              v                 v
        Kubernetes API       AWS Services
              |                 |
              v                 v
          Kubernetes RBAC     AWS IAM
```

There are two different authorization systems:

| System          | Controls                                            |
| --------------- | --------------------------------------------------- |
| Kubernetes RBAC | What the workload can do against the Kubernetes API |
| AWS IAM         | What the workload can do against AWS APIs           |

A Kubernetes Role does not grant permission to read an S3 bucket.

An AWS IAM policy does not grant permission to list Kubernetes Secrets.

We must configure each permission system independently.

## 190.3. ServiceAccount authentication flow

When a pod runs with a ServiceAccount, Kubernetes provides an identity for that workload.

The high-level flow is:

```
Developer deploys application
          |
          v
Deployment references ServiceAccount
          |
          v
Kubernetes schedules Pod
          |
          v
Pod receives ServiceAccount identity
          |
          v
Application requests Kubernetes API
          |
          v
API Server authenticates identity
          |
          v
RBAC authorizes requested action
          |
          v
Request allowed or denied
```

For AWS access using IRSA, the flow is different:

```
Order Service Pod
        |
        v
ServiceAccount annotation
        |
        v
Projected OIDC token
        |
        v
AWS STS AssumeRoleWithWebIdentity
        |
        v
IAM trust policy validation
        |
        v
Temporary AWS credentials
        |
        v
AWS API request
        |
        v
IAM policy authorization
```

The application receives temporary credentials rather than requiring permanent AWS access keys in its container image or environment configuration.

## 190.4. Kubernetes identity resources

We will use the following Kubernetes resources.

| Resource           | Purpose                                                     |
| ------------------ | ----------------------------------------------------------- |
| ServiceAccount     | Represents a workload identity                              |
| Role               | Defines permissions inside one namespace                    |
| ClusterRole        | Defines cluster-wide or reusable permissions                |
| RoleBinding        | Grants a Role or ClusterRole to subjects within a namespace |
| ClusterRoleBinding | Grants a ClusterRole at cluster scope                       |
| Secret             | Stores sensitive data; access must be restricted            |
| Deployment         | Associates pods with a ServiceAccount                       |

### 190.4.1. ServiceAccount

A ServiceAccount represents an identity that a pod can use.

Example:

```
apiVersion: v1
kind: ServiceAccount

metadata:
  name: order-service-sa
  namespace: ecommerce

  labels:
    app.kubernetes.io/name: order-service
    app.kubernetes.io/part-of: ecommerce

automountServiceAccountToken: false
```

Important setting:

```
automountServiceAccountToken: false
```

This prevents automatic mounting of the standard Kubernetes API credential into the pod.

Use this when the workload does not need to call the Kubernetes API.

For IRSA, the EKS web identity token projection is a separate mechanism. Do not assume that disabling automatic Kubernetes API token mounting disables IRSA. Validate the projected token and SDK behavior in the actual EKS configuration.

### 190.4.2. Role

A Role defines permissions within one namespace.

Example:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:
  name: order-service-config-reader
  namespace: ecommerce

rules:
  - apiGroups:
      - ""

    resources:
      - configmaps

    resourceNames:
      - order-service-config

    verbs:
      - get
```

This Role allows reading one named ConfigMap.

It does not grant access to all ConfigMaps.

### 190.4.3. RoleBinding

A RoleBinding attaches permissions to a subject.

Example:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:
  name: order-service-config-reader-binding
  namespace: ecommerce

subjects:
  - kind: ServiceAccount
    name: order-service-sa
    namespace: ecommerce

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: order-service-config-reader
```

The relationship is:

```
ServiceAccount
      |
      v
RoleBinding
      |
      v
Role
      |
      v
Allowed API operations
```

### 190.4.4. ClusterRole

A ClusterRole can define permissions for cluster-scoped resources or reusable permissions that are bound within namespaces.

Example:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole

metadata:
  name: application-service-discovery

rules:
  - apiGroups:
      - ""

    resources:
      - services
      - endpoints
      - pods

    verbs:
      - get
      - list
      - watch
```

This example is intentionally broader than a single-object permission. It should only be granted if the workload genuinely needs discovery access.

### 190.4.5. ClusterRoleBinding

A ClusterRoleBinding grants a ClusterRole at cluster scope.

Example:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding

metadata:
  name: application-service-discovery-binding

subjects:
  - kind: ServiceAccount
    name: discovery-service-sa
    namespace: ecommerce

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: application-service-discovery
```

Use ClusterRoleBinding carefully.

A ClusterRoleBinding does not limit its permissions to the ServiceAccount's namespace.

## 190.5. Define workload identity requirements

Before creating a ServiceAccount, identify what the workload needs.

For each microservice, ask:

1. Does it call the Kubernetes API?
2. Does it need to read ConfigMaps?
3. Does it need to read Secrets?
4. Does it need to list pods?
5. Does it need to watch Services?
6. Does it need AWS API access?
7. Which AWS services does it access?
8. Does it need access to S3?
9. Does it need to read from Secrets Manager?
10. Does it need to publish messages to AWS services?
11. Does it need to assume another IAM role?
12. Does it need access to resources in another namespace?

Do not create permissions simply because another microservice uses them.

### 190.5.1. Workload access inventory

Create:

`service-identity-inventory.yaml`

```
services:

  - name: auth-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: true
      services:
        - secretsmanager

    permissions:
      - action: read-application-secret
        resource: auth-service-secret

  - name: user-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: false

    permissions: []

  - name: customer-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: false

    permissions: []

  - name: catalog-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: false

    permissions: []

  - name: product-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: true
      services:
        - s3

    permissions:
      - action: read-product-images
        resource: product-images-bucket

  - name: inventory-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: false

    permissions: []

  - name: pricing-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: false

    permissions: []

  - name: promotion-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: false

    permissions: []

  - name: cart-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: false

    permissions: []

  - name: order-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: true
      services:
        - sqs

    permissions:
      - action: publish-order-events
        resource: order-events-queue

  - name: payment-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: true
      services:
        - secretsmanager

    permissions:
      - action: read-payment-configuration
        resource: payment-service-secret

  - name: fraud-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: false

    permissions: []

  - name: shipping-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: false

    permissions: []

  - name: delivery-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: false

    permissions: []

  - name: notification-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: true
      services:
        - ses

    permissions:
      - action: send-approved-notifications
        resource: approved-sender-identity

  - name: search-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: false

    permissions: []

  - name: recommendation-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: false

    permissions: []

  - name: review-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: false

    permissions: []

  - name: tax-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: false

    permissions: []

  - name: invoice-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: true
      services:
        - s3

    permissions:
      - action: write-invoice-files
        resource: invoice-archive-bucket

  - name: reporting-service

    namespace: ecommerce

    kubernetesApiAccess: false

    awsAccess:
      enabled: true
      services:
        - s3

    permissions:
      - action: write-report-files
        resource: reporting-archive-bucket
```

Important: The AWS service requirements above are illustrative architecture assumptions. They are not verified requirements for the 21 applications. Application owners must confirm the actual API calls, resources and actions before IAM policies are created.

## 190.6. Define the repository structure

We will maintain reusable identity templates in the platform repository.

```
company-ecommerce-platform/
|
+-- service-accounts/
|   |
|   +-- charts/
|   |   |
|   |   +-- service-identity/
|   |       |
|   |       +-- Chart.yaml
|   |       +-- values.yaml
|   |       +-- values.schema.json
|   |       |
|   |       +-- templates/
|   |           |
|   |           +-- _helpers.tpl
|   |           +-- serviceaccount.yaml
|   |           +-- role.yaml
|   |           +-- rolebinding.yaml
|   |           +-- clusterrole.yaml
|   |           +-- clusterrolebinding.yaml
|   |
|   +-- environments/
|   |   |
|   |   +-- dev/
|   |   +-- qa/
|   |   +-- staging/
|   |   +-- production/
|   |
|   +-- policies/
|   |   |
|   |   +-- auth-service-iam-policy.json
|   |   +-- product-service-iam-policy.json
|   |   +-- order-service-iam-policy.json
|   |   +-- payment-service-iam-policy.json
|   |   +-- notification-service-iam-policy.json
|   |   +-- invoice-service-iam-policy.json
|   |   +-- reporting-service-iam-policy.json
|   |
|   +-- tests/
|   |   |
|   |   +-- rbac-test-cases.yaml
|   |   +-- iam-policy-checks.md
|   |
|   +-- scripts/
|   |   |
|   |   +-- validate-chart.sh
|   |   +-- check-rbac.sh
|   |   +-- check-serviceaccounts.sh
|   |
|   +-- README.md
```

### 190.6.1. Repository ownership

| Area                            | Owner                           |
| ------------------------------- | ------------------------------- |
| Reusable Helm chart             | Platform Engineering            |
| Application access requirements | Application team                |
| Kubernetes RBAC review          | Platform Engineering            |
| AWS IAM policy review           | Cloud Platform / Security       |
| Production approval             | Authorized change approver      |
| GitOps deployment               | Platform Engineering            |
| Access audit                    | Security / Platform Engineering |

Application teams should request permissions through pull requests. They should not receive unrestricted production IAM or Kubernetes administrator access.

## 190.7. Build the reusable Helm chart

### 190.7.1. Chart metadata

`charts/service-identity/Chart.yaml`

```
apiVersion: v2

name: service-identity

description: Reusable Kubernetes workload identity and RBAC templates

type: application

version: 1.0.0

appVersion: "1.0"
```

### 190.7.2. Default values

`charts/service-identity/values.yaml`

```
nameOverride: ""

fullnameOverride: ""

namespace: ecommerce

service:
  name: order-service

  labels:
    app.kubernetes.io/part-of: ecommerce

serviceAccount:
  create: true

  name: ""

  automountServiceAccountToken: false

  annotations: {}

rbac:
  enabled: false

  role:
    create: false

    rules: []

  roleBinding:
    create: false

  clusterRole:
    create: false

    name: ""

    rules: []

  clusterRoleBinding:
    create: false

aws:
  irsa:
    enabled: false

    roleArn: ""

    stsRegionalEndpoints: true
```

The chart separates ServiceAccount creation from RBAC creation.

A workload that only needs AWS access through IRSA may not need any Kubernetes Role or RoleBinding.

### 190.7.3. Helper templates

`templates/_helpers.tpl`

```
{{/*
Return chart name.
*/}}

{{- define "service-identity.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}


{{/*
Return full release name.
*/}}

{{- define "service-identity.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name (include "service-identity.name" .) | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}


{{/*
Return ServiceAccount name.
*/}}

{{- define "service-identity.serviceAccountName" -}}
{{- if .Values.serviceAccount.name -}}
{{- .Values.serviceAccount.name -}}
{{- else -}}
{{- printf "%s-sa" .Values.service.name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}


{{/*
Return common labels.
*/}}

{{- define "service-identity.labels" -}}

app.kubernetes.io/name: {{ .Values.service.name }}

app.kubernetes.io/instance: {{ .Release.Name }}

app.kubernetes.io/managed-by: {{ .Release.Service }}

app.kubernetes.io/part-of: ecommerce

{{- with .Values.service.labels }}
{{ toYaml . }}
{{- end }}

{{- end -}}
```

### 190.7.4. ServiceAccount template

`templates/serviceaccount.yaml`

```
{{- if .Values.serviceAccount.create }}

apiVersion: v1
kind: ServiceAccount

metadata:
  name: {{ include "service-identity.serviceAccountName" . }}

  namespace: {{ .Values.namespace }}

  labels:
    {{- include "service-identity.labels" . | nindent 4 }}

  {{- $annotations := deepCopy .Values.serviceAccount.annotations }}

  {{- if .Values.aws.irsa.enabled }}

  {{- if not .Values.aws.irsa.roleArn }}
  {{- fail "aws.irsa.roleArn must be provided when IRSA is enabled" }}
  {{- end }}

  {{- $_ := set $annotations "eks.amazonaws.com/role-arn" .Values.aws.irsa.roleArn }}

  {{- end }}

  {{- with $annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}

automountServiceAccountToken: {{ .Values.serviceAccount.automountServiceAccountToken }}

{{- end }}
```

The IRSA annotation associates the Kubernetes ServiceAccount with an IAM role ARN.

The IAM role must also have a trust policy that accepts the correct cluster OIDC provider and ServiceAccount subject.

The annotation alone does not grant AWS access.

### 190.7.5. Role template

`templates/role.yaml`

```
{{- if and .Values.rbac.enabled .Values.rbac.role.create }}

apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:
  name: {{ include "service-identity.fullname" . }}-role

  namespace: {{ .Values.namespace }}

  labels:
    {{- include "service-identity.labels" . | nindent 4 }}

rules:
  {{- toYaml .Values.rbac.role.rules | nindent 2 }}

{{- end }}
```

### 190.7.6. RoleBinding template

`templates/rolebinding.yaml`

```
{{- if and .Values.rbac.enabled .Values.rbac.roleBinding.create }}

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:
  name: {{ include "service-identity.fullname" . }}-rolebinding

  namespace: {{ .Values.namespace }}

  labels:
    {{- include "service-identity.labels" . | nindent 4 }}

subjects:
  - kind: ServiceAccount

    name: {{ include "service-identity.serviceAccountName" . }}

    namespace: {{ .Values.namespace }}

roleRef:
  apiGroup: rbac.authorization.k8s.io

  kind: Role

  name: {{ include "service-identity.fullname" . }}-role

{{- end }}
```

### 190.7.7. ClusterRole template

`templates/clusterrole.yaml`

```
{{- if and .Values.rbac.enabled .Values.rbac.clusterRole.create }}

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole

metadata:
  name: {{ default (printf "%s-clusterrole" (include "service-identity.fullname" .)) .Values.rbac.clusterRole.name }}

  labels:
    {{- include "service-identity.labels" . | nindent 4 }}

rules:
  {{- toYaml .Values.rbac.clusterRole.rules | nindent 2 }}

{{- end }}
```

### 190.7.8. ClusterRoleBinding template

`templates/clusterrolebinding.yaml`

```
{{- if and .Values.rbac.enabled .Values.rbac.clusterRoleBinding.create }}

{{- if not .Values.rbac.clusterRole.create }}
{{- fail "clusterRole.create must be true when clusterRoleBinding.create is true in this chart" }}
{{- end }}

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding

metadata:
  name: {{ include "service-identity.fullname" . }}-clusterrolebinding

  labels:
    {{- include "service-identity.labels" . | nindent 4 }}

subjects:
  - kind: ServiceAccount

    name: {{ include "service-identity.serviceAccountName" . }}

    namespace: {{ .Values.namespace }}

roleRef:
  apiGroup: rbac.authorization.k8s.io

  kind: ClusterRole

  name: {{ default (printf "%s-clusterrole" (include "service-identity.fullname" .)) .Values.rbac.clusterRole.name }}

{{- end }}
```

Production note: The chart above demonstrates reusable resource generation. Before adopting it, validate every rendered combination. In particular, use a dedicated, reviewed mechanism for cluster-scoped permissions. A normal application release should not be allowed to create arbitrary ClusterRoles or ClusterRoleBindings.

## 190.8. Configure ServiceAccount for a microservice

Example:

`environments/dev/order-service-values.yaml`

```
namespace: ecommerce

service:
  name: order-service

  labels:
    environment: dev

serviceAccount:
  create: true

  name: order-service-sa

  automountServiceAccountToken: false

  annotations: {}

rbac:
  enabled: false

aws:
  irsa:
    enabled: true

    roleArn: arn:aws:iam::111122223333:role/ecommerce-dev-order-service-role
```

This configuration creates a ServiceAccount associated with the specified IAM role.

The account ID and role name are illustrative. Replace them with the actual values from the AWS environment.

### 190.8.1. Render the chart

```
helm template order-service-identity \
  ./charts/service-identity \
  --namespace ecommerce \
  -f ./environments/dev/order-service-values.yaml
```

Expected ServiceAccount structure:

```
apiVersion: v1
kind: ServiceAccount

metadata:
  name: order-service-sa
  namespace: ecommerce

  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::111122223333:role/ecommerce-dev-order-service-role

automountServiceAccountToken: false
```

Inspect the rendered file before deployment.

## 190.9. Attach the ServiceAccount to a Deployment

Creating a ServiceAccount is not enough.

The application Deployment must reference it.

Example:

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: order-service

  namespace: ecommerce

spec:
  replicas: 2

  selector:
    matchLabels:
      app.kubernetes.io/name: order-service

  template:
    metadata:
      labels:
        app.kubernetes.io/name: order-service
        app.kubernetes.io/part-of: ecommerce

    spec:
      serviceAccountName: order-service-sa

      automountServiceAccountToken: false

      containers:
        - name: order-service

          image: 111122223333.dkr.ecr.ap-south-1.amazonaws.com/order-service:1.0.0

          ports:
            - name: http
              containerPort: 8080

          env:
            - name: AWS_REGION
              value: ap-south-1
```

The pod-level `automountServiceAccountToken` setting takes precedence over the ServiceAccount setting.

For IRSA, validate that the EKS integration projects the web identity token required by the AWS SDK. Do not remove or override the SDK's expected web identity configuration when hardening the pod.

### 190.9.1. Inspect the running pod

```
kubectl get pods -n ecommerce \
  -l app.kubernetes.io/name=order-service
```

Inspect its ServiceAccount:

```
kubectl get pod <order-pod-name> \
  -n ecommerce \
  -o jsonpath='{.spec.serviceAccountName}'
```

Expected:

```
order-service-sa
```

Inspect the ServiceAccount:

```
kubectl get serviceaccount order-service-sa \
  -n ecommerce \
  -o yaml
```

## 190.10. Kubernetes RBAC implementation

Most application containers do not need Kubernetes API access.

If an application only serves HTTP requests and connects to other services, databases or AWS APIs, it normally does not need a Role.

We will start with no Kubernetes RBAC permissions.

When an application genuinely needs Kubernetes API access, define the smallest possible permissions.

### 190.10.1. Example: read one ConfigMap

Suppose Order Service needs to read:

```
order-service-config
```

Create:

`rbac/order-service-config-reader.yaml`

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:
  name: order-service-config-reader
  namespace: ecommerce

rules:
  - apiGroups:
      - ""

    resources:
      - configmaps

    resourceNames:
      - order-service-config

    verbs:
      - get
```

Bind it:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:
  name: order-service-config-reader-binding
  namespace: ecommerce

subjects:
  - kind: ServiceAccount

    name: order-service-sa

    namespace: ecommerce

roleRef:
  apiGroup: rbac.authorization.k8s.io

  kind: Role

  name: order-service-config-reader
```

Apply in DEV:

```
kubectl apply -f rbac/order-service-config-reader.yaml
```

For production, submit the manifests through the reviewed GitOps process rather than making an unmanaged manual change.

### 190.10.2. Verify access

Check whether the ServiceAccount can read the ConfigMap:

```
kubectl auth can-i get configmap/order-service-config \
  -n ecommerce \
  --as=system:serviceaccount:ecommerce:order-service-sa
```

Expected:

```
yes
```

Check another ConfigMap:

```
kubectl auth can-i get configmap/another-service-config \
  -n ecommerce \
  --as=system:serviceaccount:ecommerce:order-service-sa
```

Expected:

```
no
```

The test identity must be authorized to perform impersonation. If the operator running `kubectl auth can-i --as` does not have impersonation permission, the test itself may be denied.

### 190.10.3. Check permissions in a namespace

```
kubectl auth can-i --list \
  -n ecommerce \
  --as=system:serviceaccount:ecommerce:order-service-sa
```

Review the effective permissions, including permissions granted through other bindings.

### 190.10.4. Verify that application cannot list Secrets

```
kubectl auth can-i list secrets \
  -n ecommerce \
  --as=system:serviceaccount:ecommerce:order-service-sa
```

Expected:

```
no
```

Do not grant `list` or `watch` on Secrets unless the workload has a documented requirement.

### 190.10.5. Why `resourceNames` matters

Compare these two rules.

Broad permission:

```
rules:
  - apiGroups:
      - ""

    resources:
      - configmaps

    verbs:
      - get
      - list
      - watch
```

Narrow permission:

```
rules:
  - apiGroups:
      - ""

    resources:
      - configmaps

    resourceNames:
      - order-service-config

    verbs:
      - get
```

The second rule limits access to one named ConfigMap for `get`.

There is an important Kubernetes RBAC detail: a `list` or `watch` request with `resourceNames` generally needs a matching `metadata.name` field selector. Do not assume `resourceNames` turns an ordinary unfiltered list into a safe, narrowly scoped operation.

## 190.11. When to use ClusterRole

Use a ClusterRole only when the workload genuinely needs access to cluster-scoped resources or reusable permissions across namespaces.

Examples of cluster-scoped resources include:

- Nodes.
- PersistentVolumes.
- Namespaces.
- CustomResourceDefinitions.

Some resources are namespaced, such as Pods and ConfigMaps, but a ClusterRole can define reusable rules for them.

### 190.11.1. Example: read nodes

This example is for a trusted infrastructure component that genuinely needs node information, not for a normal business microservice.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole

metadata:
  name: node-metadata-reader

rules:
  - apiGroups:
      - ""

    resources:
      - nodes

    verbs:
      - get
      - list
      - watch
```

Bind it to an explicitly approved ServiceAccount:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding

metadata:
  name: node-metadata-reader-binding

subjects:
  - kind: ServiceAccount

    name: infrastructure-observer-sa

    namespace: monitoring

roleRef:
  apiGroup: rbac.authorization.k8s.io

  kind: ClusterRole

  name: node-metadata-reader
```

Do not grant this to all 21 application ServiceAccounts.

## 190.12. EKS IAM Roles for Service Accounts (IRSA)

IRSA allows Kubernetes workloads to obtain temporary AWS credentials through an IAM role.

It avoids putting permanent IAM access keys inside application containers.

### 190.12.1. Why IRSA is needed

Consider an application that stores product images in S3.

Without workload identity, a developer might put credentials into a Kubernetes Secret:

```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

This creates risks:

- Credentials may be copied into logs.
- Credentials may be committed accidentally.
- Credentials may be reused by another workload.
- Rotation becomes difficult.
- A compromised pod may expose long-lived keys.

With IRSA:

```
Product Service Pod
        |
        v
Product Service ServiceAccount
        |
        v
OIDC Web Identity Token
        |
        v
AWS STS
        |
        v
IAM Role
        |
        v
S3 Permissions
```

The application receives temporary credentials through the supported AWS SDK credential provider chain.

### 190.12.2. Verify EKS cluster identity

Set variables:

```
export AWS_REGION=ap-south-1

export CLUSTER_NAME=ecommerce-dev
```

Inspect the cluster:

```
aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --query 'cluster.identity.oidc.issuer' \
  --output text
```

Example output:

```
https://oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E
```

The OIDC issuer URL identifies the cluster's identity provider.

### 190.12.3. Check the OIDC provider

```
aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --query 'cluster.identity.oidc.issuer' \
  --output text
```

Check whether the corresponding IAM OIDC provider exists:

```
aws iam list-open-id-connect-providers
```

Inspect the provider ARN and configuration using:

```
aws iam get-open-id-connect-provider \
  --open-id-connect-provider-arn <oidc-provider-arn>
```

If the provider is missing, create it using the approved cluster provisioning workflow.

### 190.12.4. Associate the OIDC provider

For an EKS cluster where the provider has not yet been associated, the AWS CLI workflow can use:

```
eksctl utils associate-iam-oidc-provider \
  --cluster ecommerce-dev \
  --region ap-south-1 \
  --approve
```

This command requires `eksctl` and the necessary AWS permissions.

For a platform managed through Terraform, manage the OIDC provider and related IAM resources in the Terraform infrastructure repository instead of creating unmanaged resources manually.

### 190.12.5. Extract the OIDC issuer ID

```
OIDC_ISSUER=$(aws eks describe-cluster \
  --name "$CLUSTER_NAME" \
  --region "$AWS_REGION" \
  --query 'cluster.identity.oidc.issuer' \
  --output text)

OIDC_ID="${OIDC_ISSUER##*/}"

echo "$OIDC_ID"
```

Example:

```
EXAMPLED539D4633E53DE1B716D3041E
```

The IAM OIDC provider ARN normally follows this pattern:

```
arn:aws:iam::111122223333:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLE
```

Use the actual account ID and issuer ID.

### 190.12.6. Understand the IAM trust relationship

The IAM role must trust the EKS OIDC provider.

The trust policy must restrict both:

- `aud`: the intended audience, normally `sts.amazonaws.com`.
- `sub`: the exact Kubernetes ServiceAccount identity.

For example:

```
system:serviceaccount:ecommerce:order-service-sa
```

This prevents another namespace or ServiceAccount from assuming the same role unless explicitly included in the trust policy.

## 190.13. Create the IAM trust policy

Create:

`policies/order-service-trust-policy.json`

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:aud": "sts.amazonaws.com",
          "oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:sub": "system:serviceaccount:ecommerce:order-service-sa"
        }
      }
    }
  ]
}
```

Replace:

- AWS account ID.
- OIDC provider ID.
- Region.
- Namespace.
- ServiceAccount name.

Do not use a wildcard subject for normal application roles.

### 190.13.1. Understand the trust policy conditions

| Condition                   | Meaning                                              |
| --------------------------- | ---------------------------------------------------- |
| `aud`                       | Token audience must be STS                           |
| `sub`                       | Token subject must match the intended ServiceAccount |
| `Federated`                 | Trusts the cluster's IAM OIDC provider               |
| `AssumeRoleWithWebIdentity` | Allows the web identity token to assume the role     |

The trust policy answers:

Which workload is allowed to assume this IAM role?

It does not answer:

What can that workload do after assuming the role?

The second question is answered by the role's permissions policy.

## 190.14. Create an IAM role for Order Service

Create the IAM role:

```
aws iam create-role \
  --role-name ecommerce-dev-order-service-role \
  --assume-role-policy-document file://policies/order-service-trust-policy.json \
  --description "IRSA role for Order Service in ecommerce DEV"
```

Check the role:

```
aws iam get-role \
  --role-name ecommerce-dev-order-service-role
```

The trust policy must be reviewed before attaching permissions.

## 190.15. Create an AWS permissions policy

Suppose Order Service needs to publish messages to one SQS queue.

This is an illustrative requirement.

Create:

`policies/order-service-iam-policy.json`

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublishOrderEvents",
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:GetQueueAttributes",
        "sqs:GetQueueUrl"
      ],
      "Resource": "arn:aws:sqs:ap-south-1:111122223333:ecommerce-order-events"
    }
  ]
}
```

This policy does not allow the application to:

- Delete the queue.
- Create arbitrary queues.
- Read messages.
- Receive messages.
- Modify unrelated SQS resources.

The exact permissions depend on whether the application publishes, consumes or performs other queue operations.

### 190.15.1. Create the IAM policy

```
aws iam create-policy \
  --policy-name ecommerce-dev-order-service-policy \
  --policy-document file://policies/order-service-iam-policy.json \
  --description "Least privilege permissions for Order Service"
```

Capture the returned policy ARN.

Example:

```
arn:aws:iam::111122223333:policy/ecommerce-dev-order-service-policy
```

### 190.15.2. Attach the policy

```
aws iam attach-role-policy \
  --role-name ecommerce-dev-order-service-role \
  --policy-arn arn:aws:iam::111122223333:policy/ecommerce-dev-order-service-policy
```

Inspect attached policies:

```
aws iam list-attached-role-policies \
  --role-name ecommerce-dev-order-service-role
```

### 190.15.3. Inspect the effective permissions

Review:

- Attached managed policies.
- Inline policies.
- Permissions boundaries.
- Organization-level service control policies.
- Resource policies on the destination AWS service.
- Any applicable VPC endpoint policies.

An attached policy alone does not guarantee that an AWS request will succeed.

## 190.16. Create IRSA role for Product Service

Suppose Product Service needs to read product images from a particular S3 bucket.

Create:

`policies/product-service-iam-policy.json`

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListProductImageBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::ecommerce-product-images"
    },
    {
      "Sid": "ReadProductImages",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::ecommerce-product-images/*"
    }
  ]
}
```

This permits listing the bucket and reading objects.

If the application only needs to read a known object key and does not need to list the bucket, remove `s3:ListBucket`.

If the application needs uploads, explicitly review the required write permissions.

Do not give Product Service `s3:*`.

### 190.16.1. Product Service trust policy

The trust policy should identify:

```
system:serviceaccount:ecommerce:product-service-sa
```

It should not reuse Order Service's subject.

### 190.16.2. Product Service Helm values

```
namespace: ecommerce

service:
  name: product-service

serviceAccount:
  create: true

  name: product-service-sa

  automountServiceAccountToken: false

aws:
  irsa:
    enabled: true

    roleArn: arn:aws:iam::111122223333:role/ecommerce-dev-product-service-role
```

The Product Service receives its own role and permissions.

## 190.17. Configure the AWS SDK inside applications

IRSA works through the AWS SDK credential provider chain.

The application must use an AWS SDK version that supports web identity credentials.

For Java applications, use a supported AWS SDK for Java version and its default credentials provider chain.

Illustrative Java example:

```
import software.amazon.awssdk.auth.credentials.DefaultCredentialsProvider;
import software.amazon.awssdk.services.s3.S3Client;

public class S3Configuration {

    public S3Client createS3Client() {

        return S3Client.builder()
                .region(software.amazon.awssdk.regions.Region.AP_SOUTH_1)
                .credentialsProvider(
                    DefaultCredentialsProvider.create()
                )
                .build();
    }
}
```

The default credentials provider chain can use the web identity token configuration injected for IRSA.

Do not hardcode access keys:

```
// Do not use hardcoded production credentials.

AwsBasicCredentials credentials =
    AwsBasicCredentials.create(
        "ACCESS_KEY",
        "SECRET_KEY"
    );
```

Do not store AWS credentials in:

- Dockerfiles.
- Source code.
- GitHub repository files.
- Helm values.
- ConfigMaps.
- Container images.

Use the workload identity mechanism.

## 190.18. Deploy the IRSA configuration

### 190.18.1. Deploy the ServiceAccount

In DEV, you can test the rendered manifest:

```
helm template order-service-identity \
  ./charts/service-identity \
  --namespace ecommerce \
  -f ./environments/dev/order-service-values.yaml
```

After review, deploy through the normal platform deployment process.

For a direct DEV test:

```
helm upgrade --install order-service-identity \
  ./charts/service-identity \
  --namespace ecommerce \
  --create-namespace \
  -f ./environments/dev/order-service-values.yaml
```

Do not use this manual installation as a substitute for the production GitOps workflow.

### 190.18.2. Verify the ServiceAccount annotation

```
kubectl describe serviceaccount order-service-sa \
  -n ecommerce
```

Verify that the role ARN is present.

### 190.18.3. Restart the application if required

If the Deployment was not updated to reference the ServiceAccount, update it through its source manifest or Helm values.

Then verify the pod:

```
kubectl get pods -n ecommerce \
  -l app.kubernetes.io/name=order-service
```

### 190.18.4. Verify the projected token

For a pod using IRSA, inspect the pod specification:

```
kubectl get pod <order-pod-name> \
  -n ecommerce \
  -o yaml
```

Check for the projected web identity token volume and the relevant environment configuration injected by the EKS integration.

Do not print the token itself.

A projected token is a credential. Treat it as sensitive.

## 190.19. Test IRSA from the running pod

For a controlled test image containing AWS CLI:

```
kubectl exec -it <order-pod-name> \
  -n ecommerce \
  -- aws sts get-caller-identity
```

Expected output should identify the assumed IAM role session associated with the Order Service role.

The ARN typically resembles:

```
arn:aws:sts::111122223333:assumed-role/ecommerce-dev-order-service-role/<session-name>
```

The exact session name may vary.

### 190.19.1. Test permitted SQS access

If Order Service is configured to publish to the approved queue, execute an application-level integration test.

For example:

```
aws sqs get-queue-attributes \
  --queue-url <approved-queue-url> \
  --attribute-names QueueArn
```

Only run this command if the application's IAM policy permits it.

Then test message publishing through the actual application code.

### 190.19.2. Test denied access

Try accessing an unrelated S3 bucket from a controlled test pod using the same role.

Expected result:

```
AccessDenied
```

The test should use a real resource that the role is not permitted to access.

A successful `get-caller-identity` only proves that the role was assumed. It does not prove that the permissions are least privilege.

### 190.19.3. Do not leak credentials

Never run:

```
env
```

and publish the complete output of a production workload.

Environment variables may contain secrets and token-related paths.

For troubleshooting, inspect only the specific non-secret variables and configuration fields required.

## 190.20. EKS Pod Identity alternative

IRSA is one supported EKS workload identity method.

EKS Pod Identity is another method for associating IAM roles with Kubernetes ServiceAccounts.

Conceptually:

```
Kubernetes ServiceAccount
          |
          v
EKS Pod Identity Association
          |
          v
EKS Pod Identity Agent
          |
          v
AWS Credentials
          |
          v
IAM Role Permissions
```

### 190.20.1. IRSA versus EKS Pod Identity

| Area                      | IRSA                              | EKS Pod Identity                                   |
| ------------------------- | --------------------------------- | -------------------------------------------------- |
| Identity mechanism        | OIDC web identity                 | EKS Pod Identity association                       |
| IAM role trust            | OIDC provider                     | EKS Pod Identity service principal                 |
| ServiceAccount annotation | Commonly used for role ARN        | Association is managed separately                  |
| Credential delivery       | Web identity token and STS        | Pod Identity Agent and EKS integration             |
| Cluster configuration     | OIDC provider required            | Pod Identity Agent and supported EKS configuration |
| Use case                  | Established EKS workload identity | Alternative managed EKS workload identity          |

Do not configure both mechanisms for the same workload without a deliberate design.

For this chapter, we use IRSA as the main implementation path. Pod Identity is an alternative that the platform team can evaluate separately.

## 190.21. IAM policy examples for other services

The following examples illustrate how to scope access to individual workloads. They must be reviewed against actual application behavior.

### 190.21.1. Auth Service — read one Secrets Manager secret

`policies/auth-service-iam-policy.json`

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadAuthConfiguration",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:ap-south-1:111122223333:secret:ecommerce/dev/auth-service-*"
    }
  ]
}
```

If the secret uses a customer-managed KMS key, the role may also require appropriately scoped `kms:Decrypt` permission and the KMS key policy must permit the operation.

Do not grant `secretsmanager:*`.

### 190.21.2. Payment Service — read payment configuration

`policies/payment-service-iam-policy.json`

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadPaymentConfiguration",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:ap-south-1:111122223333:secret:ecommerce/dev/payment-service-*"
    }
  ]
}
```

Payment Service should not automatically receive access to every secret in the AWS account.

### 190.21.3. Notification Service — send email

If the application uses Amazon SES, the required permissions depend on the API and identity configuration.

Illustrative policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SendApprovedEmail",
      "Effect": "Allow",
      "Action": [
        "ses:SendEmail",
        "ses:SendRawEmail"
      ],
      "Resource": "arn:aws:ses:ap-south-1:111122223333:identity/example.com"
    }
  ]
}
```

The actual SES resource ARN and required actions must match the configured SES identity and API usage.

### 190.21.4. Invoice Service — write invoice documents

`policies/invoice-service-iam-policy.json`

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "WriteInvoiceDocuments",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::ecommerce-invoice-archive/*"
    }
  ]
}
```

If the application needs multipart uploads, object tagging, encryption headers, or reads, review the additional actions individually.

### 190.21.5. Reporting Service — write reports

`policies/reporting-service-iam-policy.json`

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "WriteReports",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::ecommerce-reporting-archive/*"
    }
  ]
}
```

Do not reuse Invoice Service's IAM role merely because both services use S3.

Separate roles provide better isolation and auditability.

## 190.22. Separate IAM roles by environment

Avoid using one IAM role across DEV, QA, STAGING and PRODUCTION.

Recommended:

```
DEV
|
+-- ecommerce-dev-order-service-role
+-- ecommerce-dev-product-service-role
+-- ecommerce-dev-payment-service-role

QA
|
+-- ecommerce-qa-order-service-role
+-- ecommerce-qa-product-service-role
+-- ecommerce-qa-payment-service-role

STAGING
|
+-- ecommerce-staging-order-service-role
+-- ecommerce-staging-product-service-role
+-- ecommerce-staging-payment-service-role

PRODUCTION
|
+-- ecommerce-production-order-service-role
+-- ecommerce-production-product-service-role
+-- ecommerce-production-payment-service-role
```

Each environment should have:

- Its own cluster identity.
- Its own OIDC trust relationship where using IRSA.
- Its own IAM roles.
- Its own resource ARNs.
- Its own approval process.

Production workloads should not assume development roles.

## 190.23. Manage IRSA using Terraform

For a production platform, Terraform should manage the AWS IAM resources.

A typical repository structure:

```
company-ecommerce-infrastructure/
|
+-- terraform/
    |
    +-- modules/
    |   |
    |   +-- eks/
    |   +-- iam-irsa/
    |   +-- iam-policy/
    |
    +-- environments/
        |
        +-- dev/
        |   +-- eks.tf
        |   +-- irsa.tf
        |   +-- variables.tf
        |
        +-- qa/
        +-- staging/
        +-- production/
```

### 190.23.1. IRSA Terraform module

`modules/iam-irsa/variables.tf`

```
variable "role_name" {
  description = "IAM role name"
  type        = string
}

variable "oidc_provider_arn" {
  description = "EKS IAM OIDC provider ARN"
  type        = string
}

variable "oidc_provider_url" {
  description = "EKS OIDC issuer URL without https://"
  type        = string
}

variable "namespace" {
  description = "Kubernetes namespace"
  type        = string
}

variable "service_account_name" {
  description = "Kubernetes ServiceAccount name"
  type        = string
}

variable "policy_arns" {
  description = "IAM policy ARNs to attach"
  type        = list(string)
  default     = []
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default     = {}
}
```

### 190.23.2. Trust policy

`modules/iam-irsa/main.tf`

```
data "aws_iam_policy_document" "trust" {
  statement {
    effect = "Allow"

    actions = [
      "sts:AssumeRoleWithWebIdentity"
    ]

    principals {
      type = "Federated"

      identifiers = [
        var.oidc_provider_arn
      ]
    }

    condition {
      test = "StringEquals"

      variable = "${var.oidc_provider_url}:aud"

      values = [
        "sts.amazonaws.com"
      ]
    }

    condition {
      test = "StringEquals"

      variable = "${var.oidc_provider_url}:sub"

      values = [
        "system:serviceaccount:${var.namespace}:${var.service_account_name}"
      ]
    }
  }
}

resource "aws_iam_role" "workload" {
  name               = var.role_name
  assume_role_policy = data.aws_iam_policy_document.trust.json

  tags = var.tags
}

resource "aws_iam_role_policy_attachment" "managed" {
  for_each = toset(var.policy_arns)

  role       = aws_iam_role.workload.name
  policy_arn = each.value
}
```

This module creates the role and attaches managed policies. The policies themselves should be separately reviewed and managed.

### 190.23.3. Module outputs

`modules/iam-irsa/outputs.tf`

```
output "role_arn" {
  description = "IAM role ARN for the Kubernetes ServiceAccount"
  value       = aws_iam_role.workload.arn
}

output "role_name" {
  description = "IAM role name"
  value       = aws_iam_role.workload.name
}
```

### 190.23.4. Call the module

`environments/dev/irsa.tf`

```
module "order_service_irsa" {
  source = "../../modules/iam-irsa"

  role_name = "ecommerce-dev-order-service-role"

  oidc_provider_arn = aws_iam_openid_connect_provider.eks.arn

  oidc_provider_url = replace(
    aws_iam_openid_connect_provider.eks.url,
    "https://",
    ""
  )

  namespace = "ecommerce"

  service_account_name = "order-service-sa"

  policy_arns = [
    aws_iam_policy.order_service_sqs.arn
  ]

  tags = {
    Environment = "dev"
    Application = "order-service"
    ManagedBy   = "terraform"
  }
}
```

The example assumes that the OIDC provider and `aws_iam_policy.order_service_sqs` resource are defined in the same Terraform configuration or are passed through the appropriate module interface.

### 190.23.5. Terraform execution

```
terraform init
```

Format:

```
terraform fmt -recursive
```

Validate:

```
terraform validate
```

Review:

```
terraform plan
```

Apply only after approval:

```
terraform apply
```

For production, use the established remote state backend, state locking strategy, environment isolation and protected deployment workflow.

Do not store Terraform state in an unsecured local directory for shared production infrastructure.

## 190.24. Integrate ServiceAccount with the application Helm chart

The identity chart creates the ServiceAccount, but the application chart must use the same name.

For the Order Service application chart:

`values.yaml`

```
serviceAccount:
  create: false

  name: order-service-sa

  automountServiceAccountToken: false
```

In the application chart's Deployment template:

```
spec:
  template:
    spec:
      serviceAccountName: {{ .Values.serviceAccount.name }}

      automountServiceAccountToken: false

      containers:
        - name: order-service
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

This avoids creating duplicate ServiceAccounts in two different charts.

The platform identity chart owns the ServiceAccount resource. The application chart references it.

### 190.24.1. Why this separation matters

Without ownership separation:

```
Identity chart creates order-service-sa
Application chart creates order-service-sa
GitOps manages both
```

This can cause ownership conflicts and unexpected resource changes.

With clear ownership:

```
Platform Identity Chart
        |
        +-- Creates ServiceAccount
        |
        +-- Creates approved RBAC

Application Chart
        |
        +-- References ServiceAccount
        |
        +-- Deploys application pods
```

Only one chart should own the ServiceAccount resource.

## 190.25. Environment-specific configuration

Create separate files.

```
service-accounts/environments/
|
+-- dev/
|   +-- order-service-values.yaml
|   +-- product-service-values.yaml
|
+-- qa/
|   +-- order-service-values.yaml
|   +-- product-service-values.yaml
|
+-- staging/
|   +-- order-service-values.yaml
|   +-- product-service-values.yaml
|
+-- production/
    +-- order-service-values.yaml
    +-- product-service-values.yaml
```

Example production Order Service configuration:

`environments/production/order-service-values.yaml`

```
namespace: ecommerce

service:
  name: order-service

  labels:
    environment: production

serviceAccount:
  create: true

  name: order-service-sa

  automountServiceAccountToken: false

  annotations: {}

rbac:
  enabled: false

aws:
  irsa:
    enabled: true

    roleArn: arn:aws:iam::111122223333:role/ecommerce-production-order-service-role
```

Production uses a production IAM role, not a DEV role.

The role's trust policy must reference the production cluster's OIDC provider and the production workload's ServiceAccount identity.

## 190.26. GitHub Actions validation pipeline

We need to validate:

- Helm chart syntax.
- YAML structure.
- Kubernetes resource schemas.
- ServiceAccount names.
- Role and RoleBinding references.
- IAM trust policy JSON.
- IAM policy JSON.
- Unexpected wildcard permissions.
- Rendered manifests.

Create:

`.github/workflows/service-identity-validation.yaml`

```
name: Service Identity Validation

on:
  pull_request:
    paths:
      - "service-accounts/**"
      - ".github/workflows/service-identity-validation.yaml"

  push:
    branches:
      - main
    paths:
      - "service-accounts/**"

permissions:
  contents: read

jobs:
  validate:
    name: Validate Service Identity

    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install Helm
        uses: azure/setup-helm@v4
        with:
          version: v3.16.4

      - name: Install jq
        run: |
          sudo apt-get update
          sudo apt-get install -y jq

      - name: Helm lint
        run: |
          helm lint \
            service-accounts/charts/service-identity

      - name: Render Order Service identity
        run: |
          helm template order-service-identity \
            service-accounts/charts/service-identity \
            --namespace ecommerce \
            -f service-accounts/environments/dev/order-service-values.yaml \
            > rendered-order-service.yaml

      - name: Validate trust policy JSON
        run: |
          for file in service-accounts/policies/*trust-policy.json; do
            jq empty "$file"
          done

      - name: Validate IAM policy JSON
        run: |
          for file in service-accounts/policies/*iam-policy.json; do
            jq empty "$file"
          done

      - name: Verify rendered ServiceAccount
        run: |
          grep -q "kind: ServiceAccount" rendered-order-service.yaml

      - name: Upload validation artifact
        uses: actions/upload-artifact@v4
        with:
          name: service-identity-manifests
          path: rendered-order-service.yaml
```

For production:

- Pin actions to reviewed commit SHAs.
- Use approved tool versions.
- Add schema validation.
- Add policy-as-code checks.
- Review IAM policy changes separately from application code.
- Protect production branches.
- Require the appropriate reviewers.
- Avoid exposing AWS production credentials to untrusted pull-request jobs.

### 190.26.1. Add a wildcard permission check

A simple first-pass check:

```
#!/usr/bin/env bash

set -euo pipefail

for file in service-accounts/policies/*iam-policy.json; do

    echo "Checking $file"

    if jq -e '
      .Statement[]
      | select(
          .Effect == "Allow"
          and (
            .Action == "*"
            or (
              .Action | type == "array"
              and any(.[]; . == "*")
            )
          )
        )
    ' "$file" >/dev/null; then

        echo "ERROR: Wildcard Action detected in $file"
        exit 1
    fi

done

echo "IAM wildcard check passed"
```

This catches some broad permissions but is not a complete IAM security analyzer. It does not detect every dangerous combination of actions, resources, conditions or policy interactions.

Use an established IAM policy analysis tool and human review for production.

## 190.27. GitOps deployment

The GitOps repository owns the deployment configuration.

Example:

```
company-ecommerce-gitops/
|
+-- applications/
|   |
|   +-- order-service-identity-dev.yaml
|   +-- order-service-identity-qa.yaml
|   +-- order-service-identity-staging.yaml
|   +-- order-service-identity-production.yaml
|
+-- service-identities/
    |
    +-- dev/
    |   +-- order-service-values.yaml
    |
    +-- qa/
    |   +-- order-service-values.yaml
    |
    +-- staging/
    |   +-- order-service-values.yaml
    |
    +-- production/
        +-- order-service-values.yaml
```

### 190.27.1. ArgoCD Application example

```
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: order-service-identity-dev
  namespace: argocd

spec:
  project: ecommerce

  source:
    repoURL: https://github.com/company-ecommerce/company-ecommerce-platform.git

    targetRevision: main

    path: service-accounts/charts/service-identity

    helm:
      valueFiles:
        - values.yaml

  destination:
    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

This is a basic illustrative Application. The environment values must be made available to ArgoCD through the configured source path or a supported multi-source arrangement.

For production:

- Use a production-specific Application.
- Restrict who can change production role ARNs.
- Require reviewed changes.
- Keep AWS IAM creation under Terraform ownership.
- Keep Kubernetes ServiceAccount creation under the platform chart.
- Ensure the application chart references the correct ServiceAccount.

Do not use ArgoCD to create IAM roles. ArgoCD manages Kubernetes resources; Terraform or another approved AWS infrastructure workflow manages IAM resources.

## 190.28. ServiceAccount security standards

Every application ServiceAccount should follow these standards.

### 190.28.1. Unique identity per workload

Preferred:

```
order-service-sa
payment-service-sa
product-service-sa
invoice-service-sa
```

Avoid:

```
default
application-sa
common-service-account
admin-sa
```

A shared ServiceAccount can cause unrelated applications to share permissions and AWS identities.

### 190.28.2. Disable unnecessary token mounting

For applications that do not call the Kubernetes API:

```
automountServiceAccountToken: false
```

For workloads that require Kubernetes API access, enable only the required token projection and RBAC permissions.

### 190.28.3. Avoid the default ServiceAccount

Do not deploy business applications using the namespace's default ServiceAccount without reviewing its permissions.

Set explicitly:

```
spec:
  template:
    spec:
      serviceAccountName: order-service-sa
```

### 190.28.4. No administrator permissions

Never grant normal application workloads:

```
resources:
  - "*"

verbs:
  - "*"
```

Also avoid binding application ServiceAccounts to:

```
cluster-admin
```

A production application should not need to administer the Kubernetes cluster.

### 190.28.5. Avoid broad Secret access

Avoid this for a normal microservice:

```
rules:
  - apiGroups:
      - ""

    resources:
      - secrets

    verbs:
      - get
      - list
      - watch
```

If the application needs a secret, consider whether it should consume it through a controlled secret-delivery mechanism rather than querying the Kubernetes API.

A mounted Secret volume does not require the application to have Kubernetes API permissions to list Secrets.

### 190.28.6. Protect ServiceAccount creation

Permission to create or modify pods, Deployments, ServiceAccounts and RoleBindings can be security-sensitive.

For example, a user who can create a pod in a namespace may be able to choose a ServiceAccount in that namespace. If that ServiceAccount has powerful permissions, the user may be able to use those permissions indirectly.

Restrict:

- Pod creation.
- Deployment creation.
- ServiceAccount modification.
- RoleBinding creation.
- ClusterRoleBinding creation.
- Impersonation.
- Token creation.

Use admission controls and repository reviews to enforce approved workload identities.

## 190.29. Kubernetes RBAC auditing

The platform team must periodically review who can perform sensitive operations.

### 190.29.1. List ServiceAccounts

```
kubectl get serviceaccounts -A
```

### 190.29.2. List Roles

```
kubectl get roles -A
```

### 190.29.3. List RoleBindings

```
kubectl get rolebindings -A
```

### 190.29.4. List ClusterRoles

```
kubectl get clusterroles
```

### 190.29.5. List ClusterRoleBindings

```
kubectl get clusterrolebindings
```

### 190.29.6. Find bindings to cluster-admin

```
kubectl get clusterrolebindings -o json |
jq -r '
  .items[]
  | select(.roleRef.name == "cluster-admin")
  | {
      binding: .metadata.name,
      subjects: .subjects
    }
'
```

Review every result. Some cluster-admin bindings may be necessary for cluster administration, but application ServiceAccounts should not receive them.

### 190.29.7. Find ServiceAccounts with AWS role annotations

```
kubectl get serviceaccounts -A -o json |
jq -r '
  .items[]
  | select(
      .metadata.annotations["eks.amazonaws.com/role-arn"] != null
    )
  | [
      .metadata.namespace,
      .metadata.name,
      .metadata.annotations["eks.amazonaws.com/role-arn"]
    ]
  | @tsv
'
```

Use this inventory to compare the actual cluster configuration against the approved identity register.

## 190.30. Validate effective RBAC permissions

A Role or ClusterRole is only one part of the effective permission set.

A ServiceAccount may receive permissions through multiple RoleBindings and ClusterRoleBindings.

Use:

```
kubectl auth can-i --list \
  -n ecommerce \
  --as=system:serviceaccount:ecommerce:order-service-sa
```

Test specific operations:

```
kubectl auth can-i get configmaps \
  -n ecommerce \
  --as=system:serviceaccount:ecommerce:order-service-sa
```

```
kubectl auth can-i list secrets \
  -n ecommerce \
  --as=system:serviceaccount:ecommerce:order-service-sa
```

```
kubectl auth can-i create deployments \
  -n ecommerce \
  --as=system:serviceaccount:ecommerce:order-service-sa
```

Expected results for an ordinary application with no Kubernetes API access:

| Operation           | Expected |
| ------------------- | -------- |
| Get ConfigMaps      | No       |
| List Secrets        | No       |
| Create Deployments  | No       |
| Create Pods         | No       |
| Create Roles        | No       |
| Create RoleBindings | No       |

For a workload with an approved ConfigMap reader Role, only the explicitly permitted operation should return `yes`.

## 190.31. IRSA troubleshooting

### Scenario 1 — AWS CLI reports no credentials

Example:

```
Unable to locate credentials
```

Check:

```
kubectl get serviceaccount order-service-sa \
  -n ecommerce \
  -o yaml
```

Verify:

- The IAM role annotation exists.
- The ServiceAccount name is correct.
- The pod uses that ServiceAccount.
- The OIDC provider exists.
- The role trust policy is correct.
- The EKS integration is configured.
- The AWS SDK supports web identity credentials.

Check the pod's environment variable names without printing secret values:

```
kubectl exec <pod-name> -n ecommerce -- \
  sh -c 'printf "AWS_ROLE_ARN=%s\n" "$AWS_ROLE_ARN"; \
         printf "AWS_WEB_IDENTITY_TOKEN_FILE=%s\n" \
         "${AWS_WEB_IDENTITY_TOKEN_FILE:+configured}"'
```

Do not display the token file contents.

### Scenario 2 — AccessDenied from AWS

Example:

```
AccessDenied
```

The pod may have successfully assumed the role, but the role does not have the required permission.

Check:

```
aws iam list-attached-role-policies \
  --role-name ecommerce-dev-order-service-role
```

Inspect inline policies:

```
aws iam list-role-policies \
  --role-name ecommerce-dev-order-service-role
```

Then inspect the exact action and resource in the denied request.

Check:

- IAM identity policies.
- Trust policy.
- Permissions boundary.
- SCP.
- Resource policy.
- KMS key policy if encryption is involved.
- VPC endpoint policy if the request uses an endpoint.

### Scenario 3 — InvalidIdentityToken

Possible causes:

- Incorrect OIDC provider.
- Incorrect issuer URL.
- Wrong audience.
- Expired token.
- Trust policy condition mismatch.
- Wrong ServiceAccount subject.

Check the exact `aud` and `sub` condition keys.

For example, the subject must match:

```
system:serviceaccount:ecommerce:order-service-sa
```

### Scenario 4 — Wrong IAM role is assumed

Inspect:

```
kubectl get pod <pod-name> \
  -n ecommerce \
  -o jsonpath='{.spec.serviceAccountName}'
```

Then inspect the annotation:

```
kubectl get serviceaccount order-service-sa \
  -n ecommerce \
  -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}'
```

Also inspect whether another credential source is taking precedence in the application's SDK configuration.

### Scenario 5 — IAM changes are not reflected

Check whether the pod is using the expected identity and whether the AWS SDK has cached temporary credentials.

For a controlled test, restart the application pod after correcting its ServiceAccount or deployment configuration.

```
kubectl rollout restart deployment/order-service \
  -n ecommerce
```

Do not restart production workloads without considering availability, disruption budgets and the approved change process.

## 190.32. Kubernetes RBAC troubleshooting

### Scenario 1 — Forbidden error

Example:

```
Error from server (Forbidden):
configmaps is forbidden:
User "system:serviceaccount:ecommerce:order-service-sa"
cannot get resource "configmaps"
```

Check the ServiceAccount:

```
kubectl get serviceaccount order-service-sa \
  -n ecommerce
```

Check the Role:

```
kubectl get roles -n ecommerce
```

Check the RoleBinding:

```
kubectl get rolebindings -n ecommerce
```

Check the effective permission:

```
kubectl auth can-i get configmaps \
  -n ecommerce \
  --as=system:serviceaccount:ecommerce:order-service-sa
```

Common causes:

- RoleBinding is in the wrong namespace.
- ServiceAccount name is incorrect.
- Role name is incorrect.
- RoleBinding references a different ServiceAccount.
- The required API group is missing.
- The requested verb is not permitted.
- The application is using a different ServiceAccount.

### Scenario 2 — ServiceAccount not found

Example:

```
serviceaccount "order-service-sa" not found
```

Check:

```
kubectl get serviceaccounts -n ecommerce
```

The ServiceAccount may not have been deployed, or the Deployment may be referencing the wrong namespace or name.

### Scenario 3 — Pod uses default ServiceAccount

Check:

```
kubectl get pod <pod-name> \
  -n ecommerce \
  -o jsonpath='{.spec.serviceAccountName}'
```

If the result is:

```
default
```

Check the Deployment's `serviceAccountName` configuration.

### Scenario 4 — Token not mounted

For an application that needs the Kubernetes API, inspect:

```
kubectl get pod <pod-name> \
  -n ecommerce \
  -o yaml
```

Check:

- ServiceAccount configuration.
- Pod-level `automountServiceAccountToken`.
- Projected token volumes.
- Token expiration and refresh behavior.
- API server connectivity.

For applications that do not need the Kubernetes API, a missing automatic token is expected.

## 190.33. Production incident simulation — excessive permissions

Incident: Security review discovers that Payment Service has permissions to list Secrets across its namespace.

### Situation

The existing Role contains:

```
rules:
  - apiGroups:
      - ""

    resources:
      - secrets

    verbs:
      - get
      - list
      - watch
```

The Payment Service only needs one application configuration value.

### Investigation

Find the binding:

```
kubectl get rolebindings -n ecommerce
```

Inspect the Role:

```
kubectl describe role payment-service-role \
  -n ecommerce
```

Check permissions:

```
kubectl auth can-i list secrets \
  -n ecommerce \
  --as=system:serviceaccount:ecommerce:payment-service-sa
```

### Root cause

A broad RBAC rule was copied from another workload without reviewing the Payment Service requirement.

### Immediate containment

The platform and security teams review whether the permission is actively used.

If it is not required, remove the broad permission through the approved change process.

### Corrective configuration

If the application genuinely needs one ConfigMap, use a narrowly scoped rule:

```
rules:
  - apiGroups:
      - ""

    resources:
      - configmaps

    resourceNames:
      - payment-service-config

    verbs:
      - get
```

### Verification

```
kubectl auth can-i get configmap/payment-service-config \
  -n ecommerce \
  --as=system:serviceaccount:ecommerce:payment-service-sa
```

Expected:

```
yes
```

Verify that Secret access is denied:

```
kubectl auth can-i list secrets \
  -n ecommerce \
  --as=system:serviceaccount:ecommerce:payment-service-sa
```

Expected:

```
no
```

### Preventive action

Add RBAC review and automated policy checks to the pull request process.

## 190.34. Production incident simulation — IRSA access failure

Incident: Invoice generation fails because Invoice Service cannot upload a PDF to S3.

Example application log:

```
S3Exception:
User is not authorized to perform s3:PutObject
```

### Step 1 — Confirm pod identity

```
kubectl get pod <invoice-pod> \
  -n ecommerce \
  -o jsonpath='{.spec.serviceAccountName}'
```

### Step 2 — Inspect ServiceAccount

```
kubectl describe serviceaccount invoice-service-sa \
  -n ecommerce
```

### Step 3 — Confirm assumed role

From a controlled diagnostic container using the same ServiceAccount and identity configuration:

```
aws sts get-caller-identity
```

### Step 4 — Inspect IAM policy

```
aws iam list-attached-role-policies \
  --role-name ecommerce-production-invoice-service-role
```

### Step 5 — Verify resource ARN

Check that the permission resource matches the actual bucket and object path.

For example:

```
arn:aws:s3:::ecommerce-invoice-archive/*
```

### Step 6 — Check encryption permissions

If the bucket requires a customer-managed KMS key, verify that the role and key policy permit the required encryption operation.

### Step 7 — Apply the approved fix

Update the IAM policy through Terraform.

Review the Terraform plan.

Apply through the production approval workflow.

### Step 8 — Verify recovery

Run an application-level invoice generation test.

Check that:

- The PDF is generated.
- The upload succeeds.
- The object is stored in the approved bucket.
- No unrelated bucket permissions were added.

## 190.35. ServiceAccount operational runbook

Create:

`runbooks/serviceaccount-and-irsa-troubleshooting.md`

```
# ServiceAccount and IRSA Troubleshooting

## Purpose

Troubleshoot Kubernetes workload identity and AWS permission failures.

## Symptoms

- Kubernetes API Forbidden
- ServiceAccount not found
- Pod uses default ServiceAccount
- AWS credentials unavailable
- InvalidIdentityToken
- AccessDenied
- Wrong IAM role assumed
- Application cannot access S3, SQS or Secrets Manager

## Step 1: Identify the workload

kubectl get pods -n <namespace>

## Step 2: Check ServiceAccount reference

kubectl get pod <pod-name> \
  -n <namespace> \
  -o jsonpath='{.spec.serviceAccountName}'

## Step 3: Inspect ServiceAccount

kubectl get serviceaccount <service-account> \
  -n <namespace> -o yaml

## Step 4: Check RBAC bindings

kubectl get roles -n <namespace>

kubectl get rolebindings -n <namespace>

kubectl get clusterrolebindings

## Step 5: Test Kubernetes authorization

kubectl auth can-i <verb> <resource> \
  -n <namespace> \
  --as=system:serviceaccount:<namespace>:<service-account>

## Step 6: Check IRSA annotation

Inspect the ServiceAccount role ARN annotation.

Do not expose projected token contents.

## Step 7: Check IAM trust policy

Verify:

- OIDC provider ARN
- OIDC issuer
- Audience condition
- ServiceAccount subject
- Role ARN

## Step 8: Check IAM permissions

Inspect:

- Attached policies
- Inline policies
- Permissions boundaries
- Resource policies
- SCPs
- KMS policies
- VPC endpoint policies

## Step 9: Test AWS identity

aws sts get-caller-identity

Run only from a controlled workload using the same identity.

## Step 10: Test the required application action

Perform a controlled operation against the exact approved resource.

## Step 11: Fix through GitOps and Terraform

Kubernetes identity resources:
- Platform Helm chart
- ArgoCD

AWS IAM resources:
- Terraform
- Reviewed infrastructure pipeline

## Step 12: Verify

Confirm that the required action succeeds and unrelated actions remain denied.

## Escalation

Escalate to Platform Engineering, Cloud Engineering,
Application Engineering and Security as appropriate.
```

## 190.36. ServiceAccount policy review process

Every new permission request should follow this process.

```
Application team identifies requirement
              |
              v
Create permission request
              |
              v
Platform team checks Kubernetes API requirement
              |
              v
Cloud team checks AWS API requirement
              |
              v
Security reviews scope and risk
              |
              v
Terraform / Helm change created
              |
              v
Pull request validation
              |
              v
Approval
              |
              v
GitOps / Terraform deployment
              |
              v
Positive and negative tests
              |
              v
Permission inventory updated
```

### 190.36.1. Permission request template

`templates/service-identity-request.md`

```
# Service Identity Permission Request

## Application

Service name:

Namespace:

Environment:

Application owner:

Technical contact:

## Kubernetes API access

Is Kubernetes API access required?

Yes / No

Required API groups:

Required resources:

Required verbs:

Resource names:

Reason:

## AWS access

Is AWS API access required?

Yes / No

AWS services:

Required actions:

Resource ARNs:

Region:

Reason:

## Security

Does this request involve sensitive data?

Does this request involve production access?

Can the permission be scoped to one resource?

Is there an alternative that avoids API access?

## Testing

Positive test:

Expected result:

Negative test:

Expected result:

## Approval

Application owner:

Platform reviewer:

Cloud reviewer:

Security reviewer:

Production approver:

## Review date

Requested expiry or next review date:
```

### 190.36.2. Permission review frequency

A practical governance schedule:

| Review                            | Frequency      |
| --------------------------------- | -------------- |
| Production IAM role review        | Quarterly      |
| Kubernetes privileged RBAC review | Quarterly      |
| ServiceAccount inventory          | Monthly        |
| New permission request            | Every change   |
| Emergency permission review       | After incident |
| Unused permission analysis        | Periodically   |

The exact frequency should follow the organization's security policy and regulatory obligations.

## 190.37. Reusable ServiceAccount rollout for all 21 microservices

The platform team should create an individual ServiceAccount for each application.

| Microservice   | ServiceAccount              | Kubernetes RBAC default | AWS identity     |
| -------------- | --------------------------- | ----------------------- | ---------------- |
| Auth           | `auth-service-sa`           | None                    | Only if approved |
| User           | `user-service-sa`           | None                    | Only if approved |
| Customer       | `customer-service-sa`       | None                    | Only if approved |
| Catalog        | `catalog-service-sa`        | None                    | Only if approved |
| Product        | `product-service-sa`        | None                    | Only if approved |
| Inventory      | `inventory-service-sa`      | None                    | Only if approved |
| Pricing        | `pricing-service-sa`        | None                    | Only if approved |
| Promotion      | `promotion-service-sa`      | None                    | Only if approved |
| Cart           | `cart-service-sa`           | None                    | Only if approved |
| Order          | `order-service-sa`          | None                    | Only if approved |
| Payment        | `payment-service-sa`        | None                    | Only if approved |
| Fraud          | `fraud-service-sa`          | None                    | Only if approved |
| Shipping       | `shipping-service-sa`       | None                    | Only if approved |
| Delivery       | `delivery-service-sa`       | None                    | Only if approved |
| Notification   | `notification-service-sa`   | None                    | Only if approved |
| Search         | `search-service-sa`         | None                    | Only if approved |
| Recommendation | `recommendation-service-sa` | None                    | Only if approved |
| Review         | `review-service-sa`         | None                    | Only if approved |
| Tax            | `tax-service-sa`            | None                    | Only if approved |
| Invoice        | `invoice-service-sa`        | None                    | Only if approved |
| Reporting      | `reporting-service-sa`      | None                    | Only if approved |

The default is no Kubernetes API permissions.

The AWS role is optional and must be based on actual application requirements.

### 190.37.1. Example generated identity inventory

```
ecommerce namespace
|
+-- auth-service-sa
+-- user-service-sa
+-- customer-service-sa
+-- catalog-service-sa
+-- product-service-sa
+-- inventory-service-sa
+-- pricing-service-sa
+-- promotion-service-sa
+-- cart-service-sa
+-- order-service-sa
+-- payment-service-sa
+-- fraud-service-sa
+-- shipping-service-sa
+-- delivery-service-sa
+-- notification-service-sa
+-- search-service-sa
+-- recommendation-service-sa
+-- review-service-sa
+-- tax-service-sa
+-- invoice-service-sa
+-- reporting-service-sa
```

## 190.38. Testing strategy

We need to test both identity creation and effective authorization.

### 190.38.1. ServiceAccount tests

- Every workload has a dedicated ServiceAccount.
- Every Deployment references the correct ServiceAccount.
- Unnecessary automatic API token mounting is disabled.
- No production application uses the default ServiceAccount unintentionally.
- IRSA annotations are correct where required.

### 190.38.2. RBAC tests

- Applications without Kubernetes API requirements have no unnecessary RBAC.
- Required operations succeed.
- Unapproved operations fail.
- No normal application ServiceAccount has cluster-admin.
- Secret access is restricted.
- Cross-namespace permissions are intentional.

### 190.38.3. AWS identity tests

- OIDC provider is configured.
- Trust policy has the correct audience.
- Trust policy has the correct subject.
- Correct IAM role is assumed.
- Required AWS API action succeeds.
- Unapproved AWS API action is denied.
- Resource ARN is appropriately scoped.
- Production and non-production roles are separated.

### 190.38.4. Pipeline tests

- Helm lint passes.
- Helm rendering succeeds.
- YAML validation passes.
- IAM JSON validation passes.
- Kubernetes schema validation passes.
- IAM policy review passes.
- Production approval is enforced.

## 190.39. Production readiness checklist

Acceptance criteria

- ServiceAccount strategy documented.
- Unique ServiceAccount for each microservice.
- Default ServiceAccount usage reviewed.
- Unnecessary token mounting disabled.
- Kubernetes Role templates created.
- Kubernetes RoleBinding templates created.
- ClusterRole templates controlled.
- ClusterRoleBinding creation restricted.
- RBAC least-privilege rules reviewed.
- Kubernetes permission tests implemented.
- EKS OIDC provider verified.
- IRSA trust policies implemented where required.
- IAM policies scoped to approved resources.
- IAM roles separated by environment.
- Application SDK credential chain verified.
- No long-lived AWS keys in application containers.
- Terraform module created.
- Helm chart created.
- Application charts reference the platform ServiceAccounts.
- GitHub Actions validation implemented.
- ArgoCD deployment configured.
- Production approval process implemented.
- Runbooks created.
- Security audit process documented.

## 190.40. Final deliverables

At the end of Chapter 190, our platform repository should contain:

```
SERVICE IDENTITY PLATFORM
|
+-- Reusable ServiceAccount Helm Chart
|
+-- Kubernetes Role Templates
|
+-- Kubernetes RoleBinding Templates
|
+-- Controlled ClusterRole Templates
|
+-- Controlled ClusterRoleBinding Templates
|
+-- IRSA Trust Policy Definitions
|
+-- AWS IAM Permission Policies
|
+-- Terraform IRSA Module
|
+-- Environment-specific Values
|
+-- 21 Microservice Identity Inventory
|
+-- RBAC Validation
|
+-- IAM Policy Validation
|
+-- GitHub Actions Workflow
|
+-- ArgoCD GitOps Applications
|
+-- Security Review Process
|
+-- Troubleshooting Runbooks
```

Final principle: Every workload must have its own identity, and every identity must have only the permissions required to perform its approved job. Kubernetes RBAC controls Kubernetes API access; AWS IAM controls AWS API access. Neither should be used as a substitute for the other.

## 190.41. How Chapter 190 connects to the next platform components

| Previous chapter                        | Connection                                                       |
| --------------------------------------- | ---------------------------------------------------------------- |
| 182 — Reusable Helm Templates           | ServiceAccount resources are packaged as reusable charts         |
| 183 — Reusable GitHub Actions           | Identity validation can be included in shared CI actions         |
| 184 — Terraform Modules                 | IAM roles and OIDC resources are managed through Terraform       |
| 185 — Reusable GitHub Actions Workflows | Shared validation workflows can be reused                        |
| 186 — Security Policy Templates         | IAM and RBAC checks become security controls                     |
| 187 — Monitoring Templates              | Monitoring workloads also need controlled identities             |
| 188 — Alert Rule Templates              | Alert automation can use dedicated identities                    |
| 189 — NetworkPolicy Templates           | Network permissions and workload identities work together        |
| 191 — GitOps Generators                 | Identity resources can be generated and promoted consistently    |
| 192 — Microservice Golden Path          | Every new application receives a standard identity configuration |

Next chapter — Chapter 191: GitOps Generators

We will build reusable generators for ArgoCD Applications, ApplicationSets, environment-specific values, microservice deployment configuration, Helm chart references, and automated onboarding of all 21 services.

Implementation note: The IAM ARNs, account IDs, bucket names, queue names and application permissions in this chapter are examples. Confirm them against the actual AWS account and application requirements before production deployment. The chapter is a consolidated implementation guide; I have not verified it as a 2,000-line file.