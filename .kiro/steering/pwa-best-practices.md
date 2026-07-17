---
title: Progressive Web App Best Practices
inclusion: always
---

# Progressive Web App (PWA) Best Practices

## Overview

Progressive Web Apps must deliver a native-like experience with consistent design across all devices and pages. All PWA projects use **React 18 + TypeScript + Vite + Tailwind CSS** for the frontend and **AWS CDK v2 + TypeScript + Lambda** for the backend infrastructure.

This steering file defines the required technology stack, project structure, hosting infrastructure (CloudFront + S3), backend configuration flow, multi-environment CDK setup, deployment pipeline, uniform design standards, DRY component architecture, service workers, caching, manifests, mobile-first design, and offline-first architecture.

---

## Required Technology Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Frontend framework | React 18 + TypeScript | Functional components, hooks only |
| Build tool | Vite | With `vite-plugin-pwa` for service worker |
| Styling | Tailwind CSS | Design tokens via `tailwind.config.js` |
| State management | React Context + React Query | No Redux |
| Routing | React Router | Route-level code splitting |
| Infrastructure | AWS CDK v2 + TypeScript | Separate stacks for shared/ephemeral |
| Backend | AWS Lambda (TypeScript) | `NodejsFunction` constructs |
| Database | DynamoDB | Single-table design with GSIs |
| Auth | AWS Cognito | JWT tokens via Amplify client |
| CI/CD | GitHub Actions | Ephemeral envs + production deploy |

---

## Project Directory Structure (REQUIRED)

```
project-root/
├── .github/
│   └── workflows/
│       ├── deploy.yml              # Production deployment
│       ├── ephemeral-env.yml       # Per-PR ephemeral environment
│       └── test.yml                # CI tests
├── .kiro/
│   └── steering/                   # Development standards
├── frontend/
│   ├── public/                     # Static assets (icons, manifest)
│   ├── src/
│   │   ├── components/             # Feature-specific components
│   │   │   ├── ui/                 # Shared reusable UI library
│   │   │   │   ├── Alert.tsx
│   │   │   │   ├── Button.tsx
│   │   │   │   ├── ButtonGroup.tsx
│   │   │   │   ├── Card.tsx
│   │   │   │   ├── EmptyState.tsx
│   │   │   │   ├── Error.tsx
│   │   │   │   ├── IconButton.tsx
│   │   │   │   ├── Input.tsx
│   │   │   │   ├── Modal.tsx
│   │   │   │   ├── PageHeader.tsx
│   │   │   │   ├── Section.tsx
│   │   │   │   ├── TabNavigation.tsx
│   │   │   │   └── index.ts       # Barrel export for all UI components
│   │   │   ├── __tests__/          # Component tests
│   │   │   ├── FeatureComponent.tsx
│   │   │   └── index.ts           # Barrel export for feature components
│   │   ├── contexts/               # React Context providers
│   │   ├── hooks/                  # Custom hooks (useAuth, useNetworkStatus, etc.)
│   │   ├── layouts/                # Layout wrappers (DashboardLayout, AuthLayout)
│   │   ├── pages/                  # Route-level page components
│   │   ├── providers/              # App-level providers (AppProviders wrapper)
│   │   ├── routes/                 # Route definitions & guards (ProtectedRoute)
│   │   ├── types/                  # TypeScript type definitions
│   │   ├── utils/                  # Business logic, API client, services
│   │   ├── App.tsx                 # Root component with routing
│   │   ├── index.css               # Tailwind directives + global styles
│   │   └── main.tsx                # Entry point
│   ├── index.html                  # HTML template
│   ├── jest.config.js              # Test configuration
│   ├── package.json
│   ├── postcss.config.js
│   ├── tailwind.config.js          # Design tokens source of truth
│   ├── tsconfig.json
│   └── vite.config.ts              # Vite + PWA plugin config
├── infrastructure/
│   ├── bin/                        # CDK app entry point
│   ├── lib/                        # CDK stacks
│   │   ├── app-stack.ts            # Ephemeral stack (API, Lambda, S3, CloudFront)
│   │   ├── shared-stack.ts         # Shared stack (DynamoDB) — NEVER destroyed
│   │   ├── auth-stack.ts           # Shared auth stack (Cognito) — NEVER destroyed
│   │   └── github-oidc-stack.ts    # GitHub OIDC for CI/CD
│   ├── lambda/                     # Lambda function handlers
│   │   ├── functionName/
│   │   │   ├── src/
│   │   │   │   └── index.ts       # Handler entry point
│   │   │   ├── package.json       # Isolated dependencies
│   │   │   └── tsconfig.json
│   │   └── anotherFunction/
│   │       └── ...
│   ├── cdk.json
│   ├── jest.config.js
│   ├── package.json
│   └── tsconfig.json
├── package.json                    # Root workspace config & scripts
└── README.md
```

