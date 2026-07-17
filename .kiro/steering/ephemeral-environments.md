---
title: Ephemeral Environment Testing
inclusion: always
---

# Ephemeral Environment Testing

## Overview

Every pull request MUST be testable in an isolated, ephemeral environment that mirrors production. The ephemeral environment is automatically deployed on every PR and destroyed on PR close/merge.

Databases (primarily DynamoDB) and user management (Cognito) live in **separate shared stacks** that are deployed only when a specific PR label is set. These shared resources are NEVER deleted.

This is a reusable standard applicable to any project with infrastructure-as-code (CDK, Terraform, CloudFormation) and a frontend deployed to cloud hosting.

---

## Workflow Structure

### Ephemeral Stack (deploys on every PR)

```yaml
name: Ephemeral Environment

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches: [main]

concurrency:
  group: ephemeral-pr-${{ github.event.pull_request.number }}
  cancel-in-progress: false
```

### Shared Infrastructure Stack (deploys only on label)

```yaml
name: Shared Infrastructure

on:
  pull_request:
    types: [labeled]
    branches: [main]

# Only runs when 'deploy-shared-infra' label is added
```

**Key design decisions:**
- Ephemeral stack triggers on every PR open/update — no label required
- Shared infrastructure (databases, user management) deploys ONLY when a label (e.g., `deploy-shared-infra`) is added to the PR
- Shared resources are NEVER destroyed — not on PR close, not on cleanup
- Use `concurrency` to prevent parallel deployments for the same PR
- Set `cancel-in-progress: false` to avoid partial deployments

---

## Required Jobs

| Job | Trigger | Purpose |
|-----|---------|---------|
| `deploy-ephemeral` | Every PR open/update | Deploy isolated ephemeral stack per PR |
| `deploy-shared-infra` | PR labeled with `deploy-shared-infra` | Deploy/update shared database and user management stacks |
| `cleanup-ephemeral` | PR closed/merged | Destroy ephemeral stack only (NOT shared infra) |
| `diff-against-production` | After deploy | Show infrastructure diff vs production |

---

## Shared vs Ephemeral Resources

### Shared Resources (NEVER destroyed)

Databases and user management live in **separate CDK stacks** that are shared across all ephemeral environments. These resources:

- Are deployed only when a PR label (e.g., `deploy-shared-infra`) is set
- Are NEVER destroyed — not on PR close, not on merge, not ever by automation
- Persist across all PR environments
- Use a stable naming convention (no PR suffix)
- Use `RemovalPolicy.RETAIN` always

**Shared resources include:**
- DynamoDB tables (primary data store)
- Cognito User Pools / identity providers
- Any stateful data stores

```typescript
// Shared stack - deployed only via PR label, NEVER destroyed
export class SharedInfraStack extends cdk.Stack {
  public readonly userPool: cognito.UserPool;
  public readonly table: dynamodb.Table;

  constructor(scope: Construct, id: string, props: StackProps) {
    super(scope, id, props);

    // These resources are NEVER destroyed
    this.userPool = new cognito.UserPool(this, 'UserPool', {
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    this.table = new dynamodb.Table(this, 'DataTable', {
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });
  }
}
```

### Ephemeral Resources (destroyed on PR close)

Only stateless, per-PR resources are created and destroyed with each ephemeral environment:
- API Gateway / Lambda functions
- S3 buckets (frontend hosting)
- CloudFront distributions
- IAM roles specific to the PR stack

Ephemeral stacks reference shared resources via imports (e.g., `Table.fromTableName()`, `UserPool.fromUserPoolId()`).

---

## Deployment Requirements

1. **Every PR gets an ephemeral environment**: No label required — deploys automatically on PR open/update
2. **Unique stack per PR**: Use PR number as suffix (e.g., `AppStack-pr-42`)
3. **Full-stack deployment**: Backend + Frontend + Infrastructure (not just frontend preview)
4. **CDK context for isolation**: Pass `--context stage=pr-<number>` to differentiate resources
5. **AWS credentials via OIDC**: Use `id-token: write` permission with role assumption
6. **Frontend deployment**: Build, upload to S3, invalidate CloudFront
7. **Post environment URL to PR**: Comment with frontend URL + API URL for reviewers
8. **Caching disabled**: Ephemeral environments MUST disable all caching (see Caching section below)
9. **Shared infra is label-gated**: Database and user management stacks deploy only when `deploy-shared-infra` label is added

