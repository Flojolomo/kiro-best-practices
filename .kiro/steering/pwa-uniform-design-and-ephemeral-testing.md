---
title: PWA Uniform Design & Ephemeral Environment Testing
inclusion: always
---

# PWA Uniform Design & Ephemeral Environment Testing

## Problem Statement

Without enforced design consistency, PWA pages and components drift apart visually over time:
- Pages use different spacing, typography, and layout patterns
- Inline Tailwind classes vary between components doing the same thing
- Shared UI components exist but are bypassed with raw HTML
- Loading, error, and empty states look different on each page
- No single source of truth for colors, spacing scales, or component variants

This steering file establishes standards to prevent visual inconsistency and code duplication in Progressive Web Apps, and defines ephemeral environments as the standard for testing changes before merge.

---

## Design System Foundations

### Design Tokens (REQUIRED)

Define all visual values in a single location. Never use magic color/spacing values inline.

```typescript
// src/styles/tokens.ts or tailwind.config.js theme.extend
export const tokens = {
  colors: {
    primary: { 50: '...', 500: '...', 600: '...', 700: '...' },
    danger: { 50: '...', 500: '...', 600: '...', 700: '...' },
    neutral: { 50: '...', 100: '...', 500: '...', 900: '...' },
  },
  spacing: {
    page: { x: 'px-4 sm:px-6', y: 'py-6' },
    section: 'space-y-4 sm:space-y-6',
    card: 'p-4 sm:p-6',
  },
  radius: {
    card: 'rounded-lg',
    button: 'rounded-md',
    modal: 'rounded-xl',
  },
  shadow: {
    card: 'shadow',
    elevated: 'shadow-lg',
  },
};
```

**Rules:**
- All colors MUST come from the Tailwind config `theme.extend.colors` or a tokens file
- Never write bare hex/rgb values in component files
- Spacing between sections MUST use the standard scale (`space-y-4 sm:space-y-6`)
- Border radius, shadows, and font sizes follow the token system

### Tailwind Configuration

Extend `tailwind.config.js` with project-specific tokens:

```javascript
theme: {
  extend: {
    colors: {
      primary: { /* full scale */ },
      danger: { /* full scale */ },
    },
    spacing: { /* custom values */ },
    screens: { xs: '475px' },
  }
}
```

---

## Shared UI Component Library (REQUIRED)

### Component Inventory

Maintain a canonical set of reusable components in `src/components/ui/`:

| Component | Purpose | Use Instead Of |
|-----------|---------|----------------|
| `Button` | All interactive buttons | Raw `<button>` with inline classes |
| `Card` / `Section` | Content containers | Custom `<div>` with shadow/border |
| `PageHeader` | Page title + actions | Inline `<h1>` / `<h2>` elements |
| `Input` | Form inputs | Raw `<input>` elements |
| `Modal` | Dialogs and confirmations | Custom overlay implementations |
| `Alert` / `ErrorAlert` | Status messages | Custom error `<div>` elements |
| `EmptyState` | No-data placeholders | Inline "no results" text |
| `LoadingSpinner` | Loading indicators | Custom spinners per page |
| `TabNavigation` | Tab interfaces | Custom tab implementations |

### Rules for UI Components

1. **ALWAYS check `src/components/ui/` before creating new UI elements**
2. **NEVER write raw HTML for patterns that exist as shared components**
3. **Extend existing components** via props/variants rather than creating parallel implementations
4. **Export all shared components** from `src/components/ui/index.ts` barrel file
5. **Use composition patterns**: `<Card><Section title="...">content</Section></Card>`
6. Components MUST accept a `className` prop for contextual overrides
7. Components MUST define TypeScript interfaces for all props

### Adding New Shared Components

When you identify a pattern used in 2+ places:
1. Extract to `src/components/ui/ComponentName.tsx`
2. Define a clear TypeScript interface for props
3. Support variants via a `variant` prop (not separate components)
4. Add to the barrel export in `src/components/ui/index.ts`
5. Refactor existing usages to use the new shared component

---

## Page Structure & Layout Consistency (REQUIRED)

### Standard Page Template

Every page MUST follow this structure:

```typescript
export const SomePage: React.FC = () => {
  return (
    <LandingPage>              {/* Shared layout wrapper */}
      <DataCacheProvider>       {/* Data context if needed */}
        <SomePageContent />     {/* Actual page content */}
      </DataCacheProvider>
    </LandingPage>
  );
};

const SomePageContent: React.FC = () => {
  // Early returns for loading/error states
  if (isLoading) return <LoadingSpinner />;
  if (error) return <ErrorAlert message={error} />;

  return (
    <div className="space-y-4 sm:space-y-6">   {/* Standard section spacing */}
      <PageHeader
        title="Page Title"
        description="Optional subtitle"
        actions={<Button>Action</Button>}
      />

      {/* Page content using shared components */}
      <Section title="Section Title">
        {/* content */}
      </Section>
    </div>
  );
};
```

