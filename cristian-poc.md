# Cristian's POC (Ignition) — Review for Byte Deploy

Code-grounded review of the contractor's POC at `cristian-deploy-poc/` and the RFC at `2026-05-10-deployment-platform-dev-proposal.html`.

**Verdict:** the architectural skeleton was rejected (incompatible with our DynamoDB-as-source-of-truth and per-app envelope model), but eight engineering patterns inside the POC are worth lifting verbatim, and his depth in a handful of areas makes him a strong staffing pick for specific tasks.

---

## What got rejected (don't re-litigate)

Each item below: what it is in his POC, then why it doesn't fit byte-deploy.

### Entity workflow per AWS account as source of truth
**What it is.** His RFC proposes one long-running Temporal workflow per AWS account (workflow ID = `project-{awsAccountID}`). The workflow runs forever; all project + deployment state lives inside the workflow's memory. To do anything you send it a signal. Mutex falls out of `REJECT_DUPLICATE` workflow-ID policy; sequential ordering comes from Temporal's FIFO signal channel.

**Why we reject it.** byte-deploy keeps state in DynamoDB. Workflows are ephemeral executions, not state holders. Putting state in workflow memory means doubling the consistency surface (workflow memory + DynamoDB), querying via Temporal Visibility instead of DynamoDB reads (slower, no good pagination, no GraphQL story), and breaking multi-tenant access patterns like "list all apps for tenant X" — you can't ask N entity workflows to enumerate themselves. See `04-decisions.md`.

### Four-service split (Creator / Lifecycle / Ignition / Builder)
**What it is.** His RFC proposes four separate deployable services: **Creator** scaffolds the project, **Lifecycle** hosts the per-account entity workflow, **Ignition** runs the build + deploy pipeline (the POC itself), and **Builder** is a future visual editor. Each has its own task queue, deployable, CI, secrets, metrics.

**Why we reject it.** For a small team shipping in 6 weeks, four deployables is four times the operational surface for no gain in separation. The same bounded contexts can be enforced inside one Go module via package boundaries (`internal/scaffold`, `internal/deploy`, `internal/builder`). Splitting later is a refactor, not a re-architect. Splitting now is premature.

### ECS Fargate stack (~75 AWS resources per app)
**What it is.** His POC provisions a full container-hosting stack for every deployed app: VPC with public + private subnets, NAT gateway, security groups, an Application Load Balancer with target groups + HTTP/HTTPS listeners, ECS cluster + Fargate task definition + service, IAM roles for task execution + runtime, EFS for SQLite persistence — about 75 individual AWS resources per deploy.

**Why we reject it.** byte-deploy v1 deploys SPAs (TanStack Start in SPA mode) — HTML/JS/CSS files. The serving stack is an S3 prefix + CloudFront distribution + Route 53 record + ACM cert. Five or six resources, not seventy-five. No containers, no load balancer, no VPC. The entire ECS stack is dead weight for what we're shipping.

### GitHub Actions YAML generation + commit-to-user-repo
**What it is.** On first deploy, his POC writes a `.github/workflows/deploy.yml` file into the user's repo and commits + pushes it. Deploys are then triggered by GitHub Actions running in the user's CI, which calls back into Ignition. The user's repo carries the deployment glue.

**Why we reject it.** byte-deploy registers a webhook on the user's repo — no file written, no commit made by the platform. A `git push` fires the webhook → our API receives it → starts the Deploy workflow. The user's repo stays clean (just their code). This is the Vercel/Netlify model. Writing into user repos is invasive, requires elevated write scope on every existing customer repo just to bootstrap, and locks customers into our YAML format.

### 5-step browser wizard
**What it is.** His POC's deploy UI is a multi-step browser form (`internal/domain/wizard/`). The user walks through five sequential screens, each persisting partial config to a server-side session: (1) repo URL + GitHub PAT + name prefix, (2) pick an AWS profile (from the local `~/.aws/config`) + region, (3) domain config, (4) review scan results + confirm detected framework, (5) confirm and launch.

