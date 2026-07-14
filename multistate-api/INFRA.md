# multistate-api/INFRA.md

How this service's AWS substrate is provisioned and what deviated from the cfn-author Skill's defaults.

## Stack layout (after W6 D4)

- **multistate-bootstrap-dev** — bootstrap stack (Task 1): bootstrap S3 bucket for templates + CFN-deploy IAM role for GitHub Actions to assume via OIDC.
- **multistate-artifacts-dev** — hardened S3 artefact bucket (Task 3): KMS, PAB, lifecycle (90d → STANDARD_IA, 365d → GLACIER_IR), deny-non-TLS, Retain x2 on both policies.
- **multistate-network-dev** — 3-AZ VPC, 6 subnets (public + private), IGW, 1-3 NAT GWs (Conditions-gated by EnvName), application security group (Task 2).
- **multistate-app-dev** — RDS Postgres + DB SG + DB subnet group + Secrets Manager master credentials (Task 3).
- **multistate-cost-dev** — Bedrock IRSA + SNS cost alarms + CloudWatch alarm (`TreatMissingData: breaching`) + monthly Budget/BudgetsAction + CUR bucket (W6 D4).

## Deploy order

1. **multistate-bootstrap-dev** (Task 1) — bootstrap bucket + CFN-deploy IAM role. One-time per account.
2. **multistate-artifacts-dev** (Task 3, second) — artefact bucket. Independent of network; deploy early so it exists before app-stack writes to it.
3. **multistate-network-dev** (Task 2) — VPC + subnets + SG. Exports VpcId, VpcCidr, PrivateSubnets, PublicSubnets, AppSecurityGroupId.
4. **multistate-app-dev** (Task 3, first) — RDS + Secrets Manager secret + SecretTargetAttachment. Consumes network via `!ImportValue`.

## ChangeSet flow (every stack, every change)

```bash
# 1. Create the change set
aws cloudformation create-change-set \
  --stack-name <stack-name> \
  --change-set-name <descriptive-name> \
  --change-set-type CREATE_OR_UPDATE \
  --template-body file://cfn/<template>.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters ParameterKey=EnvName,ParameterValue=dev \
  --region us-east-1

# 2. Review the diff
aws cloudformation describe-change-set \
  --stack-name <stack-name> \
  --change-set-name <descriptive-name> \
  --region us-east-1
# Paste the JSON output into the PR body for reviewer inspection.

# 3. Execute after approval
aws cloudformation execute-change-set \
  --stack-name <stack-name> \
  --change-set-name <descriptive-name> \
  --region us-east-1

# 4. Wait for completion
aws cloudformation wait stack-create-complete \
  --stack-name <stack-name> \
  --region us-east-1
```

The `describe-change-set` output enumerates every Replace (dangerous), Modify (usually safe), Add, and Remove action per resource. PR reviewers read the diff before the `execute-change-set` step. The CFN-deploy role (from Task 1) is what GitHub Actions assumes via OIDC to run these commands.

## Cross-stack references (why !ImportValue)

The application stack imports network outputs at deploy time via `!ImportValue`:
- `multistate-network-dev-VpcId` → application security group must live in the same VPC
- `multistate-network-dev-PrivateSubnets` → RDS subnet group uses these subnets
- `multistate-network-dev-AppSgId` → DB security group ingresses from the app SG

Benefits:
- **No hardcoded IDs** — if the network stack is rebuilt, outputs change automatically; no template edits.
- **Export safety** — CFN refuses to delete a stack if its exports are in use (`"Export ... is in use by stack ..."`). This caught a dangerous delete attempt in Task 3.
- **Composability** — new services can consume the network exports without touching the network template.

## Drift detection (deliberate verification)

CloudFormation drift detection catches when live resources diverge from the template. The verify step in Task 4:

```bash
# 1. Initiate drift detection
aws cloudformation detect-stack-drift \
  --stack-name multistate-artifacts-dev \
  --region us-east-1
# Returns a StackDriftDetectionId

# 2. Poll until complete
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id <id> \
  --region us-east-1

# 3. Per-resource detail
aws cloudformation describe-stack-resource-drifts \
  --stack-name multistate-artifacts-dev \
  --region us-east-1
```

Expected flow: drift detection starts, runs within 30 seconds, returns DRIFTED if live resources were manually edited in the console, or IN_SYNC if they match the template. A revert (re-running the ChangeSet flow with the original template) brings IN_SYNC status back.

In practice: the artifacts bucket (multistate-artifacts-dev) was verified as IN_SYNC. No manual console edits had drifted it from the template.

## Hardening patterns applied

### S3 bucket (bootstrap + artifacts)
- **Public Access Block (PAB)**: All four toggles on (BlockPublicAcls, BlockPublicPolicy, IgnorePublicAcls, RestrictPublicBuckets).
- **Default encryption**: aws:kms with `alias/aws/s3` (AWS-managed key).
- **Versioning**: Enabled. Allows rollback of accidental deletes/overwrites.
- **Lifecycle rules**: Noncurrent versions expire after 30 days (bootstrap) or 90 days (artifacts). Artifacts also tier to STANDARD_IA at 90d, GLACIER_IR at 365d.
- **Bucket policy**: Deny any request where `aws:SecureTransport` is false (enforce TLS).
- **DeletionPolicy + UpdateReplacePolicy**: Both set to `Retain`. A stack delete or property change that triggers replacement will NOT destroy the bucket.