### Directory Rules

- **Frontend and infrastructure are sibling directories** — never nested
- **Each Lambda function has its own directory** with isolated `package.json` and `tsconfig.json`
- **CDK stacks are separated by lifecycle**: shared (auth, database) vs ephemeral (API, frontend hosting)
- **Components follow a two-tier structure**: `components/ui/` for generic reusable UI, `components/` for feature-specific components
- **All shared UI components MUST be exported** from `components/ui/index.ts`
- **Pages are thin** — they compose components, not contain raw HTML
- **Hooks, contexts, utils, and types each get their own directory** — never mix concerns
- **Tests co-locate with source** in `__tests__/` subdirectories or `.test.ts` files
- **Config files live at the package root** (`frontend/`, `infrastructure/`), not in `src/`

### Naming Conventions

| Item | Convention | Example |
|------|-----------|---------|
| Components | PascalCase | `ActiveTimerWidget.tsx` |
| Hooks | camelCase with `use` prefix | `useAuth.ts`, `useNetworkStatus.ts` |
| Utils/services | camelCase | `timeRecordService.ts`, `apiClient.ts` |
| Types | PascalCase (file), PascalCase (exports) | `types/index.ts`, `TimeRecord` |
| Pages | PascalCase with `Page` suffix | `ActiveTimerPage.tsx`, `ProfilePage.tsx` |
| Contexts | PascalCase with `Context` suffix | `ActiveTimerContext.tsx` |
| Lambda handlers | camelCase (directory) | `lambda/timeRecords/`, `lambda/todos/` |
| CDK stacks | kebab-case (file), PascalCase (class) | `auth-stack.ts`, `AuthStack` |

---

## Design Token System (REQUIRED)

### Single Source of Truth for Visual Values

Define all visual values in ONE location. Never use magic values inline.

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

### Token Rules

- All colors MUST come from the Tailwind config `theme.extend.colors` or a tokens file
- Never write bare hex/rgb values in component files
- Spacing between sections MUST use the standard scale (`space-y-4 sm:space-y-6`)
- Border radius, shadows, and font sizes follow the token system
- Tailwind config is the source of truth:
  ```javascript
  theme: {
    extend: {
      colors: { primary: { /* full scale */ }, danger: { /* full scale */ } },
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
    <LayoutWrapper>             {/* Shared layout wrapper */}
      <DataProvider>             {/* Data context if needed */}
        <SomePageContent />      {/* Actual page content */}
      </DataProvider>
    </LayoutWrapper>
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

- **One layout wrapper per page type**: e.g., `DashboardLayout` for authenticated, `AuthLayout` for auth flows
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

## DRY Principles (Don't Repeat Yourself)

### The 2-Occurrence Rule

- If you write similar JSX in **2 places**, extract a shared component
- If you write similar logic in **3 places**, extract a utility function or custom hook
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

- List container handles: scrolling, grouping, sorting, empty states
- Item components handle: display, actions, item-specific state

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

Tables across pages MUST use consistent structure. If a table is needed in 2+ places, extract a `<DataTable>` component:

```typescript
<DataTable
  columns={[
    { key: 'name', label: 'Name', sortable: true },
    { key: 'duration', label: 'Duration' },
    { key: 'percentage', label: '%' },
  ]}
  data={items}
  striped
  responsive
/>
```

### Composition Over Inheritance

```typescript
// GOOD: Composition via children and slots
<Card>
  <CardHeader title="Title" actions={<Button>Edit</Button>} />
  <CardContent>
    <Section title="Details">{content}</Section>
  </CardContent>
</Card>

// BAD: Monolithic component with many props
<Card title="Title" actions={...} content={...} footer={...} variant="..." />
```

---

## Hosting Infrastructure — CloudFront + S3 (REQUIRED)

Every new PWA MUST include CloudFront + S3 hosting infrastructure defined in CDK. This is not optional — the hosting stack is part of the project from day one.

### Architecture Overview

```
┌─────────────┐     ┌──────────────────┐     ┌────────────┐
│   Browser   │────▶│   CloudFront     │────▶│  S3 Bucket │
│             │     │  (HTTPS + OAC)   │     │  (private) │
└─────────────┘     └──────────────────┘     └────────────┘
                           │
                    ┌──────┴──────┐
                    │ SPA Routing │
                    │  Function   │
                    └─────────────┘
