---
title: Ephemeral Environment Testing
inclusion: always
---

# Ephemeral Environment Testing

## Overview

Every pull request MUST be testable in an isolated, ephemeral environment that mirrors production. This environment is automatically deployed on PR open/update and destroyed on PR close/merge.

This is a reusable standard applicable to any project with infrastructure-as-code (CDK, Terraform, CloudFormation) and a frontend deployed to cloud hosting.

---

## Workflow Structure

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

**Key design decisions:**
- Trigger on `synchronize` to redeploy on every push to the PR branch
- Use `concurrency` to prevent parallel deployments for the same PR
- Set `cancel-in-progress: false` to avoid partial deployments

---

## Required Jobs

| Job | Trigger | Purpose |
|-----|---------|---------|
| `deploy-ephemeral` | PR opened/updated | Deploy isolated stack per PR |
| `cleanup-ephemeral` | PR closed/merged | Destroy stack and clean up resources |
| `diff-against-production` | After deploy | Show infrastructure diff vs production |

---

## Deployment Requirements

1. **Unique stack per PR**: Use PR number as suffix (e.g., `AppStack-pr-42`)
2. **Full-stack deployment**: Backend + Frontend + Infrastructure (not just frontend preview)
3. **CDK context for isolation**: Pass `--context stage=pr-<number>` to differentiate resources
4. **AWS credentials via OIDC**: Use `id-token: write` permission with role assumption
5. **Frontend deployment**: Build, upload to S3, invalidate CloudFront
6. **Post environment URL to PR**: Comment with frontend URL + API URL for reviewers

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

1. **Automatic destruction**: On PR close (merged or not), destroy all resources
2. **Force flag**: Use `--force` on `cdk destroy` to skip confirmation
3. **Post cleanup comment**: Confirm destruction with merged/closed status
4. **No orphaned resources**: Ensure S3 buckets have `autoDeleteObjects: true` and `removalPolicy: DESTROY`

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
- Non-production stacks MUST use `RemovalPolicy.DESTROY`
- S3 buckets in non-production MUST have `autoDeleteObjects: true`
- DynamoDB tables in non-production SHOULD use `RemovalPolicy.DESTROY`
- CloudFront distributions SHOULD use minimal configuration (no custom domain, no WAF)

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
      - cdk deploy --context stage=pr-<number> --require-approval never --all
      - get stack outputs (CloudFront URL, API URL)
      - deploy frontend (build, s3 sync, CloudFront invalidation)
      - post PR comment with URLs

  cleanup-ephemeral:
    runs-on: ubuntu-latest
    if: github.event.action == 'closed'
    steps:
      - checkout
      - setup-node
      - configure-aws-credentials (OIDC)
      - cdk destroy --context stage=pr-<number> --all --force
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
