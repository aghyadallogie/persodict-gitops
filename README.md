# Persodict GitOps

Helm chart and ArgoCD Applications for the Persodict NestJS API.

**Repos:**
- `Persodict-BE` — application code + Dockerfile + CI
- `persodict-infra` — AWS infrastructure via Terraform
- `persodict-gitops` (this repo) — deployment definitions (Git is truth)

---

## How it works
Persodict-BE push to main
→ CI builds image, pushes to ECR
→ CI bumps image tag in this repo (apps/persodict/chart/testing.yaml)
→ ArgoCD sees the Git change → syncs to cluster
→ pods roll to the new image


CI never touches the cluster. ArgoCD watches this repo and reconciles the cluster to match. Rollback = `git revert`.

---

## File reference

| File | What it does |
|---|---|
| `Chart.yaml` | Chart name (`persodict`), chart version, app version. Bump `version` when chart structure changes. |
| `values.yaml` | Defaults for all envs: image repo, service port (80→3000), resources, probes, autoscaling off. Envs override only what differs. |
| `testing.yaml` | Env overrides: `image.tag`, `ingress.host`, `externalSecret.secretPath`, `replicaCount`. |
| `staging.yaml` | Same shape, staging values. |
| `prod.yaml` | Same shape, prod values + HPA enabled (2–5 replicas). |
| `_helpers.tpl` | Defines `persodict.fullname`, `persodict.labels`, `persodict.selectorLabels`, `persodict.serviceAccountName`. Ensures consistent naming/labels across all resources. |
| `serviceaccount.yaml` | Creates a ServiceAccount named `persodict-<release>`. Pod identity for future IRSA needs. |
| `deployment.yaml` | The pod spec: image from `values.yaml`, env vars (`NODE_ENV`, `PORT`, `DATABASE_URL`, `DEEPL_AUTH_KEY`, `JWT_SECRET`), liveness/readiness probes on `/health:3000`, resource limits. Secrets injected via `secretKeyRef`. `replicas` omitted when HPA is on. |
| `service.yaml` | ClusterIP service on port 80, forwards to pod port named `http` (3000). Ingress targets this, not the pod directly. |
| `externalsecret.yaml` | Tells External Secrets Operator: fetch `persodict/<env>` from AWS Secrets Manager, create a K8s Secret named `persodict-secrets` with keys `DATABASE_URL`, `DEEPL_AUTH_KEY`, `JWT_SECRET`. Refreshes hourly. |
| `ingress.yaml` | Declares an Ingress with class `alb`. AWS Load Balancer Controller turns it into a real ALB. Annotations control scheme (`internet-facing`), target type (`ip`), listen ports, health check path. |
| `hpa.yaml` | Scales pods between `minReplicas` and `maxReplicas` based on CPU utilization. Only created when `autoscaling.enabled: true` (prod only). |

---

## Testing a render locally

```bash
helm template persodict apps/persodict/chart -f apps/persodict/chart/testing.yaml
