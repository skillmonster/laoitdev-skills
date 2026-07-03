# Storage Utility & Layout Store

## `core/utils/storage.ts`

Type-safe localStorage wrapper. All keys are centralised in `STORAGE_KEYS` — never use raw strings.

```ts
export const storage = {
  get<T>(key: string): T | null {
    try {
      const item = localStorage.getItem(key)
      return item ? (JSON.parse(item) as T) : null
    } catch { return null }
  },

  getOrDefault<T>(key: string, defaultValue: T): T {
    return this.get<T>(key) ?? defaultValue
  },

  set<T>(key: string, value: T): void {
    try { localStorage.setItem(key, JSON.stringify(value)) } catch { /* quota exceeded */ }
  },

  remove(key: string): void {
    try { localStorage.removeItem(key) } catch { /* noop */ }
  },

  has(key: string): boolean {
    return localStorage.getItem(key) !== null
  },
}

export const STORAGE_KEYS = {
  ACCESS_TOKEN: 'app-access-token',
  THEME_MODE:   'theme-mode',
} as const
```

Usage:
```ts
import { storage, STORAGE_KEYS } from '@/core/utils/storage'

storage.get<string>(STORAGE_KEYS.ACCESS_TOKEN)
storage.set(STORAGE_KEYS.ACCESS_TOKEN, token)
storage.remove(STORAGE_KEYS.ACCESS_TOKEN)
storage.getOrDefault(STORAGE_KEYS.THEME_MODE, 'light')
```

Add new keys to `STORAGE_KEYS` — never pass raw string literals to `storage.*`.

---

## `core/store/layout.store.ts`

Zustand store for UI layout state. `persist` middleware keeps state across page reloads.

```ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface LayoutStore {
  sidebarOpen: boolean
  toggleSidebar: () => void
  setSidebarOpen: (open: boolean) => void
}

export const useLayoutStore = create<LayoutStore>()(
  persist(
    (set) => ({
      sidebarOpen: true,
      toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
      setSidebarOpen: (open) => set({ sidebarOpen: open }),
    }),
    { name: 'layout-storage' }
  )
)
```

Usage:
```ts
const { sidebarOpen, toggleSidebar, setSidebarOpen } = useLayoutStore()
```
