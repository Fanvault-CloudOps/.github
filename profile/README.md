# Fanvault-CloudOps

**FanVault** is a cloud-native fan merchandise e-commerce platform built on AWS and Kubernetes. This organization hosts all application services, infrastructure-as-code, and the GitOps delivery layer that run FanVault in production.

---

## Repositories

| Repository | Language | Description |
|---|---|---|
| [fanvault-user-service](https://github.com/Fanvault-CloudOps/fanvault-user-service) | Node.js 18 | Authentication (Cognito) and user profile / shipping address management |
| [fanvault-commerce-service](https://github.com/Fanvault-CloudOps/fanvault-commerce-service) | Node.js 18 | Product catalog, order management, inventory events via SNS / EventBridge |
| [fanvault-ai-service](https://github.com/Fanvault-CloudOps/fanvault-ai-service) | Python 3.12 | AI-powered product metadata generation via Amazon Bedrock (Nova Pro) |
| [fanvault-frontend](https://github.com/Fanvault-CloudOps/fanvault-frontend) | React 18 + Vite | Single-page application served through nginx |
| [Fanvault-GitOps](https://github.com/Fanvault-CloudOps/Fanvault-GitOps) | Helm / YAML | Kubernetes Helm charts and ArgoCD Application manifests for all environments |
| [Terraform-Fanvault-Infra](https://github.com/Fanvault-CloudOps/Terraform-Fanvault-Infra) | Terraform | Complete AWS infrastructure — EKS, Karpenter, Cognito, DynamoDB, S3, ECR, and more |

---

## Architecture Overview

```
<img width="1392" height="1130" alt="github-image" src="https://github.com/user-attachments/assets/04443c82-0ed5-4f8b-b9df-cdd3e499a819" />

```

---

## Microservices

### fanvault-user-service — Node.js 18 / Express
Handles all identity and profile operations. Delegates token verification to AWS Cognito and persists user profiles and shipping addresses in DynamoDB. All secrets are read from AWS Secrets Manager at runtime.

**Key routes:** `POST /api/auth/register` · `POST /api/auth/login` · `GET /api/users/profile` · `PUT /api/users/address`

---

### fanvault-commerce-service — Node.js 18 / Express
Product catalog and order lifecycle management. Generates pre-signed S3 URLs for product images, publishes order events to SNS and inventory-low signals to EventBridge, and queries configuration from SSM Parameter Store.

**Key routes:** `GET /api/products` · `POST /api/orders` · `GET /api/orders/:id` · `POST /api/admin/products`

---

### fanvault-ai-service — Python 3.12 / FastAPI
Generates structured product metadata (title, description, tags) from product images stored in S3. Calls Amazon Bedrock (Nova Pro) via a cross-account assumed role. Emits custom CloudWatch metrics on every inference.

**Key routes:** `POST /generate-metadata` · `GET /health/bedrock`

---

### fanvault-frontend — React 18 / Vite / nginx
Single-page application that consumes the three backend services through the Kubernetes Gateway API. Built with Vite and served via nginx; runtime API endpoints are injected through environment variables at container start via `nginx.conf.template`.

---

## Infrastructure

All infrastructure is provisioned with Terraform from the [Terraform-Fanvault-Infra](https://github.com/Fanvault-CloudOps/Terraform-Fanvault-Infra) repository.

```
Terraform modules
├── vpc                — VPC, subnets, NAT gateway, route tables
├── eks                — EKS 1.35 cluster, managed node group (t2.large)
├── karpenter          — Karpenter controller, NodePool (spot + on-demand, amd64)
├── eks_addons         — CoreDNS, kube-proxy, VPC-CNI, EBS CSI driver
├── cognito            — User pool, app client, domain
├── ecr                — 6 private ECR repos with KMS encryption + lifecycle policy
├── dynamodb           — 4 tables: UserProfiles, Products, Orders, InventoryEvents
├── s3                 — Product images bucket, frontend assets bucket
├── cloudfront         — CDN distribution pointing to frontend S3 origin
├── iam/eks_irsa       — 5 IRSA roles scoped per-service
├── secrets_manager    — Service secrets (db creds, JWT secret, etc.)
├── argocd             — ArgoCD installed via Helm into the cluster
├── observability      — kube-prometheus-stack: Prometheus, Grafana, Alertmanager
├── notifications      — KMS-encrypted SNS topics + SQS queues + DLQ
└── event_processing   — EventBridge rules for inventory and order events
```

---

## CI/CD Pipeline

Each service repository runs an identical GitHub Actions pipeline:

```
push to develop / main
        │
        ├─► SonarQube static analysis  ─┐
        ├─► Snyk dependency scan       ─┤ (parallel)
        │                               │
        └───────────────────────────────┘
                        │
                  Docker Build
               (artifact saved, no push)
                        │
                  Trivy Image Scan
            (blocks on CRITICAL/HIGH/MEDIUM)
                        │
          ┌─────────────┴──────────────┐
          │ develop branch             │ main branch
          │                            │
          ▼                            ▼
   Push dev-{SHA}          ┌─────────────────────┐
   tag to ECR dev          │ environment: prod    │
          │                │ (manual approval)    │
          │                └──────────┬──────────┘
          │                           │
          │                  Push semver + latest
          │                  to ECR prod
          │                           │
          │                  gh release create
          │                  (auto-generated notes)
          │                           │
          └─────────────┬─────────────┘
                        │
                 Helm Updater
           (commits new image tag to
            Fanvault-GitOps values file)
                        │
                   ArgoCD syncs
              (Kubernetes rolls out
               new Deployment revision)
                        │
          develop only ─┤
                        ▼
                  Smoke test
             (HTTP health check
              against dev cluster)
```

Prod deployments require a reviewer with access to the `prod` GitHub environment to click **Approve** before the image is pushed or a release is created. The `rollback-run.yml` workflow in each service repo provides a one-click rollback to any previous image tag.

---

## GitOps Delivery

[Fanvault-GitOps](https://github.com/Fanvault-CloudOps/Fanvault-GitOps) is the single source of truth for what runs in Kubernetes.

```
Fanvault-GitOps/
├── argocd/
│   ├── bootstrap-dev.yaml        — App-of-Apps for dev environment
│   ├── bootstrap-prod.yaml       — App-of-Apps for prod environment
│   ├── apps-dev/                 — ArgoCD Application per service (dev)
│   ├── apps-prod/                — ArgoCD Application per service (prod)
│   └── projects/                 — ArgoCD Project RBAC definitions
├── charts/
│   ├── user-service/             — Helm chart (Deployment, HPA, PDB, NetworkPolicy,
│   ├── commerce-service/           ServiceMonitor, RBAC, ConfigMap, Secret)
│   ├── ai-service/
│   └── frontend/
├── environments/
│   ├── dev/values-{service}.yaml — Dev image tags (updated by CI)
│   └── prod/values-{service}.yaml— Prod image tags (updated by CI after approval)
└── gateway/
    ├── gateway-config.yaml       — Kubernetes Gateway API GatewayClass + Gateway
    ├── httproutes-dev.yaml        — HTTPRoutes for dev.fanvault.garden
    └── httproutes-prod.yaml       — HTTPRoutes for prod
```

ArgoCD polls for changes every ~3 minutes. When CI commits a new `image.tag` to an environment values file, ArgoCD detects the diff and rolls out the new version automatically.

---

## Branching Strategy

```
main ──────────────────────────────────────────────────────► (protected, PRs only)
  ▲                                                           semver release on merge
  │  pull request
  │
develop ──────────────────────────────────────────────────► (protected, PRs only)
  ▲                                                           dev-{SHA} image on push
  │  pull request
  │
feature/my-feature
hotfix/urgent-fix
```

| Branch | Trigger | Image tag | Environment |
|--------|---------|-----------|-------------|
| `feature/*` / `hotfix/*` | PR only — no push | None | None |
| `develop` | Push (post-merge) | `dev-{SHA}` | dev |
| `main` | Push + approval | `v{semver}` + `latest` | prod |

---

## Observability

The cluster runs **kube-prometheus-stack** deployed via Terraform:

- **Prometheus** — scrapes all ServiceMonitors, 20 GiB persistent volume
- **Grafana** — 5 pre-built dashboards (cluster overview, per-service latency/errors, Bedrock inference metrics), 10 GiB persistent volume
- **Alertmanager** — routes alerts to SNS via IRSA, 5 GiB persistent volume
- **VPA** (Vertical Pod Autoscaler) in recommendation mode for all 4 services

Custom metrics emitted by `fanvault-ai-service` to CloudWatch cover Bedrock invocation count, latency, and error rate per model.

---

## Secrets Management

No secrets are baked into images or committed to source control.

| Layer | Mechanism |
|-------|-----------|
| Application runtime | AWS Secrets Manager via IRSA |
| Configuration | SSM Parameter Store (`/fanvault/*`) |
| Kubernetes objects | External Secrets Operator (planned) |
| CI/CD credentials | GitHub Actions encrypted secrets |
| Grafana admin password | SSM parameter injected at Helm install |

---

## Getting Started

> Full setup instructions live in each repository's own README. This is a high-level orientation.

**Provision infrastructure (new environment):**
```bash
git clone https://github.com/Fanvault-CloudOps/Terraform-Fanvault-Infra
cd Terraform-Fanvault-Infra
git checkout dev

terraform -chdir=modules/vpc init && terraform -chdir=modules/vpc apply
terraform -chdir=modules/eks init && terraform -chdir=modules/eks apply
# ... follow the deployment order in Terraform-Fanvault-Infra/README.md
```

**Bootstrap ArgoCD GitOps:**
```bash
kubectl apply -f Fanvault-GitOps/argocd/bootstrap-dev.yaml
# ArgoCD deploys all services automatically
```

**Trigger a deployment:**
```bash
# Merge a PR to develop → CI builds, scans, and deploys to dev automatically
# Merge a PR to main   → CI builds, scans, waits for manual approval, then deploys to prod
```

**Roll back a service:**
```bash
# GitHub Actions → fanvault-{service} repo → Actions → "Rollback" → Run workflow
# Enter: environment (dev/prod) and the image tag to restore
```
