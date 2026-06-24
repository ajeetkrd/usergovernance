# user-governance

CDK stack that enforces a **per-user monthly Bedrock spend budget** by removing
over-budget users from a specified AWS IAM Identity Center group.

It scans your existing `BedrockModelUsageTracker` DynamoDB table, computes
each user's USD spend for the current month across all models, and removes
anyone above the configured budget. Runs hourly via EventBridge and can also
be invoked on demand.

## How it works

1. Scans the usage table for items where `sk` ends with `MONTH#<currentMonth>`.
2. For each row, computes cost from `inputTokens`, `outputTokens`,
   `cacheReadTokens`, `cacheWriteTokens` using a per-model price map.
3. Sums each user's cost across all models for the month.
4. If `userMonthlyCostUSD > budgetUsd`, removes that user's membership from the
   configured Identity Center group.

## Configuration

All settings come from CDK context. Set in `cdk.json` or pass with `-c`:

| Key | Required | Default | Notes |
|---|---|---|---|
| `usageTableName` | yes | (set in cdk.json) | DynamoDB table to scan |
| `usageTableRegion` | yes | `ap-south-1` | Region of the usage table |
| `budgetUsd` | yes | `10` | USD per user per month |
| `identityStoreId` | yes | (placeholder) | e.g. `d-xxxxxxxxxx` |
| `identityCenterRegion` | yes | `us-east-1` | Region of Identity Center |
| `groupName` | yes | `BedrockUsers` | Group to enforce |
| `scheduleExpression` | no | `rate(1 hour)` | EventBridge schedule |
| `dryRun` | no | `true` | `true` = log only, no removal |
| `unknownModelPolicy` | no | `warn` | `warn` (treat as $0) or `fail` |
| `pricingOverrides` | no | `{}` | JSON merged on top of defaults |

### Pricing overrides

Pricing keys use normalized model ids (ARN prefix, regional prefix, and version
suffix stripped). Example normalizations:

| Raw `modelId` | Normalized key |
|---|---|
| `anthropic.claude-sonnet-4-20250514-v1:0` | `anthropic.claude-sonnet-4` |
| `arn:aws:bedrock:ap-south-1:...:inference-profile/apac.anthropic.claude-3-5-sonnet-20240620-v1:0` | `anthropic.claude-3-5-sonnet` |

Override an entry:

```bash
cdk deploy -c pricingOverrides='{"anthropic.claude-sonnet-4":{"inputPerM":3,"outputPerM":15,"cacheWritePerM":3.75,"cacheReadPerM":0.3}}'
```

## Deploy

```bash
npm install
npm run build

# First-time only, if account/region not bootstrapped:
npx cdk bootstrap

# Update cdk.json with your real values, or pass via -c on each deploy.
npx cdk deploy \
  -c identityStoreId=d-xxxxxxxxxx \
  -c groupName=BedrockUsers \
  -c budgetUsd=25 \
  -c dryRun=true
```

Start in `dryRun=true`. Inspect CloudWatch logs of the Lambda to confirm
expected behaviour, then redeploy with `dryRun=false`.

## On-demand invocation

```bash
aws lambda invoke \
  --function-name <EnforcerFunctionName-from-output> \
  --region <stack-region> \
  /tmp/out.json
cat /tmp/out.json
```

## IAM permissions granted to the Lambda

- `dynamodb:Scan`, `dynamodb:DescribeTable` on the specific usage table ARN.
- `identitystore:ListGroups`, `GetGroupId`, `ListUsers`, `GetUserId`,
  `ListGroupMemberships`, `ListGroupMembershipsForMember`,
  `DeleteGroupMembership` on the specified identity store and its child
  user / group / membership resources.

## Assumptions

- The `user` attribute in DynamoDB matches the Identity Center `UserName` exactly.
- A single calendar-month budget (UTC) is enforced. No rolling window.
- Removal = removing group membership. The user account is **not** deleted.
- Same AWS account hosts both the usage table and Identity Center.

## Cleanup

```bash
npx cdk destroy
```
