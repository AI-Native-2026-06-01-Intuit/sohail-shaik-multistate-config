# W6 D2 — Config Repo PR

**Title:** `multistate Argo CD GitOps cutover (config repo) — sohail-shaik`

**Branch:** `w6d2-implementation` → `main` (**merged**)

**App-repo PR:** https://github.com/AI-Native-2026-06-01-Intuit/sohail-shaik-multi-state/pull/38 (merged)

/assign me

/cc @yusufumautiauptimecrew

---

## Summary

Bootstraps the GitOps config repo for the multistate capstone: W5 D3 manifests under `base/`, Kustomize overlays for dev/staging/prod, and all seven Argo CD artefacts (`AppProject`, Application anchor, `ApplicationSet`, notifications ConfigMap). Argo CD v2.11.7 on k3d reconciles dev and staging to **Synced + Healthy** (UI screenshot attached).

**Depends on:** app-repo PR merged first (lands `_bump-config.yml` + `GITOPS_REPO_TOKEN`) — **done**.

---

## Customer impact

- **Taxpayers / accountants:** Desired cluster state lives in Git with narrow `AppProject` guardrails — a compromised config repo cannot deploy to `kube-system` or create cluster-scoped privileged resources.
- **On-call engineers:** `on-sync-failed` + `on-health-degraded` only (no deploy-firehose); prod weekend `syncWindows` blocks unattended Friday-night promotion.
- **Next engineer:** Adding a fourth env is one list entry in the ApplicationSet matrix; `preserveResourcesOnDeletion: true` prevents accidental teardown when a generator row is removed.

---

## What's in this PR

| Path | Purpose |
|------|---------|
| `base/` | W5 D3 manifests (namespace through ServiceMonitor) |
| `overlays/{dev,staging,prod}/` | Per-env replicas, log level, `k8s` Spring profile |
| `argocd/projects/multistate.yaml` | AppProject allow-lists, RBAC roles, weekend syncWindow |
| `argocd/applications/multistate-api-dev.yaml` | Dev Application anchor |
| `argocd/applicationsets/multistate-api-envs.yaml` | Matrix generator (list × clusters) |
| `argocd-system/notifications-cm.yaml` | Slack triggers (failures only) |

### k3d lab deviations (documented in app-repo `GITOPS.md`)

- **`clusterResourceWhitelist`:** Namespace-only entry required for `CreateNamespace=true` (Argo CD treats Namespace as cluster-scoped).
- **Namespace manifest removed from `base/`:** Git `Namespace` YAML blocked by empty cluster whitelist; namespaces created via `CreateNamespace` sync option.
- **`k8s` Spring profile** in all overlays (not `dev`/`staging`/`prod`) so the image boots in k3d without Postgres/Redis/Kafka.
- **`ignoreDifferences` on Deployment `/status`:** k8s 1.35 `terminatingReplicas` field breaks Argo CD v2.11 comparison with `ServerSideApply`.

---

## Evidence (k3d lab, 2026-07-10)

### `kubectl -n argocd get applications` (live cluster)

```text
NAME                     SYNC        HEALTH
multistate-api-dev       Unknown     Healthy
multistate-api-staging   Unknown     Healthy
multistate-api-prod      OutOfSync   Missing
```

> **UI screenshot attached** — dev + staging showed **Synced + Healthy** in Argo CD UI during healthy reconciliation window. Transient `Unknown` sync status observed later due to k3d CoreDNS / repo-server timeouts; staging deployment remained **2/2 READY**.

### Deployments reconciled

```bash
kubectl -n multistate-dev get deploy multistate-api
kubectl -n multistate-staging get deploy multistate-api
```

```text
multistate-dev:     READY 1/1  (desired 1 — scaled back after drift test)
multistate-staging: READY 2/2  (desired 2)
```

### AppProject allow-lists (no wildcards)

```bash
kubectl -n argocd get appproject multistate -o yaml | grep -E 'sourceRepos|destinations|clusterResourceWhitelist|syncWindows' -A6
```

