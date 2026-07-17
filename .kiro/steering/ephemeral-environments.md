---
title: Ephemeral Environment Testing
inclusion: always
---

# Ephemeral Environment Testing

## Overview

Every pull request MUST be testable in an isolated, ephemeral environment that mirrors production. This environment is automatically deployed when a PR is tagged for deployment and destroyed on PR close/merge.

This is a reusable standard applicable to any project with infrastructure-as-code (CDK, Terraform, CloudFormation) and a frontend deployed to cloud hosting.

---

## Workflow Structure

```yaml
name: Ephemeral Environment

on:
  pull_request:
    types: [labeled, synchronize, closed]
    branches: [main]

concurrency:
  group: ephemeral-pr-${{ github.event.pull_request.number }}
  cancel-in-progress: false
```

**Key design decisions:**
- Trigger on `labeled` to deploy only when a specific tag (e.g., `deploy-ephemeral`) is added to the PR
- Trigger on `synchronize` to redeploy on every push to the PR branch (only if tag is present)
- Use `concurrency` to prevent parallel deployments for the same PR
- Set `cancel-in-progress: false` to avoid partial deployments
- The shared infrastructure stack (databases, user management) is deployed separately from `main` — NOT as part of the ephemeral workflow

---

## Required Jobs

| Job | Trigger | Purpose |
|-----|---------|---------|
| `deploy-ephemeral` | PR tagged + opened/updated | Deploy isolated stack per PR |
| `cleanup-ephemeral` | PR closed/merged | Destroy ephemeral-only resources (NOT shared infra) |
| `diff-against-production` | After deploy | Show infrastructure diff vs production |

---

## Shared vs Ephemeral Resources

### Shared Resources (NEVER destroyed on PR close)

Databases and user management infrastructure are **shared** across all ephemeral environments. These resources:

- Are deployed from `main` branch only, via a separate workflow
- Are deployed only when a release tag is set (e.g., `v*` or `infra-*`)
- Persist across all PR environments
- Use a stable naming convention (no PR suffix)

**Shared resources include:**
- DynamoDB tables / RDS databases
- Cognito User Pools / identity providers
- Any stateful data stores

```typescript
// Shared stack - deployed from main only, never destroyed by ephemeral cleanup
export class SharedInfraStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: StackProps) {
    super(scope, id, props);

    // These resources are NEVER destroyed by ephemeral environment cleanup
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

---

## Deployment Requirements

1. **Tag-gated deployment**: Ephemeral environment deploys ONLY when a label (e.g., `deploy-ephemeral`) is set on the PR
2. **Unique stack per PR**: Use PR number as suffix (e.g., `AppStack-pr-42`)
3. **Full-stack deployment**: Backend + Frontend + Infrastructure (not just frontend preview)
4. **CDK context for isolation**: Pass `--context stage=pr-<number>` to differentiate resources
5. **AWS credentials via OIDC**: Use `id-token: write` permission with role assumption
6. **Frontend deployment**: Build, upload to S3, invalidate CloudFront
7. **Post environment URL to PR**: Comment with frontend URL + API URL for reviewers
8. **Caching disabled**: Ephemeral environments MUST disable all caching (see Caching section below)

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
- Shared stacks (databases, Cognito) are deployed ONLY from `main` and ONLY when a release tag is present
- Shared resources MUST use `RemovalPolicy.RETAIN`
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
  deploy-ephemeral:
    runs-on: ubuntu-latest
    if: >
      github.event.action != 'closed' &&
      contains(github.event.pull_request.labels.*.name, 'deploy-ephemeral')
    permissions:
      id-token: write
      contents: read
      pull-requests: write
    steps:
      - checkout
      - setup-node
      - configure-aws-credentials (OIDC)
      - install dependencies
      - cdk deploy --context stage=pr-<number> --require-approval never --all
      - get stack outputs (CloudFront URL, API URL)
      - deploy frontend (build with VITE_DISABLE_SW=true, s3 sync, CloudFront invalidation)
      - post PR comment with URLs

  cleanup-ephemeral:
    runs-on: ubuntu-latest
    if: github.event.action == 'closed'
    steps:
      - checkout
      - setup-node
      - configure-aws-credentials (OIDC)
      - cdk destroy --context stage=pr-<number> --all --force
        (destroys ONLY ephemeral stack, NOT shared infra)
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

  # SEPARATE WORKFLOW: Shared infrastructure (databases, user pools)
  # Triggered from main branch only, on release tag
  deploy-shared-infra:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && startsWith(github.ref, 'refs/tags/')
    steps:
      - checkout
      - cdk deploy SharedInfraStack --require-approval never
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
| Destroying databases/user pools on PR close | Shared infra deployed separately from `main`, never destroyed by ephemeral cleanup |
| Deploying ephemeral env on every PR automatically | Deploy only when a specific label/tag is set on the PR |
| Enabling caching in ephemeral environments | Disable all caching (CloudFront TTL=0, no service worker, no-cache headers) |
| Creating separate user pools per PR | Share user management across all ephemeral environments |
