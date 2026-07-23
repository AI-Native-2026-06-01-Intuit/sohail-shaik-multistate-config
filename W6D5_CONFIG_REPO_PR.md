# Config-repo PR — paste into GitHub

**Title:** `multistate SRE capstone (config repo) — shaik-sohail-git`

## Summary

Week 6 Day 5 gitops-side production-readiness substrate:

- KEDA `ScaledObject` + operator-IRSA `TriggerAuthentication` on `multistate-nexus-jobs-dev`
- `cfn/multistate-observability-dev.yaml` — X-Ray `ReservoirSize: 10`, `FixedRate: 0.05`
- ADOT dual-export Tempo + X-Ray (`multistate-dev`, AppProject-safe)
- SLO HPA on `multistate_inflight_requests` (avg 7, min 3 / max 28)
- Karpenter `multistate-mixed` Spot+OD + PDB `minAvailable: 2`
- Dev overlay: **no replicas pin fighting HPA**; API/worker **tolerations + nodeSelector** for the NodePool taint

App twin PR: _(paste URL)_

## Customer impact / why this way

- Queue depth scaling clears nexus backlog before summaries go stale for accountants.
- Non-zero X-Ray reservoir keeps Sunday-incident traces instead of a blank map.
- Spot capacity cuts shared cost; PDB + tolerations keep the API floor schedulable through reclaim waves.

## Evidence (fill after live apply / spike)

```text
kubectl -n multistate-dev get scaledobject multistate-worker-scaledobject
aws xray get-sampling-rules --region us-east-1
kubectl -n multistate-dev get hpa multistate-api-hpa
kubectl -n multistate-dev get pdb multistate-api-pdb
kubectl apply -f k8s/karpenter/multistate-mixed.yaml
kubectl get nodepool multistate-mixed -o jsonpath='{.spec.template.spec.requirements}'
```

## AI-tool review

See app-repo PR for the loadtest-author checklist accept/reject paragraph (shared audit).

Config-side decisions vs appendix:

- **Accepted** rubric `minAvailable: 2` (not appendix `maxUnavailable: 25%`).
- **Rejected** ADOT-only-in-`platform-svc` — AppProject destinations are `multistate-*`; dual-export still targets Tempo in `platform-svc`.

## Deploy notes

```bash
aws cloudformation create-change-set \
  --stack-name multistate-observability-dev \
  --change-set-name initial --change-set-type CREATE \
  --template-body file://cfn/multistate-observability-dev.yaml \
  --parameters ParameterKey=EnvName,ParameterValue=dev \
  --region us-east-1

kubectl apply -f k8s/karpenter/multistate-mixed.yaml
# Argo sync multistate-api-dev for namespace-scoped W6 D5 resources
```

## Deliverables checklist

- [x] ScaledObject + TriggerAuthentication (`identityOwner: operator`)
- [x] X-Ray SamplingRule ReservoirSize 10 / FixedRate 0.05
- [x] ADOT dual-export Tempo + X-Ray
- [x] HPA custom metric avg 7; PDB minAvailable 2; Karpenter Spot+OD + limits
- [x] Tolerations/nodeSelector so pool taint does not strand API pods
- [x] Overlay does not pin replicas: 1 against HPA min 3
- [x] Branch `w6d5-implementation`; ≥2 meaningful commits
- [ ] Live Done-when kubectl/aws checks pasted
- [ ] Self-assigned; ES requested as reviewer
- [ ] Cross-linked to app-repo PR

/assign @shaik-sohail-git