```

- **S3 Bucket**: Hosts the built frontend assets (HTML, JS, CSS, images)
- **CloudFront**: Provides HTTPS, global CDN, and caching layer
- **Origin Access Control (OAC)**: Ensures S3 is only accessible through CloudFront
- **SPA Routing Function**: CloudFront Function that routes all non-file requests to `index.html`

### S3 Bucket Configuration

```typescript
const websiteBucket = new s3.Bucket(this, 'WebsiteBucket', {
  // OAC handles access — no public read needed
  websiteIndexDocument: 'index.html',
  websiteErrorDocument: 'index.html',
  publicReadAccess: true,
  blockPublicAccess: new s3.BlockPublicAccess({
    blockPublicAcls: false,
    blockPublicPolicy: false,
    ignorePublicAcls: false,
    restrictPublicBuckets: false,
  }),
  // Ephemeral environments: DESTROY + autoDelete
  // Production: RETAIN
  removalPolicy: stage !== 'prod'
    ? cdk.RemovalPolicy.DESTROY
    : cdk.RemovalPolicy.RETAIN,
  autoDeleteObjects: stage !== 'prod',
});
```

### SPA Routing Function (REQUIRED)

Every PWA with client-side routing MUST include a CloudFront Function to handle SPA routing. Without this, deep links and page refreshes return 404:

```typescript
const spaRoutingFunction = new cloudfront.Function(this, 'SpaRoutingFunction', {
  code: cloudfront.FunctionCode.fromInline(`
function handler(event) {
    var request = event.request;
    var uri = request.uri;
    
    // If the URI has a file extension, serve it directly (static assets)
    if (uri.match(/\\\\.[a-zA-Z0-9]+$/)) {
        return request;
    }
    
    // For paths without extensions, serve index.html (SPA routing)
    request.uri = '/index.html';
    return request;
}
  `),
});
```

### CloudFront Distribution

```typescript
// Optional: response headers for cache control per environment
let responseHeadersPolicy: cloudfront.IResponseHeadersPolicy | undefined;
if (props.cacheMaxAge) {
  responseHeadersPolicy = new cloudfront.ResponseHeadersPolicy(this, 'CacheHeaders', {
    responseHeadersPolicyName: `${appName}-cache-${stage}`,
    customHeadersBehavior: {
      customHeaders: [{
        header: 'cache-control',
        value: `public, max-age=${props.cacheMaxAge}`,
        override: true,
      }],
    },
  });
}

const distribution = new cloudfront.Distribution(this, 'Distribution', {
  defaultBehavior: {
    origin: origins.S3BucketOrigin.withOriginAccessControl(websiteBucket),
    viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
    cachePolicy: props.cachePolicy ?? cloudfront.CachePolicy.CACHING_OPTIMIZED,
    responseHeadersPolicy,
    functionAssociations: [{
      function: spaRoutingFunction,
      eventType: cloudfront.FunctionEventType.VIEWER_REQUEST,
    }],
  },
  defaultRootObject: 'index.html',
});
```

### Required Stack Outputs

Every hosting stack MUST export these outputs for use in deployment scripts and CI/CD:

```typescript
new cdk.CfnOutput(this, 'WebsiteUrl', {
  value: `https://${distribution.distributionDomainName}`,
  description: 'CloudFront Distribution URL',
  exportName: `${this.stackName}-CloudFrontURL`,
});

new cdk.CfnOutput(this, 'S3BucketName', {
  value: websiteBucket.bucketName,
  description: 'S3 Bucket Name for frontend deployment',
});

new cdk.CfnOutput(this, 'CloudFrontDistributionId', {
  value: distribution.distributionId,
  description: 'CloudFront Distribution ID for cache invalidation',
});
```

### Hosting Infrastructure Rules

1. **Every PWA project includes CloudFront + S3 from the start** — never deploy frontend assets any other way
2. **Use OAC (Origin Access Control)** via `S3BucketOrigin.withOriginAccessControl()` — never OAI (legacy)
3. **Include the SPA routing function** — all non-file paths must resolve to `index.html`
4. **Separate cache policy per environment**: `CACHING_DISABLED` for dev/ephemeral, `CACHING_OPTIMIZED` for production
5. **Export CloudFront URL, S3 bucket name, and distribution ID** as stack outputs
6. **The CloudFront URL is the canonical frontend URL** — all OAuth callback URLs and CORS origins must include it

---

## Backend Configuration Flow (REQUIRED)

PWAs with backend services MUST use a runtime configuration pattern to pass backend stack outputs (API URLs, Cognito config, etc.) to the frontend. This eliminates build-time environment variables and allows the same frontend bundle to work across environments.

### Pattern: `amplify_outputs.json` deployed to S3

The backend stack generates a JSON configuration file and deploys it to the same S3 bucket as the frontend. The frontend fetches this file at runtime.

```
┌────────────────┐    CDK deploys     ┌────────────┐
│  Backend Stack │──────────────────▶  │  S3 Bucket │
│ (API, Cognito) │  amplify_outputs   │            │
└────────────────┘       .json        └────────────┘
                                            │
                                      CloudFront
                                            │
                                      ┌─────▼─────┐
                                      │  Frontend  │
                                      │ fetches at │
                                      │  runtime   │
                                      └────────────┘
