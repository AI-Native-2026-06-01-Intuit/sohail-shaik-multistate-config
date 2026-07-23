# Config-repo PR — W7 D5 multistate-agent-svc GitOps + Anthropic budget

## Summary

- Add `multistate-agent-svc/` kustomize base + `overlays/dev`
- Add Argo CD Application `multistate-agent-svc` → namespace `multistate-svc`
- Extend AppProject `multistate` with destination `multistate-svc` + developer sync
- Add `cfn/multistate-agent-anthropic-monthly.yaml` ($4000 + AUTOMATIC BudgetsAction; `IncludeBudget` gate)

## Deployed evidence

```text
aws cloudformation describe-stacks --stack-name multistate-agent-svc-iam-sohail
→ StackStatus=CREATE_COMPLETE (IncludeBudget=false)

kubectl -n argocd get application multistate-agent-svc
→ Application CR present (sync blocked until Argo repo-server/redis healthy + branch pushed)

kubectl apply -k multistate-agent-svc/overlays/dev
→ multistate-svc namespace resources created
```

## Test plan

- [x] `kubectl apply -f argocd/projects/multistate.yaml`
- [x] `kubectl apply -f argocd/applications/multistate-agent-svc-dev.yaml`
- [x] CFN IAM substrate CREATE_COMPLETE
- [ ] After merge: flip Application `targetRevision` to `main` (currently `w7d5-agent-svc` for pre-merge sync)
- [ ] Re-deploy CFN with `IncludeBudget=true` once `budgets:ModifyBudget` is granted
- [ ] Bump image tag away from `pending` after app-repo CI publishes
