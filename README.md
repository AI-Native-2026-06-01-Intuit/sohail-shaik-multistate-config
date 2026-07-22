# sohail-shaik-multistate-config — GitOps manifests for multistate-api (W6 D2–D5) + W7 D5 agent-svc.

Kustomize overlays and Argo CD CRs for the multistate capstone. The application repo (`sohail-shaik-multi-state`) builds images; this repo is the single source of truth for what runs in-cluster.

## Week 7 Day 5 — Multi-agent service (`multistate-agent-svc`)

- [`multistate-agent-svc/`](multistate-agent-svc/) — base + `overlays/dev` Deployment/Service/ConfigMap/Secret/SA
- [`argocd/applications/multistate-agent-svc-dev.yaml`](argocd/applications/multistate-agent-svc-dev.yaml) — Application → namespace `multistate-svc`
- [`cfn/multistate-agent-anthropic-monthly.yaml`](cfn/multistate-agent-anthropic-monthly.yaml) — $4000 Anthropic budget + AUTOMATIC BudgetsAction deny

```bash
kubectl apply -f argocd/projects/multistate.yaml -n argocd
kubectl apply -f argocd/applications/multistate-agent-svc-dev.yaml -n argocd
argocd app sync multistate-agent-svc
argocd app get multistate-agent-svc

# IAM substrate (works without budgets:ModifyBudget):
aws cloudformation deploy \
  --stack-name multistate-agent-svc-iam-sohail \
  --template-file cfn/multistate-agent-anthropic-monthly.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides IncludeBudget=false

# Full $4000 budget + AUTOMATIC BudgetsAction (needs budgets:ModifyBudget):
aws cloudformation deploy \
  --stack-name multistate-agent-anthropic-monthly \
  --template-file cfn/multistate-agent-anthropic-monthly.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides IncludeBudget=true
```

## Week 6 Day 5 — Observability, Cost Management & Auto-Scaling

Friday production-readiness gate: KEDA ScaledObject + TriggerAuthentication, X-Ray sampling CFN, ADOT dual-export, SLO-derived HPA, Karpenter Spot/On-Demand NodePool, PDB. App twin: k6 load gate + `SRE-CAPSTONE.md` in `sohail-shaik-multi-state`.

- [`k8s/multistate-api/`](k8s/multistate-api/) — worker, ScaledObject, ADOT, HPA, PDB
- [`k8s/karpenter/multistate-mixed.yaml`](k8s/karpenter/multistate-mixed.yaml) — Spot + On-Demand NodePool (cluster-scoped; apply separately)
- [`cfn/multistate-observability-dev.yaml`](cfn/multistate-observability-dev.yaml) — X-Ray `ReservoirSize: 10` / `FixedRate: 0.05`

## Week 6 Day 4 — AWS Managed Services & LLM Cost Monitoring

Cost stack: [`cfn/multistate-cost-dev.yaml`](cfn/multistate-cost-dev.yaml) (Bedrock IRSA, SNS, `CostUsd` alarm, $4200 budget + BudgetsAction, CUR). App twin: `sohail-shaik-multi-state` → [`multistate-api/COST.md`](https://github.com/AI-Native-2026-06-01-Intuit/sohail-shaik-multi-state/blob/main/multistate-api/COST.md).

## Layout

- `base/` — W5 D3 manifests (namespace, deployment, service, configmap, secret, servicemonitor)
- `overlays/{dev,staging,prod}/` — per-env namespace, replicas, Spring profile, log level (+ W6 D5 k8s resources on dev)
- `k8s/` — W6 D5 scaling / tracing manifests
- `argocd/projects/multistate.yaml` — AppProject allow-lists + RBAC
- `argocd/applications/multistate-api-dev.yaml` — API dev Application anchor
- `argocd/applications/multistate-agent-svc-dev.yaml` — W7 D5 agent-svc Application
- `argocd/applicationsets/multistate-api-envs.yaml` — matrix fan-out
- `argocd-system/notifications-cm.yaml` — Slack triggers (failures only)
- `cfn/` — CloudFormation stacks (bootstrap, network, app, artifacts, cost, observability, **agent Anthropic budget** / W7 D5)
- `multistate-agent-svc/` — W7 D5 agent-svc kustomize base + overlays

## Apply order (bootstrap)

```bash
kubectl apply -f argocd/projects/multistate.yaml -n argocd
kubectl apply -f argocd/applicationsets/multistate-api-envs.yaml -n argocd
kubectl apply -f argocd/applications/multistate-agent-svc-dev.yaml -n argocd
kubectl apply -f argocd-system/notifications-cm.yaml -n argocd
```