```

### CDK: Generate and Deploy Configuration

In the app stack (where CloudFront + API Gateway are defined), generate the config JSON and deploy it to S3:

```typescript
// Build the configuration object from stack resources
const amplifyOutputsJson = {
  version: "1",
  auth: {
    aws_region: this.region,
    user_pool_id: auth.userPool.userPoolId,
    user_pool_client_id: userPoolClient.userPoolClientId,
    password_policy: {
      min_length: 8,
      require_lowercase: true,
      require_uppercase: true,
      require_numbers: true,
      require_symbols: false,
    },
    oauth: {
      identity_providers: ["COGNITO"],
      domain: `${auth.userPoolDomain.domainName}.auth.${this.region}.amazoncognito.com`,
      scopes: ["email", "openid", "profile"],
      redirect_sign_in_uri: oauthUrls,
      redirect_sign_out_uri: oauthUrls,
      response_type: "code",
    },
    username_attributes: ["email"],
    user_verification_types: ["email"],
  },
  data: {
    aws_region: this.region,
    url: api.url,
    default_authorization_type: "AMAZON_COGNITO_USER_POOLS",
  },
  custom: {
    API: {
      [api.restApiName]: {
        endpoint: api.url,
        region: this.region,
        apiName: api.restApiName,
      },
    },
  },
};

// Deploy config JSON to S3 alongside frontend assets
new s3deploy.BucketDeployment(this, 'DeployAmplifyConfig', {
  sources: [s3deploy.Source.jsonData('amplify_outputs.json', amplifyOutputsJson)],
  destinationBucket: websiteBucket,
  distribution,
  distributionPaths: ['/amplify_outputs.json'],
});

// Also output as CfnOutput for manual inspection
new cdk.CfnOutput(this, 'AmplifyOutputsJson', {
  value: JSON.stringify(amplifyOutputsJson, null, 2),
  description: 'Complete amplify_outputs.json configuration',
});
```

### Frontend: Runtime Configuration Loader

The frontend fetches `/amplify_outputs.json` at startup and transforms it for the client library:

```typescript
// src/aws-config.ts
let amplifyConfig: any = null;

async function loadAmplifyConfig() {
  if (amplifyConfig) return amplifyConfig;

  try {
    const response = await fetch('/amplify_outputs.json');
    if (!response.ok) {
      throw new Error(`Failed to load configuration: ${response.status}`);
    }
    const config = await response.json();

    // Transform to Amplify Gen 2 format
    amplifyConfig = {
      Auth: {
        Cognito: {
          userPoolId: config.auth?.user_pool_id || '',
          userPoolClientId: config.auth?.user_pool_client_id || '',
          loginWith: {
            oauth: config.auth?.oauth ? {
              domain: config.auth.oauth.domain,
              scopes: config.auth.oauth.scopes,
              redirectSignIn: config.auth.oauth.redirect_sign_in_uri,
              redirectSignOut: config.auth.oauth.redirect_sign_out_uri,
              responseType: config.auth.oauth.response_type,
              providers: config.auth.oauth.identity_providers,
            } : undefined,
            email: config.auth?.username_attributes?.includes('email'),
          },
          passwordFormat: {
            minLength: config.auth?.password_policy?.min_length || 8,
            requireLowercase: config.auth?.password_policy?.require_lowercase,
            requireUppercase: config.auth?.password_policy?.require_uppercase,
            requireNumbers: config.auth?.password_policy?.require_numbers,
            requireSpecialCharacters: config.auth?.password_policy?.require_symbols,
          },
        },
      },
      API: {
        REST: Object.fromEntries(
          Object.entries(config.custom?.API || {}).map(([name, apiConfig]: [string, any]) => [
            name,
            { endpoint: apiConfig.endpoint, region: apiConfig.region },
          ])
        ),
      },
    };

    return amplifyConfig;
  } catch (error) {
    console.error('Failed to load AWS configuration:', error);
    // Return safe defaults for development/offline
    return { Auth: { Cognito: { userPoolId: '', userPoolClientId: '' } }, API: { REST: {} } };
  }
}