**Why we reject it.** byte-deploy is API-first — the UI is a single form ("New App from template") that submits one GraphQL mutation to start `ScaffoldAndDeploy`. No multi-step session state, no server-side wizard machinery. The CLI eventually submits the same mutation. We don't need a wizard because we don't bring per-user AWS context (next item) and don't scan for framework (the item after).

### Per-user AWS context
**What it is.** Step 2 of his wizard picks an AWS profile from the developer's local `~/.aws/config`. Each deploy runs against whatever AWS account that profile authorizes — every user brings their own AWS.

**Why we reject it.** byte-deploy uses platform-owned AWS credentials via the default credential chain (IAM role for the service). All apps deploy into one byte-owned AWS account; tenant isolation is enforced at the application layer (per-tenant resource prefixes, per-app IAM scoping, per-app envelope). Users never see, pick, or provide an AWS context — they don't need an AWS account to use the platform.

### Framework + dependency scanning
**What it is.** His POC scans the cloned repo to figure out what it is — framework (Next.js? Vite? Go? Static?), package manager (npm / yarn / pnpm), build command, container port, database dependencies (Postgres? SQLite?), AWS service dependencies (SQS? DynamoDB?). The results drive a long branching configuration tree (different Dockerfiles per framework, EFS provisioned only when SQLite is detected, etc.).

**Why we reject it.** byte-deploy only deploys Helium-scaffolded apps. Helium emits a known, uniform tree: TanStack Start SPA, pnpm, `pnpm build` produces `dist/`, no backend, no database. There's nothing to scan because everything is already known. Apps can override `RepoConfig` (build command, output dir) but the platform doesn't need to detect anything.

---

## Scope clarifier — "Deploy" vs "ScaffoldAndDeploy"

byte-deploy has two workflows that often get conflated:

- **Deploy workflow** — clone → ensure envelope (IaC) → CodeBuild → upload to S3 → smoke → CloudFront flip. **No helium anywhere.** This is what fires on every `git push` (and on every `helium deploy` later).
- **ScaffoldAndDeploy workflow** — `helium init` → git init → create remote repo → push → register webhook → then hand off to the Deploy workflow above. Fires only on "New App from template."

Helium lives **only** in ScaffoldAndDeploy. The plain Deploy workflow runs `pnpm build` inside CodeBuild — same as Vercel running your `next build`. Helium is a scaffolder, not a deploy tool.

Cristian's POC has no helium experience, so his work belongs in the **Deploy** workflow (P1.6, P2.5, P2.9, P2.10, P2.4) and **not** in P1.7 (Scaffold activity) or P2.2 (ScaffoldAndDeploy composition). His other strength — running and managing Temporal **workflow code** (retries, cleanup, mutex, idempotency) — is the same skill, just framed differently. Cluster ops (Helm on EKS) is a separate skill set and not where he plugs in.

---

## Patterns to lift — what each one buys us

Each pattern below maps to something the roadmap already requires. Lifting from his POC means it's written, tested, and idiomatic — we don't reinvent it.

