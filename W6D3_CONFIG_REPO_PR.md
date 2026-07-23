# W6 D3 — CloudFormation Infrastructure (Config Repo)

**Title:** `multistate CloudFormation infrastructure — bootstrap, network, app stack, artifacts bucket — sohail-shaik`

**Branch:** `w6d3-implementation` → `main`

**App-repo:** Separate PR in [sohail-shaik-multi-state](https://github.com/AI-Native-2026-06-01-Intuit/sohail-shaik-multi-state)

/assign me

/cc @anthony.kairo

---

## Summary

Provisions complete AWS infrastructure for the multistate capstone via four CloudFormation stacks (all YAML, no CDK):

1. **multistate-bootstrap-dev** — Bootstrap S3 bucket for template storage + CFN-deploy IAM role for GitHub Actions OIDC federation
2. **multistate-network-dev** — 3-AZ VPC (10.43.0.0/16) with 6 subnets (public + private per AZ), IGW, NAT gateway(s) per environment, app security group
3. **multistate-app-dev** — RDS PostgreSQL (db.t4g.small) + Secrets Manager master credentials (dynamic reference) + cross-stack imports
4. **multistate-artifacts-dev** — S3 artefact bucket with KMS encryption, Public Access Block, lifecycle tiering (90d→STANDARD_IA, 365d→GLACIER_IR)

All templates pass **cfn-lint**, **cfn-nag**, and **aws cloudformation validate-template** via GitHub Actions CI (green).

---

## Customer impact

- **Taxpayers / accountants:** Infrastructure as code (Git-auditable) replaces manual console clicks; stack exports ensure consistent naming across environments and prevent orphaned resources via delete-safety constraints.
- **On-call engineers:** Drift detection + ChangeSet review gate prevent silent configuration drift; explicit naming on critical resources (DB, IAM role, security group) prevents accidental replacement during updates.
- **Next engineer:** Adding staging/prod is a parameter override (`EnvName=staging`); NAT-per-AZ gating via Conditions keeps dev cost-conscious (~$45/mo single NAT) while staging/prod are HA. Secrets Manager keeps DB passwords out of CloudFormation templates entirely (never `NoEcho` Parameters).

---

## What's in this PR

| Stack | Resources | Key Features |
|-------|-----------|--------------|
| **multistate-bootstrap-dev** | S3 bucket, IAM role | PAB, KMS encryption, OIDC trust policy (GitHub Actions), enumerate 12 CFN actions only |
| **multistate-network-dev** | VPC, 6 subnets, IGW, NAT GWs (1 dev / 3 HA), app SG | 3-AZ via !GetAZs, subnets via !Cidr, exports for cross-stack refs |
| **multistate-app-dev** | RDS instance, DB subnet group, DB SG, Secrets Manager secret | Dynamic reference for password (no plaintext in template), Retain policies, cross-stack imports |
| **multistate-artifacts-dev** | S3 bucket, bucket policy | PAB, lifecycle rules (noncurrent versions + object tiering), deny-non-TLS policy, Retain policies |

### Hardening applied

- **S3 buckets (bootstrap + artifacts):** Public Access Block (all 4 toggles), default KMS encryption (`alias/aws/s3`), versioning, lifecycle rules, bucket policy denying non-TLS
- **VPC:** 3-AZ design with /24 subnets carved from /16 CIDR; NAT-per-AZ gated by `Conditions` (dev=false, staging/prod=true)
- **RDS:** Storage encryption, Secrets Manager dynamic reference for password, DB security group ingress from app SG only, explicit deny-all egress
- **Security groups:** App SG ingress 8080 from VPC only; egress 443 only (HTTPS to AWS APIs); DB SG ingress 5432 from app SG only
- **IAM role:** OIDC trust policy with `StringEquals` on `aud` (exact match), `StringLike` on `sub` (repo-scoped); 12 enumerated CloudFormation actions, no `Action: '*'`

---

## Evidence

### CI Pipeline (GitHub Actions — cfn-validate.yml)

**All validation jobs green:**

```
✓ cfn-lint (all templates)
✓ cfn-nag (all templates with documented suppressions)
✓ aws cloudformation validate-template (bootstrap)
✓ aws cloudformation validate-template (network)
✓ aws cloudformation validate-template (app)
✓ aws cloudformation validate-template (artifacts)
```

**CI Run:** https://github.com/AI-Native-2026-06-01-Intuit/sohail-shaik-multistate-config/actions/runs/29270721449/job/86887436279 (✅ PASSED)

### cfn-lint validation

No warnings or errors.

### cfn-nag scan (with design-decision suppressions)

**Suppressions documented:**
- **W28 (explicit names):** DbInstance, CfnDeployRole, MultistateAppSecurityGroup — stable resource IDs prevent accidental replacement churn; cross-stack references depend on explicit names
- **W33 (public subnet MapPublicIpOnLaunch):** Required for NAT gateway public IP assignment; public subnets intentionally auto-assign
- **W5 (security group egress 0.0.0.0/0):** HTTPS to AWS APIs (ECR, STS, Secrets Manager, CloudWatch) is intentional and required
- **W35 (AWS-managed KMS):** `alias/aws/s3` is sufficient for dev; customer-managed keys unnecessary for non-production
- **W60 (VPC Flow Logs):** Operational best practice but out of scope for this capstone; can be added post-deployment

All suppressions justified; no unaddressed warnings remain.

### aws cloudformation validate-template

All four templates validated by AWS API:
- multistate-bootstrap-dev.yaml ✓
- multistate-network-dev.yaml ✓
- multistate-app-dev.yaml ✓
- multistate-artifacts-dev.yaml ✓

### Cross-stack reference safety (documented in INFRA.md)

Network stack exports:
```
- VpcId
- VpcCidr
- PrivateSubnets
- PublicSubnets
- AppSgId
```

App stack imports all five via `!ImportValue` and `!Split` pattern. Delete-safety verified: network stack deletion refused with `"Export ... is in use by stack ..."` error.

### ChangeSet flow documented

INFRA.md includes step-by-step ChangeSet workflow:
1. `create-change-set` (create-or-update)
2. `describe-change-set` (review diff for PR)
3. `execute-change-set` (deploy after approval)
4. `wait stack-create-complete` (poll until done)

---

## Commits

| Hash | Message | Purpose |
|------|---------|---------|
| cfba8e4 | fix(cfn): resolve cfn-nag/cfn-lint validation warnings | Fix W77, F1000, W1020 issues; add initial suppressions |
| 0456c36 | fix(cfn): suppress cfn-nag design-decision warnings | Add W28, W33, W5, W35, W60 suppressions with justification |
| ad552eb | ci: switch from OIDC to ambient AWS credentials | Use GitHub secrets instead of role assumption (simpler local testing) |
| b53134f | fix(ci): persist AWS credentials across steps via GITHUB_ENV | Ensure credentials available to all validation steps |

---

## AI-tool review

**Accepted suggestions:**
- DeletionPolicy + UpdateReplacePolicy pairing on stateful resources (S3, RDS secret, RDS instance)
- Secrets Manager `GenerateSecretString` with `SecretStringTemplate` + `GenerateStringKey` (password never appears in template)
- Cross-stack Export/ImportValue pattern for network outputs
- 3-AZ design with `!GetAZs ""` and `!Select` for AZ distribution
- !Cidr for subnet CIDR computation from VPC CIDR (portable across regions/CIDR ranges)

**Rejected / corrected:**
- OIDC trust policy: kept `StringLike` on `sub` (GitHub varies per run), `StringEquals` on `aud` (exact match to `sts.amazonaws.com`)
- Lifecycle rules on artifacts bucket: added tiering (STANDARD_IA at 90d, GLACIER_IR at 365d) beyond expiry-only defaults
- Bucket policy: explicitly deny unencrypted puts (`s3:x-amz-server-side-encryption != aws:kms`) in addition to TLS requirement

---

## Deliverables checklist

- [x] cfn/multistate-bootstrap-dev.yaml (S3 + IAM role)
- [x] cfn/multistate-network-dev.yaml (3-AZ VPC + app SG, exports all 5 required outputs)
- [x] cfn/multistate-app-dev.yaml (RDS + Secrets Manager, cross-stack imports)
- [x] cfn/multistate-artifacts-dev.yaml (S3 artefact bucket, lifecycle + hardening)
- [x] multistate-api/INFRA.md (complete documentation, ChangeSet flow, cfn-author audit notes, hardening patterns, design decisions with rationale)
- [x] .github/workflows/cfn-validate.yml (cfn-lint + cfn-nag + aws cloudformation validate-template)
- [x] All CloudFormation YAML templates pass cfn-lint
- [x] All CloudFormation YAML templates pass cfn-nag (with documented suppressions)
- [x] All templates pass aws cloudformation validate-template
- [x] Branch w6d3-implementation pushed
- [x] CI green (GitHub Actions run #29270721449)
- [x] Cross-stack references documented and tested
- [x] Drift detection pattern documented
- [x] ChangeSet flow documented with examples

## Review follow-ups (2026-07-14)

### 1. CI restored to OIDC

- `cfn-validate.yml` uses `aws-actions/configure-aws-credentials` + `id-token: write` against `arn:aws:iam::742558702282:role/multistate-api-cfn-deploy` (static `AWS_ACCESS_KEY_*` removed).
- Bootstrap defaults / live trust now pin:
  - `GitHubOrg=AI-Native-2026-06-01-Intuit`
  - `GitHubRepo=sohail-shaik-multistate-config`
- Live role previously trusted **`mriganshu-malla-multistate-config`** (wrong trainee fork) — ChangeSet `fix-oidc-trust-20260714102523` updated it.
- Also split `cloudformation:ValidateTemplate` onto `Resource: "*"` so OIDC can run `validate-template` (API is not stack-scoped).

### 2. Drift demo — real CLI output

Baseline was already `IN_SYNC`. Introduced drift by setting `RestrictPublicBuckets=false`, then restored.

**DRIFTED:**
```
StackDriftDetectionId: 9db436b0-7fa9-11f1-aeb6-0e868bfe7efd
StackDriftStatus: DRIFTED
DetectionStatus: DETECTION_COMPLETE
DriftedStackResourceCount: 1
Timestamp: 2026-07-14T17:29:49.723000+00:00
```
Property: `/PublicAccessBlockConfiguration/RestrictPublicBuckets` expected `true` / actual `false`.

**IN_SYNC (after restore):**
```
StackDriftDetectionId: a2e2d010-7fa9-11f1-b529-0affd6f3713f
StackDriftStatus: IN_SYNC
DetectionStatus: DETECTION_COMPLETE
DriftedStackResourceCount: 0
Timestamp: 2026-07-14T17:29:58.417000+00:00
```

