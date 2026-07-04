---
name: tanstack-router
description: "How to use TanStack Router for file-based routing. Use this whenever creating new route files, adding search params validation, implementing route guards, using loaders, or navigating programmatically. Covers createFileRoute, validateSearch with Zod, beforeLoad guards, loader prefetch, useSearch, useParams, useNavigate, and Link."
---

# Routing with TanStack Router

## Core Invariants (always enforced — never violate)

- Never manually edit `src/routeTree.gen.ts` — it is auto-generated; commit it, don't hand-edit it.
- **Routes are global** — all route files live in `src/routes/`. Never create route files inside feature folders.
- **Search schemas never inline in route files** — always import them from `features/[feature]/schemas/[feature].schema.ts`.
- Inside a route component, always use `Route.useSearch()`, `Route.useNavigate()`, and `Route.useParams()` — never the bare hooks without `from`.
- Every `beforeLoad` on a layout route must return `{ breadcrumb: '...' }` (except redirect-only routes that only `throw redirect`).

## References

Read the relevant file when you need it:
- `references/authenticated-routes.md` — full auth guard setup, RouterContext, `beforeLoad` redirect, per-route permission guard
- `references/navigation-blocking.md` — `useBlocker`, custom dialog with `withResolver`, `Block` component, options table

---

## Route Tree Generation

`src/routeTree.gen.ts` is auto-generated — never edit it manually. Commit it to git so TypeScript can type-check in CI without running the dev server.

### Vite plugin (recommended)

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import { tanstackRouter } from '@tanstack/router-plugin/vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [
    tanstackRouter({ target: 'react', autoCodeSplitting: true }), // must come before react()
    react(),
  ],
})
```

> `TanStackRouterVite` is a deprecated alias — prefer `tanstackRouter`.

### CLI (CI / without dev server)

```bash
bun add -d @tanstack/router-cli
bunx tsr generate   # one-shot before build/type-check
bunx tsr watch      # continuous watch
```

### Plugin config options

```ts
tanstackRouter({
  target: 'react',
  routesDirectory: './src/routes',
  generatedRouteTree: './src/routeTree.gen.ts',
  autoCodeSplitting: true,
  routeFileIgnorePrefix: '-',
  quoteStyle: 'single',
  semicolons: false,
})
```

---

## Route File Naming

```
src/routes/
├── __root.tsx
├── _layout.tsx
└── _layout/
    └── your-section/
        └── your-feature/
            ├── index.tsx          # /your-section/your-feature/
            ├── create.tsx         # /your-section/your-feature/create
            └── $featureId/
                ├── index.tsx      # /your-section/your-feature/$featureId
                └── edit.tsx       # /your-section/your-feature/$featureId/edit
```

`_` prefix = pathless layout (no URL segment). `$param` = dynamic segment. Always use nested directories — never dot notation (flat routes).

---

## Search Schema Pattern

Search schemas belong in the feature's schema file — **never inline in a route file**.

```ts
// features/your-feature/schemas/your-feature.schema.ts
import { fallback } from '@tanstack/zod-adapter'

export const yourFeatureSearchSchema = z.object({
  page: fallback(z.number(), 1).default(1),
  page_size: fallback(z.number(), 20).default(20),
  search: z.string().optional(),
  status: z.string().optional(),
  sort_by: z.string().optional(),
  sort_order: z.string().optional(),
})

export type YourFeatureSearch = z.infer<typeof yourFeatureSearchSchema>
```

Use `fallback(z.number(), defaultValue).default(defaultValue)` for fields that need a fallback when the URL param is missing or invalid. Plain `.optional()` is fine for truly optional filters.

Wire up in the route file by importing and passing directly:

```ts
import { zodValidator } from '@tanstack/zod-adapter'
import { yourFeatureSearchSchema } from '@/features/your-feature/schemas/your-feature.schema'

validateSearch: zodValidator(yourFeatureSearchSchema)
```

---

## Creating Route Files

### List Page

```tsx
// src/routes/_layout/your-section/your-feature/index.tsx
import { createFileRoute } from '@tanstack/react-router'
import { zodValidator } from '@tanstack/zod-adapter'
import { yourFeatureSearchSchema } from '@/features/your-feature/schemas/your-feature.schema'
import { guardRoute } from '@/shared/utils/route-guard.util'

export const Route = createFileRoute('/_layout/your-section/your-feature/')({
  validateSearch: zodValidator(yourFeatureSearchSchema),
  beforeLoad: ({ context }) => {
    guardRoute(context.user, ['feature-index'])
    return { breadcrumb: 'breadcrumbs.features' }
  },
  component: FeaturePage,
})

function FeaturePage() {
  const { t } = useTranslation()
  const searchParams = Route.useSearch()
  const navigate = Route.useNavigate()

  const handlePageChange = (page: number) => {
    navigate({ search: (prev) => ({ ...prev, page }) })
  }

  const handlePageSizeChange = (pageSize: number) => {
    navigate({ search: (prev) => ({ ...prev, page: 1, page_size: pageSize }) })
  }

  return (
    <Container maxWidth="xl" sx={{ p: 0 }}>
      {/* header, filters, list */}
    </Container>
  )
}
```

### Create Page

```tsx
// src/routes/_layout/your-section/your-feature/create.tsx
import { createFileRoute } from '@tanstack/react-router'
import { FeatureForm } from '@/features/your-feature/components/FeatureForm'
import { guardRoute } from '@/shared/utils/route-guard.util'

