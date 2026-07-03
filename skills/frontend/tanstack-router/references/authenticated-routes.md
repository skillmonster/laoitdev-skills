# Authenticated Routes

Authentication is enforced in `beforeLoad` by throwing `redirect`. The entire protected area sits under a single pathless `_layout` route so every child inherits the guard automatically.

## Step 1 — Define router context

```ts
// src/routes/__root.tsx
import { createRootRouteWithContext, Outlet } from '@tanstack/react-router'
import type { QueryClient } from '@tanstack/react-query'
import type { User } from '@/features/auth/types'

export interface RouterContext {
  queryClient: QueryClient
  user?: User       // undefined until authenticated; populated by _layout.tsx beforeLoad
}

export const Route = createRootRouteWithContext<RouterContext>()({
  component: () => <Outlet />,
})
```

Wire context into the router:

```ts
// src/core/config/router.config.ts
export const router = createRouter({
  routeTree,
  context: { queryClient, user: undefined },
})
```

## Step 2 — Guard the layout route

`_layout.tsx` `beforeLoad` runs before every child route. Throw `redirect` to abort; return `{ user }` so children receive it via `context.user`.

```ts
// src/routes/_layout.tsx
import { createFileRoute, redirect, Outlet } from '@tanstack/react-router'
import { storage, STORAGE_KEYS } from '@/core/utils/storage'
import { authService } from '@/features/auth/services/auth.service'

export const Route = createFileRoute('/_layout')({
  beforeLoad: async ({ context }) => {
    const token = storage.get<string>(STORAGE_KEYS.ACCESS_TOKEN)

    if (!token) {
      throw redirect({ to: '/login' })
    }

    return context.queryClient
      .ensureQueryData({
        queryKey: ['auth', 'profile'],
        queryFn: () => authService.getProfile(),
      })
      .then((profile) => ({ user: profile.user }))
      .catch(() => {
        storage.remove(STORAGE_KEYS.ACCESS_TOKEN)
        throw redirect({ to: '/login' })
      })
  },
  component: LayoutComponent,
})
```

## Step 3 — Per-route permission guard

```ts
// src/shared/utils/route-guard.util.ts
import { redirect } from '@tanstack/react-router'

export function guardRoute(user: User | undefined, permissions: string[]): void {
  if (!canAccessRoute(user, permissions)) {
    throw redirect({ to: '/403' })
  }
}
```

Use in any child route's `beforeLoad`:

```ts
export const Route = createFileRoute('/_layout/section/feature/')({
  beforeLoad: ({ context }) => {
    guardRoute(context.user, ['feature-index'])
    return { breadcrumb: 'breadcrumbs.features' }
  },
})
```

## Step 4 — Redirect after login

```ts
loginMutation.mutate(credentials, {
  onSuccess: (response) => {
    storage.set(STORAGE_KEYS.ACCESS_TOKEN, response.data.access_token)
    navigate({ to: '/' })
  },
})
```

## `redirect` options

```ts
throw redirect({ to: '/login' })
throw redirect({ to: '/login', search: { from: location.pathname } })  // pass return URL
throw redirect({ to: '/403' })
throw redirect({ to: '..', replace: true })   // replace history entry
```

`redirect` can be thrown from both `beforeLoad` and `loader`.