export function isDevelopmentMode(config?: any): boolean {
  return !config?.Auth?.Cognito?.userPoolId || !config?.Auth?.Cognito?.userPoolClientId;
}

export { loadAmplifyConfig };
```

### Configuration Flow Rules

1. **NEVER use build-time environment variables** (`VITE_API_URL`, etc.) for backend endpoints — use runtime config
2. **The same frontend bundle works in all environments** — only the `amplify_outputs.json` differs
3. **CDK deploys the config file to S3** using `s3deploy.BucketDeployment` with `Source.jsonData()`
4. **Invalidate CloudFront for the config path** after deployment: `distributionPaths: ['/amplify_outputs.json']`
5. **Frontend fetches config once at startup** and caches in memory
6. **Include a fallback** for local development when config is unavailable
7. **OAuth callback URLs MUST include the CloudFront URL** — the CDK stack dynamically adds it

---

## Multi-Environment CDK Configuration (REQUIRED)

### Stack Architecture

PWA projects use a **two-tier stack architecture** separating shared (persistent) resources from per-environment (ephemeral) resources:

```
┌─────────────────────────────────────────────────┐
│               Shared Stacks                      │
│  (NEVER destroyed by automation)                │
│                                                  │
│  ┌─────────────┐  ┌──────────────────────────┐ │
│  │  AuthStack  │  │  SharedInfraStack (opt.)  │ │
│  │  (Cognito)  │  │  (DynamoDB if shared)     │ │
│  └─────────────┘  └──────────────────────────┘ │
└─────────────────────────────────────────────────┘
         │                      │
         ▼                      ▼
┌─────────────────────────────────────────────────┐
│             App Stack (per environment)          │
│  (created/destroyed per stage)                  │
│                                                  │
│  ┌──────────┐ ┌────────┐ ┌────────┐ ┌───────┐ │
│  │ S3+CF    │ │ API GW │ │ Lambda │ │DynamoDB│ │
│  │ hosting  │ │        │ │        │ │(if    │ │
│  └──────────┘ └────────┘ └────────┘ │per-env)│ │
│                                      └───────┘ │
└─────────────────────────────────────────────────┘
```

### CDK App Entry Point Pattern

```typescript
// infrastructure/bin/infrastructure.ts
#!/usr/bin/env node
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import * as cloudfront from 'aws-cdk-lib/aws-cloudfront';
import { AppStack } from '../lib/app-stack';
import { AuthStack } from '../lib/auth-stack';

const app = new cdk.App();
const stage = app.node.tryGetContext('stage') || 'dev';
const environments = app.node.tryGetContext('environments');

// Handle PR environments with wildcard matching
let envConfig = environments?.[stage];
if (!envConfig && stage.startsWith('pr-')) {
  envConfig = environments?.['pr-*'];
}

if (!envConfig) {
  throw new Error(`Environment configuration not found for stage: ${stage}`);
}

const env = {
  account: process.env.CDK_DEFAULT_ACCOUNT || envConfig.account,
  region: process.env.CDK_DEFAULT_REGION || envConfig.region,
};

// Select cache policy based on environment
const cachePolicy = envConfig.cachePolicy === 'CACHING_DISABLED'
  ? cloudfront.CachePolicy.CACHING_DISABLED
  : cloudfront.CachePolicy.CACHING_OPTIMIZED;

// Build CORS configuration
const corsConfig = {
  allowCredentials: false,
  allowedOrigins: envConfig.allowedOrigins,
  allowedMethods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Api-Key'],
  maxAge: 86400,
};

// Shared auth stack — ephemeral environments share the same auth
const auth = new AuthStack(app, 'AppAuth', { env });

// App stack — suffixed for ephemeral environments
const isEphemeral = stage.startsWith('pr-');
const stackName = isEphemeral ? `AppStack-${stage}` : 'AppStack';