export const Route = createFileRoute('/_layout/your-section/your-feature/create')({
  beforeLoad: ({ context }) => {
    guardRoute(context.user, ['feature-add'])
    return { breadcrumb: 'breadcrumbs.createFeature' }
  },
  component: FeatureCreatePage,
})

function FeatureCreatePage() {
  return <FeatureForm />
}
```

### Detail Page (with loader prefetch)

```tsx
// src/routes/_layout/your-section/your-feature/$featureId/index.tsx
import { createFileRoute } from '@tanstack/react-router'
import { FeatureDetail } from '@/features/your-feature/components/FeatureDetail'
import { YOUR_FEATURE_QUERY_KEYS } from '@/features/your-feature/constants/query-keys'
import { yourFeatureService } from '@/features/your-feature/services/your-feature.service'
import { guardRoute } from '@/shared/utils/route-guard.util'

export const Route = createFileRoute('/_layout/your-section/your-feature/$featureId/')({
  beforeLoad: ({ context }) => {
    guardRoute(context.user, ['feature-index'])
    return { breadcrumb: 'breadcrumbs.featureDetail' }
  },
  loader: async ({ params, context }) => {
    const id = Number(params.featureId)
    await context.queryClient.ensureQueryData({
      queryKey: YOUR_FEATURE_QUERY_KEYS.DETAIL(id),
      queryFn: ({ signal }) => yourFeatureService.getById(id, signal),
    })
  },
  component: FeatureDetailPage,
})

function FeatureDetailPage() {
  const { featureId } = Route.useParams()
  return <FeatureDetail featureId={Number(featureId)} />
}
```

### Edit Page

```tsx
// src/routes/_layout/your-section/your-feature/$featureId/edit.tsx
import { createFileRoute } from '@tanstack/react-router'
import { FeatureEditForm } from '@/features/your-feature/components/FeatureEditForm'
import { YOUR_FEATURE_QUERY_KEYS } from '@/features/your-feature/constants/query-keys'
import { yourFeatureService } from '@/features/your-feature/services/your-feature.service'
import { guardRoute } from '@/shared/utils/route-guard.util'

export const Route = createFileRoute('/_layout/your-section/your-feature/$featureId/edit')({
  beforeLoad: ({ context }) => {
    guardRoute(context.user, ['feature-edit'])
    return { breadcrumb: 'breadcrumbs.editFeature' }
  },
  loader: async ({ params, context }) => {
    const id = Number(params.featureId)
    await context.queryClient.ensureQueryData({
      queryKey: YOUR_FEATURE_QUERY_KEYS.DETAIL(id),
      queryFn: ({ signal }) => yourFeatureService.getById(id, signal),
    })
  },
  component: FeatureEditPage,
})

function FeatureEditPage() {
  const { featureId } = Route.useParams()
  return <FeatureEditForm featureId={Number(featureId)} />
}
```

---

## Navigation Hooks

### Inside a route file — always use `Route.*` variants

```tsx
// search params
const searchParams = Route.useSearch()

// navigate
const navigate = Route.useNavigate()
navigate({ search: (prev) => ({ ...prev, page: newPage }) })
navigate({ to: '/section/feature/$featureId', params: { featureId: String(id) } })
navigate({ to: '/section/feature' })

// params
const { featureId } = Route.useParams()
```

Never use bare `useNavigate()`, `useSearch({ from })`, or `useParams({ from })` inside the route component function — those are for feature components outside the route file.

### Inside feature components (outside route file)

```tsx
// strict mode — preferred for single-route components
const searchParams = useSearch({ from: '/_layout/section/feature/' })
const { featureId } = useParams({ from: '/_layout/section/feature/$featureId/' })

// non-strict — use only when a component renders under multiple routes
const searchParams = useSearch({ strict: false })

// select — re-renders only when selection changes
const page = useSearch({ from: '/_layout/section/feature/', select: (s) => s.page })
```

### Link

```tsx
<Button component={Link} to="/section/feature/create" variant="contained">Create</Button>
<Link to="/section/feature/$featureId" params={{ featureId: '123' }}>View</Link>
```

---

## Router Config

```ts
// core/config/router.config.ts
import { createRouter } from '@tanstack/react-router'
import { routeTree } from '@/routeTree.gen'
import { queryClient } from './query-client.config'
import type { RouterContext } from '@/routes/__root'

export const router = createRouter({
  routeTree,
  defaultPreload: 'intent',
  scrollRestoration: true,
  context: {
    queryClient,
    user: undefined,
  } as RouterContext,
})

declare module '@tanstack/react-router' {
  interface Register { router: typeof router }
}
```

## Root Layout

`_layout.tsx` wraps all authenticated routes — Navbar, Sidebar, `<Outlet />`. Its `beforeLoad` handles auth redirects (see `references/authenticated-routes.md`).