| Pattern | Roadmap task | What it gives us vs. what we'd otherwise build |
|---|---|---|
| **Live log broker** | P2.4 | Deploy logs stream into the UI in real time **and survive a tab reload** — users reconnect mid-stream without losing lines. Otherwise we'd build a polling endpoint with a worse UX. |
| **Log scrubber at the source** | P1.10 | Secrets (AWS keys, GitHub tokens, deploy secrets) are stripped **before** any log reaches the UI, ClickStack, or the DB — by construction, not by hope. Otherwise we'd scrub at every sink and pray we caught them all. |
| **Cleanup that survives cancel/crash** | P1.6 | If a deploy is cancelled or the worker crashes, half-provisioned AWS resources still get torn down. No orphan CloudFront distributions, no stranded log groups, no surprise AWS bills. |
| **Secret-safe config type** | P1.2 | Even if a dev accidentally logs the whole config object, secrets show as `***`. No reliance on team discipline — the type itself can't leak. |
| **Skip-if-already-done retries** | P2.2 | Temporal retries don't double-create GitHub repos, webhooks, or initial commits. Irreversible side effects are tracked; retry is always safe. |
| **Stuck-deploy boot sweep** | P1.9 | After a worker restart, the UI shows "interrupted" instead of "running forever." State always matches reality — no zombie rows. |
| **IaC checkpointing** | P2.9 | A half-failed envelope provision resumes from where it broke instead of starting over. No orphan resources, no full re-provision penalty. |
| **Activity-name constants** | P1.4 | Avoids a Go + Temporal footgun where a renamed activity silently mismatches its registered name. Saves hours of cryptic debugging the first time someone hits it. |

The first three are direct code copies (file refs available in his POC if needed). The rest are short idioms — high leverage, low LoC.

---

## Where Cristian helps most

### Tier 1 — direct fits (his POC contains the pattern)

- **P1.6 — Deploy workflow.** His `workflow.go` IS this task. Timeout constants table, deferred cleanup with disconnected context, cancel-vs-fail distinction, retry-policy tuning. **Pair him here.**
- **P2.4 — `deploymentEvents` subscription.** Lift `broker.go` into the GraphQL subscription resolver. Hours, not days.
- **P2.9 — Per-app AWS envelope (IaC orchestration).** He has two full backends in the POC:
  - **Pulumi** (~5,000 LoC across `internal/infra/pulumi/`): broader service coverage (CloudFront, S3, DNS, IAM, RDS, DynamoDB, KMS, SNS, SQS, WAF, ElastiCache, Amplify), Go-native, automation API → no separate binary in the worker image
  - **Terraform** (~6,200 LoC across `internal/infra/terraform/`): more workflow-integration sophistication — per-resource targeted apply, state save/restore (`Reconciler` interface), in-state-or-import-or-create lifecycle, stale-lock recovery
  - Both are real production code, picked at compile time via `//go:build`. He has reps on either. Week 1 spike decides on technical merits.
- **P2.10 — Route 53 + ACM.** Direct AWS expertise: cert reuse, wildcard coverage checks, regional vs us-east-1 split. ~1 day.
- **P2.5 — Smoke test before activation.** His `HealthCheck` activity (DNS-propagation + HTTP-polling, 12-min timeout) is exactly this.

### Tier 2 — strong fits

- **P1.11 — Registry credentials + worker image** — his secret-handling discipline (`LogValuer` + scrubber) is the right mental model for `BYTE_HELIUM_TOKEN` plumbing.
- **P1.9 — Build job tracking** — `MarkInterruptedDeploys` + DBSink for log persistence.
- **P2.6 — Per-app deploy mutex** — `internal/domain/deploy/mutex.go` is ~46 LoC of the same shape (his POC keys by `sessionID`; we'd rekey by `(tenantId, appId)`, but the lock primitive is reusable).
- **P1.4 — Orchestration framework** — activity-name string constants gotcha.
- **P2.2 — ScaffoldAndDeploy** — multi-stage init/commit/push composition pattern from `internal/ghactions` + `internal/clone`.

### Not the right fit (don't burn his time here)

| Task | Gap in his POC |
|---|---|
| P1.2 persistence (DynamoDB single-table) | POC uses Postgres |
| P1.3 tenant-scoped data access | POC is single-tenant |
| P1.5 / P2.7 VCS connections + webhook receivers | POC uses end-user PATs only — no GitHub App, no webhook receivers |
| P2.3 "New App from template" UI | POC frontend is a basic wizard, no TanStack/shadcn |
| Phase 3 (JWKS, RBAC, JWT) | No auth code in the POC |
| ClickStack / HyperDX / OTel wiring | POC observability is Postgres + slog |
| Helium CLI integration | Outside his POC entirely |