### Layout Rules

- **One layout wrapper per page type**: `LandingPage` for authenticated pages, `AuthLandingPage` for auth flows
- **Pages are thin**: they compose shared components, they don't contain raw HTML
- **Consistent section spacing**: always `space-y-4 sm:space-y-6` between top-level sections
- **PageHeader on every page**: use `<PageHeader>` for title/description/actions, never inline headings
- **Early returns**: loading, error, and empty states use shared components at the top

### State Pattern Consistency

Every page handles these states identically:

| State | Component | Pattern |
|-------|-----------|---------|
| Loading | `<LoadingSpinner />` | Centered with optional message |
| Error | `<ErrorAlert message={...} />` | With retry action |
| Empty | `<EmptyState icon={...} title={...} description={...} />` | Icon + message + optional CTA |
| Success | `<Alert type="success" message={...} />` | Dismissible notification |

---

## DRY Patterns (Don't Repeat Yourself)

### Extract at the 2nd Occurrence

- If you write similar JSX in 2 places, extract a shared component
- If you write similar logic in 3 places, extract a utility function or custom hook
- If you write similar API patterns, use a generic service function

### Scrollable List Pattern

All list views MUST use a common pattern:

```typescript
<ScrollableList
  items={items}
  renderItem={(item) => <ItemComponent {...item} />}
  emptyState={<EmptyState message="No items found" />}
  groupBy={(item) => item.category}     // Optional
  sortBy={{ field: 'date', order: 'desc' }}  // Optional
/>
```

### Form Pattern

All forms MUST use shared `Input`, `Button`, and validation patterns:

```typescript
<form onSubmit={handleSubmit} className="space-y-4">
  <Input id="field" label="Label" error={errors.field} {...register('field')} />
  <div className="flex justify-end space-x-3">
    <Button variant="secondary" onClick={onCancel}>Cancel</Button>
    <Button type="submit" loading={isSubmitting}>Save</Button>
  </div>
</form>
```

### Data Table Pattern

Tables across pages MUST use consistent structure:

```typescript
<div className="bg-white shadow rounded-lg">
  <div className="px-4 sm:px-6 py-4 border-b border-gray-200">
    <h3 className="text-base sm:text-lg font-medium text-gray-900">{title}</h3>
  </div>
  <div className="overflow-x-auto">
    <table className="min-w-full divide-y divide-gray-200">
      {/* Standard thead/tbody structure */}
    </table>
  </div>
</div>
```

If a table is needed in 2+ places, extract a `<DataTable columns={...} data={...} />` component.

---

## PWA-Specific Requirements

### Service Worker & Caching

- Use `vite-plugin-pwa` or Workbox for service worker generation
- Cache the app shell (HTML, CSS, JS) with a cache-first strategy
- Cache API responses with a network-first strategy (fall back to cache when offline)
- Version the cache and clear stale caches on update

### Web App Manifest

