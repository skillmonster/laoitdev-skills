# Core Contexts

## `core/context/ThemeContext.tsx`

Wraps MUI `ThemeProvider`, `CssBaseline`, and `LocalizationProvider`. Persists mode to localStorage.

```tsx
import { createContext, useContext, useMemo, useState, useEffect, type ReactNode } from 'react'
import { ThemeProvider as MuiThemeProvider } from '@mui/material/styles'
import CssBaseline from '@mui/material/CssBaseline'
import { LocalizationProvider } from '@mui/x-date-pickers/LocalizationProvider'
import { AdapterDayjs } from '@mui/x-date-pickers/AdapterDayjs'
import { createAppTheme } from '@/core/config/theme.config'

type ThemeMode = 'light' | 'dark'
const STORAGE_KEY = 'theme-mode'

interface ThemeContextType { mode: ThemeMode; toggleTheme: () => void }
const ThemeContext = createContext<ThemeContextType | undefined>(undefined)

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [mode, setMode] = useState<ThemeMode>(
    () => (localStorage.getItem(STORAGE_KEY) as ThemeMode) || 'light'
  )

  useEffect(() => { localStorage.setItem(STORAGE_KEY, mode) }, [mode])

  const toggleTheme = () => setMode((m) => (m === 'light' ? 'dark' : 'light'))
  const theme = useMemo(() => createAppTheme(mode), [mode])

  return (
    <ThemeContext.Provider value={{ mode, toggleTheme }}>
      <MuiThemeProvider theme={theme}>
        <CssBaseline />
        <LocalizationProvider dateAdapter={AdapterDayjs}>
          {children}
        </LocalizationProvider>
      </MuiThemeProvider>
    </ThemeContext.Provider>
  )
}

export function useThemeMode() {
  const ctx = useContext(ThemeContext)
  if (!ctx) throw new Error('useThemeMode must be used inside ThemeProvider')
  return ctx
}
```

`createAppTheme(mode)` lives in `core/config/theme.config.ts` — it's a factory that calls `createTheme()` with palette, typography, shape, and component overrides.

---

## `core/context/AuthContext.tsx`

Holds reactive user state for components (navbar, permission UI). The router's `beforeLoad` reads from storage directly — `AuthContext` is for the UI layer.

```tsx
import { createContext, useContext, useState, useEffect, type ReactNode } from 'react'
import { storage, STORAGE_KEYS } from '@/core/utils/storage'
import type { User } from '@/features/auth/types'

interface AuthContextType {
  user: User | null
  isAuthenticated: boolean
  isLoading: boolean
  login: (token: string, user: User) => void
  logout: () => void
}

const AuthContext = createContext<AuthContextType | undefined>(undefined)

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null)
  const [isLoading, setIsLoading] = useState(true)

  useEffect(() => {
    const token = storage.get<string>(STORAGE_KEYS.ACCESS_TOKEN)
    if (!token) { setIsLoading(false); return }

    authService.getProfile()
      .then((profile) => setUser(profile.user))
      .catch(() => storage.remove(STORAGE_KEYS.ACCESS_TOKEN))
      .finally(() => setIsLoading(false))
  }, [])

  const login = (token: string, userData: User) => {
    storage.set(STORAGE_KEYS.ACCESS_TOKEN, token)
    setUser(userData)
  }

  const logout = () => {
    storage.remove(STORAGE_KEYS.ACCESS_TOKEN)
    setUser(null)
  }

  return (
    <AuthContext.Provider value={{ user, isAuthenticated: !!user, isLoading, login, logout }}>
      {children}
    </AuthContext.Provider>
  )
}

export function useAuth() {
  const ctx = useContext(AuthContext)
  if (!ctx) throw new Error('useAuth must be used inside AuthProvider')
  return ctx
}
```

---

## `core/context/SnackbarContext.tsx`

Global toast notifications. Accepts a plain string or `{ title, description? }`.

```tsx
import { createContext, useContext, useState, useCallback, type ReactNode } from 'react'
import { Snackbar, Alert, AlertTitle, type AlertColor } from '@mui/material'

type SnackbarInput = string | { title: string; description?: string }

interface SnackbarContextType {
  showSuccess: (msg: SnackbarInput) => void
  showError:   (msg: SnackbarInput) => void
  showWarning: (msg: SnackbarInput) => void
  showInfo:    (msg: SnackbarInput) => void
}

const SnackbarContext = createContext<SnackbarContextType | undefined>(undefined)

export function SnackbarProvider({ children }: { children: ReactNode }) {
  const [snackbar, setSnackbar] = useState<{
    title: string; description?: string; severity: AlertColor; key: number
  } | null>(null)

  const show = useCallback((msg: SnackbarInput, severity: AlertColor) => {
    const parsed = typeof msg === 'string' ? { title: msg } : msg
    setSnackbar({ ...parsed, severity, key: Date.now() })
  }, [])

  const showSuccess = useCallback((msg: SnackbarInput) => show(msg, 'success'), [show])
  const showError   = useCallback((msg: SnackbarInput) => show(msg, 'error'),   [show])
  const showWarning = useCallback((msg: SnackbarInput) => show(msg, 'warning'), [show])
  const showInfo    = useCallback((msg: SnackbarInput) => show(msg, 'info'),    [show])

  return (
    <SnackbarContext.Provider value={{ showSuccess, showError, showWarning, showInfo }}>
      {children}
      <Snackbar
        key={snackbar?.key}
        open={!!snackbar}
        autoHideDuration={6000}
        onClose={(_, reason) => reason !== 'clickaway' && setSnackbar(null)}
        anchorOrigin={{ vertical: 'top', horizontal: 'right' }}
      >
        <Alert severity={snackbar?.severity} variant="filled" onClose={() => setSnackbar(null)} sx={{ minWidth: 300 }}>
          {snackbar?.description ? (
            <><AlertTitle>{snackbar.title}</AlertTitle>{snackbar.description}</>
          ) : snackbar?.title}
        </Alert>
      </Snackbar>
    </SnackbarContext.Provider>
  )
}

export function useSnackbar() {
  const ctx = useContext(SnackbarContext)
  if (!ctx) throw new Error('useSnackbar must be used inside SnackbarProvider')
  return ctx
}
```

Usage:
```ts
const { showSuccess, showError } = useSnackbar()
showSuccess('Saved!')
showError({ title: 'Upload failed', description: 'File exceeds 10 MB.' })
```