new AppStack(app, stackName, {
  env,
  cachePolicy,
  cacheMaxAge: envConfig.cacheMaxAge,
  corsConfig,
  oauthCallbackUrls: envConfig.oauthCallbackUrls || [],
  stage,
  auth,
});
```

### `cdk.json` Environment Configuration

```json
{
  "app": "npx ts-node --prefer-ts-exts bin/infrastructure.ts",
  "context": {
    "environments": {
      "dev": {
        "region": "eu-central-1",
        "cachePolicy": "CACHING_DISABLED",
        "cacheMaxAge": 120,
        "allowedOrigins": [
          "http://localhost:3000",
          "http://localhost:5173"
        ],
        "oauthCallbackUrls": [
          "http://localhost:3000/"
        ]
      },
      "prod": {
        "region": "eu-central-1",
        "cachePolicy": "CACHING_OPTIMIZED",
        "cacheMaxAge": 120,
        "allowedOrigins": [],
        "oauthCallbackUrls": []
      },
      "pr-*": {
        "region": "eu-central-1",
        "cachePolicy": "CACHING_DISABLED",
        "cacheMaxAge": 120,
        "allowedOrigins": [],
        "oauthCallbackUrls": []
      }
    }
  }
}
```

### App Stack Props Interface

```typescript
interface AppStackProps extends cdk.StackProps {
  /** CloudFront cache policy (DISABLED for dev/PR, OPTIMIZED for prod) */
  cachePolicy: cloudfront.ICachePolicy;
  /** Cache-Control max-age in seconds for response headers */
  cacheMaxAge?: number;
  /** CORS configuration for API Gateway and Lambda */
  corsConfig: {
    allowedOrigins: string[];
    allowCredentials: boolean;
    allowedMethods?: string[];
    allowedHeaders?: string[];
    maxAge?: number;
  };
  /** Additional OAuth callback URLs (e.g., localhost for dev) */
  oauthCallbackUrls: string[];
  /** Deployment stage name */
  stage: string;
  /** Reference to shared auth stack */
  auth: AuthStack;
}
```

### Dynamic CORS and OAuth from CloudFront URL

The app stack MUST dynamically add the CloudFront distribution URL to CORS origins and OAuth callback URLs, since the URL is only known at deploy time:

```typescript
// After CloudFront distribution is created:
const cloudfrontUrl = `https://${distribution.distributionDomainName}/`;

// Add to CORS config for Lambda/API Gateway
corsConfig.allowedOrigins = [...corsConfig.allowedOrigins, `https://${distribution.distributionDomainName}`];

// Add to OAuth callback URLs for Cognito UserPoolClient
const oauthUrls = [...props.oauthCallbackUrls, cloudfrontUrl];
```

### Multi-Environment Rules

1. **All environment config lives in `cdk.json` context** — never in code or env vars
2. **Use `--context stage=<name>`** to select environment at deploy time
3. **PR environments use wildcard matching** (`pr-*`) for configuration
4. **Shared stacks (auth, database) use `RemovalPolicy.RETAIN`** and are never destroyed
5. **App stacks use `RemovalPolicy.DESTROY`** in non-production environments
6. **CORS origins are empty for prod/PR** — the CloudFront URL is added dynamically by CDK
7. **The CloudFront URL is always added to OAuth callbacks** at deploy time — not hardcoded

---

## Deployment Pipeline (REQUIRED)

### Deployment Flow

Every PWA follows this deployment sequence:

```
1. Deploy CDK infrastructure (creates/updates S3, CloudFront, API, Lambda)
   └── CDK automatically deploys amplify_outputs.json to S3
2. Build frontend (npm run build)
3. Sync frontend dist/ to S3
4. Invalidate CloudFront cache
```

### Manual Deploy Script (`deploy.sh`)

Every project MUST include a `deploy.sh` for manual deployments:

```bash
#!/bin/bash
set -e

STACK_NAME="${STACK_NAME:-AppStack}"

# Deploy infrastructure
deploy_infrastructure() {
  echo "Deploying AWS infrastructure..."
  cd infrastructure
  npm ci
  npx cdk deploy -c stage=${STAGE:-dev} --require-approval never --all
  cd ..
}

# Deploy frontend
deploy_frontend() {
  echo "Building and deploying frontend..."
  cd frontend
  npm ci
  npm run build

  # Get S3 bucket name from CloudFormation outputs
  BUCKET_NAME=$(aws cloudformation describe-stacks \
    --stack-name $STACK_NAME \
    --query "Stacks[0].Outputs[?OutputKey=='S3BucketName'].OutputValue" \
    --output text)

  # Sync built assets to S3
  aws s3 sync dist/ s3://$BUCKET_NAME/ --delete

  # Invalidate CloudFront cache
  DISTRIBUTION_ID=$(aws cloudformation describe-stacks \
    --stack-name $STACK_NAME \
    --query "Stacks[0].Outputs[?OutputKey=='CloudFrontDistributionId'].OutputValue" \
    --output text)

  aws cloudfront create-invalidation \
    --distribution-id $DISTRIBUTION_ID \
    --paths "/*"

  cd ..
}