Required manifest fields:
```json
{
  "name": "App Full Name",
  "short_name": "AppName",
  "start_url": "/",
  "display": "standalone",
  "theme_color": "#primary-600",
  "background_color": "#ffffff",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

### Mobile-First & Responsive Design

- Design for mobile first, enhance for larger screens
- Use Tailwind responsive prefixes: base (mobile) -> `sm:` -> `md:` -> `lg:` -> `xl:`
- Touch targets: minimum 44x44px for interactive elements
- Safe area handling for iOS standalone mode:
  ```css
  @supports (padding-bottom: env(safe-area-inset-bottom)) {
    body { padding-bottom: env(safe-area-inset-bottom, 0px); }
  }
  ```
- Prevent input zoom on iOS: form inputs must be at least 16px font-size on mobile

### Offline-First UI Indicators

- Show clear online/offline status to users
- Indicate when content is stale (served from cache)
- Queue actions performed offline and sync when connection returns
- Use optimistic UI updates with rollback on failure

---

## Ephemeral Environment Testing (REQUIRED STANDARD)

### Overview

Every pull request MUST be testable in an isolated, ephemeral environment that mirrors production. This environment is automatically deployed on PR open/update and destroyed on PR close/merge.

### Workflow Structure

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

### Required Jobs

| Job | Trigger | Purpose |
|-----|---------|---------|
| `deploy-ephemeral` | PR opened/updated | Deploy isolated stack per PR |
| `cleanup-ephemeral` | PR closed/merged | Destroy stack and clean up resources |
| `diff-against-production` | After deploy | Show infrastructure diff vs production |

### Deployment Requirements

1. **Unique stack per PR**: Use PR number as suffix (e.g., `AppStack-pr-42`)
2. **Full-stack deployment**: Backend + Frontend + Infrastructure (not just frontend preview)
3. **CDK context for isolation**: Pass `--context stage=pr-<number>` to differentiate resources
4. **AWS credentials via OIDC**: Use `id-token: write` permission with role assumption
5. **Frontend deployment**: Build, upload to S3, invalidate CloudFront
6. **Post environment URL to PR**: Comment with frontend URL + API URL for reviewers

### PR Comment Format

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

### Cleanup Requirements

1. **Automatic destruction**: On PR close (merged or not), destroy all resources
2. **Force flag**: Use `--force` on `cdk destroy` to skip confirmation
3. **Post cleanup comment**: Confirm destruction with merged/closed status
4. **No orphaned resources**: Ensure S3 buckets have `autoDeleteObjects: true` and `removalPolicy: DESTROY`

### Infrastructure Diff vs Production

After ephemeral deployment, run `cdk diff` against production to show reviewers what infrastructure changes would be applied on merge:

```yaml
- name: CDK Diff vs Production
  run: |
    npx cdk diff --context stage=prod 2>&1
```

Post the diff as a collapsible section in a PR comment.

### CDK Stack Design for Ephemeral Support

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

### Required GitHub Secrets/Variables

| Variable | Purpose |
|----------|---------|
| `AWS_ROLE_ARN` | OIDC role for ephemeral environment |
| `AWS_PROD_ROLE_ARN` | Read-only role for production diff |
| `AWS_ACCOUNT_ID` | Target AWS account |

### Testing in Ephemeral Environments

Reviewers MUST:
1. Open the ephemeral frontend URL from the PR comment
2. Verify the feature works end-to-end (not just unit tests)
3. Check mobile responsiveness (PWA must work on phone browsers)
4. Verify offline behavior if applicable
5. Review the infrastructure diff for unexpected changes

---

## Anti-Patterns to Avoid

### Design Inconsistency Anti-Patterns

| Anti-Pattern | Correct Approach |
|--------------|-----------------|
| Inline `<div className="bg-white p-6 rounded shadow">` | Use `<Section>` or `<Card>` component |
| Custom loading spinner per page | Use shared `<LoadingSpinner />` |
| Raw `<h1>` / `<h2>` for page titles | Use `<PageHeader title="..." />` |
| Inline error messages with custom styling | Use `<ErrorAlert message="..." />` |
| Magic color values (`text-blue-600`) | Use semantic tokens (`text-primary-600`) |
| Different spacing between pages | Use standard `space-y-4 sm:space-y-6` |
| Duplicated header/footer across page types | Use shared layout wrapper (`LandingPage`) |
| Custom `<button>` with inline classes | Use `<Button variant="..." size="..." />` |

### Code Duplication Anti-Patterns

| Anti-Pattern | Correct Approach |
|--------------|-----------------|
| Copy-paste a component and modify slightly | Add a variant prop to the existing component |
| Same fetch/loading/error pattern in every page | Extract a custom hook (`useAsyncData`) |
| Repeated form field patterns | Use shared `Input` + form hook patterns |
| Same table markup on multiple pages | Extract `<DataTable>` component |
| Duplicated API call wrappers | Use a generic `apiRequest<T>()` utility |

---

## Checklist for New Pages/Components

Before submitting a PR:

- [ ] Page uses the standard layout wrapper (`LandingPage` or `AuthLandingPage`)
- [ ] Page title uses `<PageHeader>` component
- [ ] Loading state uses `<LoadingSpinner />`
- [ ] Error state uses `<ErrorAlert />`
- [ ] Empty state uses `<EmptyState />`
- [ ] All buttons use `<Button>` component with appropriate variant
- [ ] All form inputs use `<Input>` component
- [ ] Section spacing is `space-y-4 sm:space-y-6`
- [ ] No inline color values - all from design tokens
- [ ] No duplicated patterns - checked `src/components/ui/` first
- [ ] Mobile responsive (tested at 375px width)
- [ ] Touch targets are minimum 44x44px
- [ ] Ephemeral environment deploys successfully
- [ ] Feature is manually testable in the ephemeral environment URL
