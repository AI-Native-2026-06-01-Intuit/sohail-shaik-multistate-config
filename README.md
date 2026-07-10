# sohail-shaik-multistate-config — GitOps manifests for multistate-api (W6 D2).

Kustomize overlays and Argo CD CRs for the multistate capstone. The application repo (`sohail-shaik-multi-state`) builds images; this repo is the single source of truth for what runs in-cluster.

## Layout

- `base/` — W5 D3 manifests (namespace, deployment, service, configmap, secret, servicemonitor)
- `overlays/{dev,staging,prod}/` — per-env namespace, replicas, Spring profile, log level
- `argocd/projects/multistate.yaml` — AppProject allow-lists + RBAC
- `argocd/applications/multistate-api-dev.yaml` — dev Application anchor
- `argocd/applicationsets/multistate-api-envs.yaml` — matrix fan-out
- `argocd-system/notifications-cm.yaml` — Slack triggers (failures only)

## Apply order (bootstrap)

```bash
kubectl apply -f argocd/projects/multistate.yaml -n argocd
kubectl apply -f argocd/applicationsets/multistate-api-envs.yaml -n argocd
kubectl apply -f argocd-system/notifications-cm.yaml -n argocd
```