### VPC / Network
- **3-AZ design**: Subnets spread across three availability zones via `!GetAZs ""` and `!Select [n, ...]`. Failure in one AZ doesn't black out the entire region.
- **CIDR computation**: Subnets carved from the VPC CIDR via `!Cidr [!Ref VpcCidr, 6, 8]` so the template is portable across different VPC CIDR ranges.
- **NAT Gateway count**: Gated by Conditions. Dev gets one NAT (cost), staging/prod get per-AZ NAT (high availability).
- **Security group**: Ingress restricted to 8080 from within the VPC only. Egress permits only 443 (HTTPS). NO `0.0.0.0/0` ingress.

### RDS + Secrets Manager
- **Secrets Manager dynamic reference**: Database master password resolved at deploy time via `{{resolve:secretsmanager:multistate/dev/db-master:SecretString:password}}`. The password never appears in the CloudFormation template, Parameters API, or stack outputs. NEVER use `NoEcho: true` on a password Parameter — the value still lives in the template and Parameters API regardless.
- **Secrets Manager secret structure**: `GenerateSecretString` with `ExcludeCharacters` to avoid quotes/escapes. CFN never sees the plaintext.
- **DB security group**: Ingress 5432 from the app SG only (cross-stack reference).
- **DeletionPolicy + UpdateReplacePolicy**: Retain on both the secret and the RDS instance. A stack delete will not destroy the database.

### IAM (CFN-deploy role)
- **OIDC trust policy**: Assumes via `sts:AssumeRoleWithWebIdentity` against `token.actions.githubusercontent.com`.
- **Sub claim scoping**: Uses `StringLike` (not `StringEquals`) because GitHub OIDC sub claims vary per workflow run. Pinned to the specific gitops repo (`repo:uptimecrew/multistate-config:*`).
- **Aud claim scoping**: Uses `StringEquals` for `sts.amazonaws.com` (exact match).
- **Permission policy**: Enumerates twelve specific CloudFormation actions (create/describe/execute ChangeSet, describe stacks, detect drift, etc.) on `arn:aws:cloudformation:*:*:stack/multistate-*` only. No `Action: '*'`.

## cfn-author Skill audit notes

Ran the `cfn-author` Claude Skill (via `/cfn-author multistate --region us-east-1`) on a scratch branch to scaffold the four stacks. Compared the Skill's output to the assignment's cohort checklist. Here's what the Skill got right and where the human-authored versions diverged:

### Accepted suggestions

1. **DeletionPolicy + UpdateReplacePolicy pairing**: The Skill correctly proposed both policies on stateful resources (S3, RDS secret). This is the safer default — a stack delete or property-change replacement must not destroy live data. CFN interprets these independently; omitting one still loses data on the edge case it covers.

2. **Secrets Manager dynamic reference shape**: The Skill scaffolded `GenerateSecretString` with `SecretStringTemplate` + `GenerateStringKey` correctly. The master password is generated and stored in Secrets Manager; the CFN template never contains plaintext.

3. **Cross-stack Export/ImportValue pattern**: The Skill correctly used `Export.Name` on network outputs and `!ImportValue` in the app stack. The pattern prevents hardcoded IDs and provides delete safety.

### Rejected / corrected suggestions

1. **OIDC trust policy claim conditions**: The Skill initially scaffolded the sub claim with `StringLike: "token.actions.githubusercontent.com:sub": "repo:uptimecrew/multistate-config:*"`. This is correct (GitHub OIDC sub claims vary per run), but the Skill sometimes pairs this with `StringLike` on the `aud` claim where `StringEquals` is required. The human-authored template uses `StringEquals` on `aud` (exact match to `sts.amazonaws.com`) and `StringLike` on `sub` (allows branch/PR refs to vary).

2. **Lifecycle rules in artifacts bucket**: The Skill proposed a single lifecycle rule expiring noncurrent versions. The human-authored template adds tiering: noncurrent versions expire at 90d, but live objects tier to STANDARD_IA at 90d and GLACIER_IR at 365d. This is an operational choice (cost optimization over immediate expiry) that the Skill doesn't capture — the Skill defaults to expire-only.

3. **Bucket policy deny-unencrypted-put statement**: The Skill sometimes omits the second bucket policy statement that denies puts without `s3:x-amz-server-side-encryption: aws:kms`. The human-authored template includes it to enforce KMS encryption on every upload, not just at-rest defaults.

## Why certain choices

- **Conditions for NAT-per-AZ** — Dev uses a single NAT to save ~$45/month; staging/prod use NAT per AZ for failover tolerance. The template parameterizes this via a Condition so one template scales three environments.
- **VPC CIDR via !Cidr** — Portability. If the capstone's VPC CIDR were changed, the template compute subnet CIDRs automatically without manual editing. `!Cidr [!Ref VpcCidr, 6, 8]` carves 6 subnets with /24 masks from a /16.
- **RDS on db.t4g.small** — Cost-conscious default for dev. Staging/prod override with `db.t4g.medium` or `db.m6g.large` via parameter.
- **Secrets Manager over Parameter + NoEcho** — A `NoEcho: true` Parameter hides the value in the console but the plaintext still lives in the template and the AWS::CloudFormation::Stack resource's Parameters attribute (readable via `DescribeStacks`). Secrets Manager keeps the password out of CFN entirely.
