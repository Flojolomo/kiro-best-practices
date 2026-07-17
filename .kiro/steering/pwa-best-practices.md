---
title: Progressive Web App Best Practices
inclusion: always
---

# Progressive Web App (PWA) Best Practices

## Overview

Progressive Web Apps must deliver a native-like experience with consistent design across all devices and pages. This steering file defines standards for uniform design, DRY component architecture, service workers, caching, manifests, mobile-first design, and offline-first architecture.

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
