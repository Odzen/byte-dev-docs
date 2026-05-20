# Tools & Stack

## Backend

| Tool | Owns | Local | Prod |
|---|---|---|---|
| **Go** | API + worker processes | air hot-reload | ECS Fargate |
| **Temporal** | In-flight workflows (deploy, scaffold, promotion); durable, retries free | Dev server in Compose | Self-hosted on EKS via Helm |
| **GraphQL (gqlgen)** | The single API surface (web + future CLI); subscriptions stream workflow events | `:8080/query` | Load balancer + verified token per request |
| **Datastore** *(open)* | Single source of truth: apps, deployments, releases, connections, build jobs | LocalStack / container | DynamoDB or RDS |

## Frontend

| Tool | Owns | Local | Prod |
|---|---|---|---|
| **TanStack Start** | The dashboard; SPA mode = static output | Vite + HMR | S3 + CloudFront |
| **Tailwind v4 + shadcn/ui** | Design system; components are owned source | PostCSS via Vite | Pre-built CSS |
| **pnpm + workspaces** | TS deps + monorepo | `pnpm install` | Build-time only |

## Local dev

| Tool | Owns |
|---|---|
| **Docker Compose** | Lifecycle of every local dep |
| **LocalStack** | Local stand-in for DynamoDB/S3/Secrets/SSM/IAM — same SDK calls, no network |
| **air** | Go hot-reload |

## Build & runtime (AWS)

| Tool | Owns |
|---|---|
| **CodeBuild** | Customer-app builds, isolated, IAM-scoped |
| **S3** | Two buckets — artifacts (private) and static (CloudFront-fronted). Deploy-scoped prefix = pointer-flip activation |
| **CloudFront** | Public ingress per app; per-app TLS; ~30s global activation |
| **Route 53 + ACM** | DNS + wildcard TLS per env |
| **Secrets Manager + SSM** | Credentials + runtime config; per-app prefix isolation |

Local: LocalStack covers S3/Secrets/SSM. CodeBuild/CloudFront/Route 53 stubbed.

## Observability

| Tool | Owns |
|---|---|
| **ClickStack** (ClickHouse + OTel + HyperDX) | OLAP DB + vendor-neutral ingestion + UI. v1 = control-plane only; absorbs deployed-app telemetry later |
| **CloudWatch Logs** | Lambda + CodeBuild log capture; OTel forwards to ClickHouse |

**Log scrubbing happens in the worker, not downstream.** All subprocess stdout flows through a scrubber before any sink. Secrets never reach ClickStack.

## CI & integrations

| Tool | Owns |
|---|---|
| **GitLab CI** | byte-deploy's own pipeline; Byte already runs GitLab |
| **GitHub App + GitLab OAuth + PAT** | Auth into customers' source repos |

---

## Open decisions

**Datastore — Postgres vs DynamoDB.** Single source of truth for platform state. Small, bounded data; either fits. Switching cost grows fast once the persistence layer is in.

**IaC — Terraform vs Pulumi.** The IaC tool does two things:
1. Provisions byte-deploy's own infra (VPC, ECS, Temporal cluster, shared buckets, Route 53 zone, ACM cert).
2. Called *from inside the Deploy workflow* on **every deploy**. First deploy provisions the app's envelope (CloudFront + S3 prefix + DNS + IAM role + log group; ~10–15 min for CloudFront propagation). Subsequent deploys run a refresh-and-plan that no-ops in ~1 second when nothing has drifted — also catches manual AWS-console changes.

Trade: Pulumi-in-Go gives one language end-to-end; Terraform is the easier hire.

---

## Rejected

Temporal Cloud · Per-tenant AWS accounts · Custom customer domains · Helium CLI as deploy primitive · External pub/sub (Redis, NATS) for subscription fan-out — in-process for v1 · Four-service split (Creator/Lifecycle/Ignition/Builder) — Builder is out of v1 scope; bounded contexts in one Go module give the same separation with a quarter of the ops surface
