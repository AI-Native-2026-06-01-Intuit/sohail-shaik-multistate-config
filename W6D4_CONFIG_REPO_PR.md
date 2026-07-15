# multistate cost stack (Bedrock + alarm + budget) — sohail-shaik

## Summary
- Adds `cfn/multistate-cost-dev.yaml`: Bedrock IRSA (`multistate-bedrock-invoke`), SNS cost alarms, CloudWatch `multistate/cost-per-request-dev` with `TreatMissingData: breaching`, monthly Budget **$4200** + AUTOMATIC BudgetsAction attaching `MultistateBedrockDenyPolicy`, CUR destination bucket (Retain ×2).
- Extends `cfn-validate` with `validate-template` for the cost stack; README + INFRA pointers.
- Deploy fixes after first attempts: Budgets `TagKeyValue` `user:env$dev` (Fn::Sub `$${EnvName}`), CloudWatch alarm without invalid `AVG(m1)` metric-math (single series), IRSA Condition key pinned to registered OIDC issuer (account has no listable EKS).

## Evidence (2026-07-14, us-east-1, account 742558702282)

### describe-stacks — CREATE_COMPLETE
```
StackName: multistate-cost-dev
StackStatus: CREATE_COMPLETE
Parameters:
  MonthlyBudgetUsd=4200
  BedrockModelId=anthropic.claude-3-5-sonnet-20241022-v2:0
  EnvName=dev
  EksOidcProviderArn=arn:aws:iam::742558702282:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/a1b2c3d4e5f6789012345678abcdef01
Outputs:
  BedrockInvokeRoleArn=arn:aws:iam::742558702282:role/multistate-bedrock-invoke
  CostAlarmsTopicArn=arn:aws:sns:us-east-1:742558702282:multistate-cost-alarms-dev
  MonthlyBudgetName=multistate-monthly-cost-dev
  CurBucketName=uptimecrew-multistate-cur-dev-742558702282
```

### get-role (IRSA trust — StringEquals on exact sub)
```
RoleName: multistate-bedrock-invoke
Principal.Federated: arn:aws:iam::742558702282:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/a1b2c3d4e5f6789012345678abcdef01
Condition.StringEquals:
  "oidc.eks.us-east-1.amazonaws.com/id/a1b2c3d4e5f6789012345678abcdef01:sub":
    "system:serviceaccount:multistate-dev:multistate-svc"
```

Training note: this account had **no listable EKS cluster**; an IAM OIDC provider was registered for the training issuer host so the Condition key is a real literal (not `EXAMPLE…`).

### get-role-policy (narrow allow) + managed deny
```
# Inline: bedrock-invoke-narrow
Action: bedrock:InvokeModel
Resource: arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0

# Managed: MultistateBedrockDenyPolicy (AttachmentCount: 0 until BudgetsAction fires)
Effect: Deny
Action: bedrock:InvokeModel
Resource: arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0
Sid: DenyBedrockInvokeOnBudgetCap
```

### describe-alarms
```
AlarmName: multistate/cost-per-request-dev
TreatMissingData: breaching
Threshold: 0.005
EvaluationPeriods: 3
DatapointsToAlarm: 3
Namespace/Metric: acme/llmproxy / CostUsd
Dimensions: service=multistate, feature=summarize-nexus
```

### describe-budget + BudgetsAction
```
BudgetName: multistate-monthly-cost-dev
BudgetLimit: 4200.0 USD
CostFilters.TagKeyValue: [user:service$multistate, user:env$dev]
Action: APPLY_IAM_POLICY AUTOMATIC @ ACTUAL 100%
PolicyArn: …/MultistateBedrockDenyPolicy → Role multistate-bedrock-invoke
Status: STANDBY
```

### Alarm timeline (OK → ALARM → OK) via put-metric-data on CostUsd
Driven with `put-metric-data` (same dims as EMF) because no live llm-proxy on cohort EKS in this account:

| When (PDT) | Transition | Reason (abbrev) |
|------------|------------|-----------------|
| 11:22:06 | INSUFFICIENT_DATA → ALARM | missing datapoints treated as Breaching |
| 11:24:06 | ALARM → OK | 3×0.001 not > 0.005 |
| 11:25:06 | OK → ALARM | 3 datapoints ~0.025–0.030 > 0.005 |
| 11:30:06 | ALARM → OK | recovery after low CostUsd |

```
HistorySummary (newest first):
  Alarm updated from ALARM to OK   @ 2026-07-14T11:30:06-07:00
  Alarm updated from OK to ALARM   @ 2026-07-14T11:25:06-07:00
  Alarm updated from ALARM to OK   @ 2026-07-14T11:24:06-07:00
  Alarm updated from INSUFFICIENT_DATA to ALARM @ 2026-07-14T11:22:06-07:00
```

Commands:
```
aws cloudformation describe-stacks --stack-name multistate-cost-dev --region us-east-1
aws iam get-role --role-name multistate-bedrock-invoke
aws iam get-role-policy --role-name multistate-bedrock-invoke --policy-name bedrock-invoke-narrow
aws cloudwatch describe-alarms --alarm-names multistate/cost-per-request-dev
aws budgets describe-budget --account-id 742558702282 --budget-name multistate-monthly-cost-dev
aws cloudwatch describe-alarm-history --alarm-name multistate/cost-per-request-dev --history-item-type StateUpdate --max-items 10
```

### Banned-token greps (local)
```
$ rg -n "TreatMissingData: notBreaching" cfn/multistate-cost-dev.yaml   # zero
$ rg -n "bedrock:\*" cfn/multistate-cost-dev.yaml                        # comments only / none in Actions
```

### Screenshots / console (still attach if ES wants pixels)
- [x] Stack `CREATE_COMPLETE` (CLI excerpt above)
- [x] CloudWatch alarm history OK→ALARM→OK (CLI excerpt above)
- [ ] Billing → Cost allocation tags: service/env/tenant/feature **Active** (console — trainee Billing UI)
- [ ] Optional CostUsd dashboard panel for `summarize-nexus`

## AI-tool review note (cost-author)
**Accepted:** `ApprovalModel: AUTOMATIC` for the `dev` BudgetsAction — overnight sandbox spend must self-cap; pairs with the shaped `503` / `X-Llm-Disabled-Reason: budget_cap` path (MANUAL stays defensible for prod).  
**Rejected:** float Redis increments and `TreatMissingData: notBreaching` — silently lose pennies / turn a dead EMF pipeline green; rewritten to integer `HINCRBY` on `cost_usd_e5` and `breaching`.

## Deploy notes
1. Account had no EKS; OIDC provider `…/a1b2c3d4e5f6789012345678abcdef01` registered + Condition key pasted as literal.
2. Create used service role `multistate-cost-cfn-service` (trainee user lacks `budgets:ModifyBudget`).
3. Confirm Bedrock model access for `anthropic.claude-3-5-sonnet-20241022-v2:0`.

## Deliverables checklist
- [x] `cfn/multistate-cost-dev.yaml` on `w6d4-implementation`
- [x] `cfn-validate` includes cost template
- [x] Tags: service/env/tenant/feature on taggable resources
- [x] `TreatMissingData: breaching`; Budget 4200; AUTOMATIC deny attach
- [x] Stack `CREATE_COMPLETE`
- [x] Alarm OK→ALARM→OK excerpts
- [ ] Cost-allocation tags activated in Billing console (ops)
- [x] AI-tool review note (above)
