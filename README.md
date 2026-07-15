# sohail-shaik-multistate-config — GitOps manifests for multistate-api (W6 D2–D4).

Kustomize overlays and Argo CD CRs for the multistate capstone. The application repo (`sohail-shaik-multi-state`) builds images; this repo is the single source of truth for what runs in-cluster.

## Week 6 Day 4 — AWS Managed Services & LLM Cost Monitoring

Cost stack: [`cfn/multistate-cost-dev.yaml`](cfn/multistate-cost-dev.yaml) (Bedrock IRSA, SNS, `CostUsd` alarm, $4200 budget + BudgetsAction, CUR). App twin: `sohail-shaik-multi-state` → [`multistate-api/COST.md`](https://github.com/AI-Native-2026-06-01-Intuit/sohail-shaik-multi-state/blob/main/multistate-api/COST.md).

## Layout

- `base/` — W5 D3 manifests (namespace, deployment, service, configmap, secret, servicemonitor)
- `overlays/{dev,staging,prod}/` — per-env namespace, replicas, Spring profile, log level
- `argocd/projects/multistate.yaml` — AppProject allow-lists + RBAC
- `argocd/applications/multistate-api-dev.yaml` — dev Application anchor
- `argocd/applicationsets/multistate-api-envs.yaml` — matrix fan-out
- `argocd-system/notifications-cm.yaml` — Slack triggers (failures only)
- `cfn/` — CloudFormation stacks (bootstrap, network, app, artifacts, **cost** / W6 D4)

## Apply order (bootstrap)

```bash
kubectl apply -f argocd/projects/multistate.yaml -n argocd
kubectl apply -f argocd/applicationsets/multistate-api-envs.yaml -n argocd
kubectl apply -f argocd-system/notifications-cm.yaml -n argocd
```