# Main
deploy_infrastructure
deploy_frontend

# Print URLs
WEBSITE_URL=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='WebsiteUrl'].OutputValue" \
  --output text)
echo "Website: $WEBSITE_URL"
```

### CI/CD Production Workflow (GitHub Actions)

```yaml
name: Deploy Production

on:
  push:
    branches: [main]

concurrency:
  group: deploy-production
  cancel-in-progress: false

env:
  NODE_VERSION: '20'
  AWS_REGION: 'eu-central-1'

jobs:
  test:
    uses: ./.github/workflows/test.yml

  deploy:
    runs-on: ubuntu-latest
    needs: test
    environment: production
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: infrastructure/package-lock.json

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Deploy infrastructure
        run: |
          cd infrastructure
          npm ci
          npx cdk deploy -c stage=prod --require-approval never --all
        env:
          CDK_DEFAULT_ACCOUNT: ${{ vars.AWS_ACCOUNT_ID }}
          CDK_DEFAULT_REGION: ${{ env.AWS_REGION }}

      - name: Deploy frontend
        run: |
          cd frontend
          npm ci
          npm run build

          BUCKET_NAME=$(aws cloudformation describe-stacks \
            --stack-name AppStack \
            --query "Stacks[0].Outputs[?OutputKey=='S3BucketName'].OutputValue" \
            --output text)

          aws s3 sync dist/ s3://$BUCKET_NAME/ --delete

          DISTRIBUTION_ID=$(aws cloudformation describe-stacks \
            --stack-name AppStack \
            --query "Stacks[0].Outputs[?OutputKey=='CloudFrontDistributionId'].OutputValue" \
            --output text)

          aws cloudfront create-invalidation \
            --distribution-id $DISTRIBUTION_ID \
            --paths "/*"