```text
  clusterResourceWhitelist:
  - group: ""
    kind: Namespace
  destinations:
  - namespace: multistate-dev
    server: https://kubernetes.default.svc
  - namespace: multistate-staging
    server: https://kubernetes.default.svc
  - namespace: multistate-prod
    server: https://kubernetes.default.svc
  sourceRepos:
  - https://github.com/AI-Native-2026-06-01-Intuit/sohail-shaik-multistate-config.git
  syncWindows:
  - applications:
    - multistate-api-prod
    duration: 60h
    kind: deny
    manualSync: true
    schedule: 0 17 * * 5
```

Prod `OutOfSync` / `Missing` on Friday is **expected** — weekend deny window active (`0 17 * * 5` UTC).

### ApplicationSet `preserveResourcesOnDeletion` + `team=multistate` label

```bash
kubectl -n argocd get applicationset multistate-api-envs -o yaml | grep -E 'preserveResourcesOnDeletion|team:|targetRevision'
kubectl -n argocd get application multistate-api-dev --show-labels
```

```text
    preserveResourcesOnDeletion: true
        team: multistate
        targetRevision: main
labels: env=dev, team=multistate
```

### Notifications ConfigMap (no `on-sync-succeeded`)

```bash
kubectl -n argocd get cm argocd-notifications-cm -o jsonpath='{.data.subscriptions}'
```

```text
- recipients: [slack:multistate-deploys]
  triggers: [on-sync-failed, on-health-degraded]
  selector: team=multistate
```

`argocd-notifications-secret` created out-of-band; **pending** Slack Incoming Webhook (Uptime-Crew-Training workspace at 10-app free plan limit — escalated to ES).

### Rogue Application rejection (Task 2)

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multistate-api-rogue
  namespace: argocd
spec:
  project: multistate
  source:
    repoURL: https://github.com/AI-Native-2026-06-01-Intuit/sohail-shaik-multistate-config.git
    path: overlays/dev
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
EOF
kubectl -n argocd get application multistate-api-rogue -o jsonpath='{range .status.conditions[*]}{.type}{": "}{.message}{"\n"}{end}'
kubectl -n argocd delete application multistate-api-rogue
```

```text
InvalidSpecError: application destination server 'https://kubernetes.default.svc' and namespace 'kube-system' do not match any of the allowed destinations in project 'multistate'
```

`kube-system` is not in AppProject `destinations` — rogue deploy blocked as intended.

### Self-heal controller log (see app-repo PR #38)

```text
level=info msg="Comparing app state (cluster: https://kubernetes.default.svc, namespace: multistate-dev)" application=argocd/multistate-api-dev
level=info msg="Cluster successfully synced" server="https://kubernetes.default.svc"
```

Captured after `kubectl -n multistate-dev scale deployment multistate-api --replicas=3` drift test; deployment returned to **1/1** desired from Git.

### Screenshots

- [x] Argo CD UI — dev + staging Synced/Healthy *(attached)*
- [ ] Slack `#multistate-deploys` — deliberate sync-failed alert *(pending — webhook blocked by workspace app limit; escalated)*

---

## AI-tool review

**Accepted:** `preserveResourcesOnDeletion: true` — safer default when an env falls out of the ApplicationSet matrix.

**Rejected:** `on-sync-succeeded` Slack subscription — deploy-firehose gets muted; alert only on failure/degradation.

---

## Deliverables checklist

- [x] `base/` + `overlays/{dev,staging,prod}/`
- [x] `argocd/projects/multistate.yaml` (narrow allow-lists, roles, syncWindows)
- [x] `argocd/applications/multistate-api-dev.yaml` (anchor)
- [x] `argocd/applicationsets/multistate-api-envs.yaml` (matrix + `preserveResourcesOnDeletion`)
- [x] `argocd-system/notifications-cm.yaml` (no `on-sync-succeeded`)
- [x] Branch `w6d2-implementation` pushed; PR merged to `main`
- [x] PR opened; cross-linked to app PR #38
- [x] Rogue rejection message pasted above
- [x] Self-heal log in app-repo PR #38
- [x] Argo CD UI screenshot attached
- [ ] Slack screenshot attached *(pending ES escalation)*
- [x] ES tagged as reviewer