---

## PR Comment Format

After successful deployment, post a structured comment:

```markdown
## Ephemeral Environment Deployed

**PR #<number>** - Ready for testing!

| Resource | URL |
|----------|-----|
| Frontend | <cloudfront_url> |
| API | <api_gateway_url> |

---
**Stack:** `AppStack-pr-<number>`
**Commit:** `<sha>`

> This environment will be automatically destroyed when the PR is closed or merged.
```

---

## Cleanup Requirements

1. **Automatic destruction**: On PR close (merged or not), destroy all **ephemeral** resources
2. **NEVER destroy shared resources**: Databases and user management persist — only stateless per-PR resources are torn down
3. **Force flag**: Use `--force` on `cdk destroy` to skip confirmation
4. **Post cleanup comment**: Confirm destruction with merged/closed status
5. **No orphaned resources**: Ensure ephemeral S3 buckets have `autoDeleteObjects: true` and `removalPolicy: DESTROY`

---

## Infrastructure Diff vs Production

After ephemeral deployment, run `cdk diff` against production to show reviewers what infrastructure changes would be applied on merge:

```yaml
- name: CDK Diff vs Production
  run: |
    npx cdk diff --context stage=prod 2>&1
```

Post the diff as a collapsible section in a PR comment.

---

## CDK Stack Design for Ephemeral Support

Stacks MUST support parameterized naming for ephemeral environments:

```typescript
const stage = this.node.tryGetContext('stage') || 'prod';
const stackSuffix = stage !== 'prod' ? `-${stage}` : '';

// All resource names include the suffix
new s3.Bucket(this, 'WebBucket', {
  bucketName: `app-web${stackSuffix}`,
  removalPolicy: stage !== 'prod'
    ? cdk.RemovalPolicy.DESTROY
    : cdk.RemovalPolicy.RETAIN,
  autoDeleteObjects: stage !== 'prod',
});
```

**Design rules for ephemeral stacks:**
- Resource names MUST include the stage suffix to avoid conflicts
- Ephemeral stacks MUST use `RemovalPolicy.DESTROY` for all stateless resources
- S3 buckets in ephemeral stacks MUST have `autoDeleteObjects: true`
- Ephemeral stacks MUST reference shared resources (databases, user pools) via imports, NOT create their own
- CloudFront distributions SHOULD use minimal configuration (no custom domain, no WAF)

**Design rules for shared stacks:**
- Shared stacks (DynamoDB, Cognito) are deployed ONLY when a PR label (`deploy-shared-infra`) is set
- Shared resources MUST use `RemovalPolicy.RETAIN` — they are NEVER deleted
- Shared resources use stable names (no PR suffix)
- Ephemeral stacks receive shared resource references via CDK context or SSM parameters

---

## Caching Policy (REQUIRED)

Ephemeral environments MUST disable all caching to ensure reviewers always see the latest deployed code:

- **CloudFront**: Set `defaultTTL: 0`, `minTTL: 0`, `maxTTL: 0` — or disable caching entirely
- **API Gateway**: No response caching enabled
- **Service Worker**: Do NOT register a service worker in ephemeral builds
- **Browser caching**: Set `Cache-Control: no-cache, no-store, must-revalidate` headers on all responses
- **Build configuration**: Use a build-time environment variable to disable PWA caching in ephemeral builds

```typescript
// CDK: CloudFront with caching disabled for ephemeral environments
const cachePolicy = stage !== 'prod'
  ? new cloudfront.CachePolicy(this, 'NoCachePolicy', {
      defaultTtl: cdk.Duration.seconds(0),
      minTtl: cdk.Duration.seconds(0),
      maxTtl: cdk.Duration.seconds(0),
    })
  : cloudfront.CachePolicy.CACHING_OPTIMIZED;
```