```

### Deployment Rules

1. **Infrastructure deploys first** — frontend depends on the S3 bucket existing
2. **CDK deploys `amplify_outputs.json` automatically** — never manually copy config files
3. **Always invalidate CloudFront** after S3 sync (entire distribution: `/*`)
4. **Use `--delete` on `s3 sync`** to remove stale files from previous builds
5. **Use `--require-approval never`** in CI/CD — approvals happen at the PR level
6. **AWS credentials via OIDC** — never use long-lived access keys
7. **Stack name is parameterized** — ephemeral environments use `AppStack-pr-<number>`
8. **The deploy script supports partial deployment** (`deploy.sh infrastructure` or `deploy.sh frontend`)
9. **Print the website URL at the end** of deployment for quick access

---

## Service Worker & Caching

### Setup

- Use `vite-plugin-pwa` or Workbox for service worker generation
- Register the service worker in the app entry point
- Handle service worker updates gracefully (prompt user to reload)

### Caching Strategies

| Resource Type | Strategy | Rationale |
|--------------|----------|-----------|
| App shell (HTML, CSS, JS) | Cache-first | Fast loads, update in background |
| API responses | Network-first | Fresh data preferred, cache as fallback |
| Static assets (images, fonts) | Cache-first | Rarely change, fast loads |
| User-generated content | Network-only | Always needs fresh data |

### Cache Management

- Version the cache and clear stale caches on service worker update
- Set maximum cache sizes to prevent storage bloat
- Implement cache expiration for API responses (e.g., 24 hours)
- Use `skipWaiting()` and `clientsClaim()` for immediate activation when appropriate

---

## Web App Manifest

### Required Fields

```json
{
  "name": "App Full Name",
  "short_name": "AppName",
  "start_url": "/",
  "display": "standalone",
  "orientation": "portrait-primary",
  "theme_color": "#<primary-color>",
  "background_color": "#ffffff",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icon-maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

### Manifest Rules

- Include both regular and maskable icons
- `theme_color` must match the app's primary brand color
- `background_color` must match the app's splash screen background
- `start_url` should include a source parameter for analytics (e.g., `/?source=pwa`)
- Test installability with Lighthouse PWA audit

---

## Mobile-First & Responsive Design

### Breakpoint Strategy

Design for mobile first, then enhance for larger screens:

```
base (mobile, <640px) -> sm: (640px) -> md: (768px) -> lg: (1024px) -> xl: (1280px)
```

### Touch Targets

- **Minimum size**: 44x44px for all interactive elements
- **Spacing**: minimum 8px between adjacent touch targets
- **Implementation**:
  ```css
  @layer utilities {
    .touch-target {
      min-height: 44px;
      min-width: 44px;
    }
  }

  @media (max-width: 768px) {
    button { min-height: 44px; }
  }
  ```

### iOS Safe Area Handling

Required for standalone mode on notched devices:

```css
@supports (padding-bottom: env(safe-area-inset-bottom)) {
  body {
    padding-bottom: env(safe-area-inset-bottom, 0px);
  }
}
```

### Input Zoom Prevention

Prevent iOS zoom on input focus by ensuring minimum 16px font size:

```css
@media (max-width: 768px) {
  input[type="text"],
  input[type="email"],
  input[type="password"],
  input[type="date"],
  input[type="time"],
  textarea,
  select {
    font-size: 16px;
  }
}
```

### Responsive Patterns

- Use responsive padding: `px-4 sm:px-6 lg:px-8`
- Stack elements vertically on mobile, horizontally on desktop: `flex flex-col sm:flex-row`
- Hide non-essential columns in tables on mobile: `hidden sm:table-cell`
- Use full-width modals on mobile, centered on desktop
- Implement collapsible navigation (hamburger menu on mobile)

---

## Offline-First Architecture

### Principles

1. **Optimistic UI**: Update the interface immediately, sync in the background
2. **Queue and retry**: Failed network requests are queued and retried when online
3. **Graceful degradation**: App remains functional without network
4. **Transparent status**: User always knows their connectivity state

### Implementation Pattern

```typescript
async function createRecord(record: Record): Promise<void> {
  // 1. Update UI immediately (optimistic)
  addRecordToCache(record);

  try {
    // 2. Sync to server
    await apiClient.post('/records', record);
  } catch (error) {
    // 3. Rollback on failure
    removeRecordFromCache(record.id);
    showNotification('Failed to save. Will retry when online.');
    // 4. Queue for retry
    queueForSync(record);
  }
}
```

### Offline UI Indicators

- Show clear online/offline status (banner or icon)
- Indicate when content is stale (served from cache)
- Disable or queue actions that require network
- Show sync progress when reconnecting
- Use subtle visual cues (e.g., greyed out timestamp) for cached data

### Data Persistence

- Use LocalStorage or IndexedDB for offline data
- Implement conflict resolution for concurrent edits
- Timestamp all cached data for staleness detection
- Clear expired cache entries on app startup

---

## Performance Requirements

### Core Web Vitals Targets

| Metric | Target | How |
|--------|--------|-----|
| LCP (Largest Contentful Paint) | < 2.5s | Cache-first for app shell |
| FID (First Input Delay) | < 100ms | Minimal main thread work |
| CLS (Cumulative Layout Shift) | < 0.1 | Reserve space for dynamic content |

### Optimization Techniques

- Route-level code splitting: `const Page = lazy(() => import('./pages/Page'))`
- Preload critical resources with `<link rel="preload">`
- Debounce user input (search, autocomplete): 300ms minimum
- Virtualize long lists (react-window or similar)
- Compress images and use modern formats (WebP, AVIF)

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
| Duplicated header/footer across page types | Use shared layout wrapper |
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

- [ ] Page uses the standard layout wrapper
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
- [ ] Lighthouse PWA score >= 90
- [ ] App works in standalone mode (no browser chrome)
- [ ] Offline mode shows appropriate cached content

---

## Checklist for New PWA Projects

Infrastructure setup (must be complete before first deployment):

- [ ] CloudFront + S3 hosting defined in CDK (`infrastructure/lib/app-stack.ts`)
- [ ] SPA routing CloudFront Function included
- [ ] OAC configured (not OAI) via `S3BucketOrigin.withOriginAccessControl()`
- [ ] Stack outputs exported: `WebsiteUrl`, `S3BucketName`, `CloudFrontDistributionId`
- [ ] `amplify_outputs.json` generated and deployed to S3 via `s3deploy.BucketDeployment`
- [ ] Frontend loads config at runtime via `fetch('/amplify_outputs.json')`  — no build-time env vars
- [ ] `cdk.json` has environment config for `dev`, `prod`, and `pr-*`
- [ ] Cache policy: `CACHING_DISABLED` for dev/PR, `CACHING_OPTIMIZED` for prod
- [ ] CloudFront URL dynamically added to CORS origins and OAuth callback URLs
- [ ] Shared stacks (auth, database) use `RemovalPolicy.RETAIN`
- [ ] Ephemeral stacks use `RemovalPolicy.DESTROY` + `autoDeleteObjects: true`
- [ ] `deploy.sh` script present with infrastructure + frontend deployment
- [ ] GitHub Actions workflow: test → deploy infra → build frontend → S3 sync → CloudFront invalidation
- [ ] AWS credentials via OIDC (no long-lived access keys)