```typescript
// Vite build: disable service worker in ephemeral environments
// vite.config.ts
VitePWA({
  disable: process.env.VITE_DISABLE_SW === 'true',
})
```

---

## Required GitHub Secrets/Variables

| Variable | Purpose |
|----------|---------|
| `AWS_ROLE_ARN` | OIDC role for ephemeral environment |
| `AWS_PROD_ROLE_ARN` | Read-only role for production diff |
| `AWS_ACCOUNT_ID` | Target AWS account |

---

## Testing in Ephemeral Environments

Reviewers MUST:
1. Open the ephemeral frontend URL from the PR comment
2. Verify the feature works end-to-end (not just unit tests)
3. Check mobile responsiveness (PWA must work on phone browsers)
4. Verify offline behavior if applicable
5. Review the infrastructure diff for unexpected changes

---

## Example Workflow Reference

A complete ephemeral environment workflow includes:

```yaml
jobs:
  # Deploys on EVERY PR — no label required
  deploy-ephemeral:
    runs-on: ubuntu-latest
    if: github.event.action != 'closed'
    permissions:
      id-token: write
      contents: read
      pull-requests: write
    steps:
      - checkout
      - setup-node
      - configure-aws-credentials (OIDC)
      - install dependencies
      - cdk deploy EphemeralStack --context stage=pr-<number> --require-approval never
      - get stack outputs (CloudFront URL, API URL)
      - deploy frontend (build with VITE_DISABLE_SW=true, s3 sync, CloudFront invalidation)
      - post PR comment with URLs

  # Deploys ONLY when 'deploy-shared-infra' label is on the PR
  # Resources are NEVER destroyed
  deploy-shared-infra:
    runs-on: ubuntu-latest
    if: >
      github.event.action != 'closed' &&
      contains(github.event.pull_request.labels.*.name, 'deploy-shared-infra')
    steps:
      - checkout
      - setup-node
      - configure-aws-credentials (OIDC)
      - cdk deploy SharedInfraStack --require-approval never
      # NOTE: This stack is NEVER destroyed by any automated workflow

  cleanup-ephemeral:
    runs-on: ubuntu-latest
    if: github.event.action == 'closed'
    steps:
      - checkout
      - setup-node
      - configure-aws-credentials (OIDC)
      - cdk destroy EphemeralStack --context stage=pr-<number> --force
        # Destroys ONLY ephemeral stack — shared infra is untouched
      - post cleanup comment

  diff-against-production:
    runs-on: ubuntu-latest
    needs: deploy-ephemeral
    if: github.event.action != 'closed'
    steps:
      - checkout
      - setup-node
      - configure-aws-credentials (production read-only role)
      - cdk diff --context stage=prod
      - post diff as collapsible PR comment
```

---

## Anti-Patterns

| Anti-Pattern | Correct Approach |
|--------------|-----------------|
| Frontend-only preview (Vercel/Netlify) without backend | Full-stack ephemeral deployment |
| Shared staging environment for all PRs | Isolated stack per PR |
| Manual cleanup of test environments | Automatic destruction on PR close |
| No visibility into infra changes | CDK diff posted to PR |
| Hardcoded resource names preventing parallel stacks | Parameterized names with stage suffix |
| `RemovalPolicy.RETAIN` on ephemeral resources | `RemovalPolicy.DESTROY` + `autoDeleteObjects` |
| Destroying databases/user pools on PR close | Shared infra in separate stack, NEVER destroyed by any automation |
| Requiring a label to deploy ephemeral env | Every PR deploys automatically; only shared infra is label-gated |
| Enabling caching in ephemeral environments | Disable all caching (CloudFront TTL=0, no service worker, no-cache headers) |
| Creating separate user pools/databases per PR | Share databases and user management across all ephemeral environments |
| Putting databases in the ephemeral stack | Databases and user management belong in a separate shared stack |
