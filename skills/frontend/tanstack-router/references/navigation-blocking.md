# Navigation Blocking

Block navigation away from a page when there is unsaved state (e.g. form submitting, dirty form). Uses `useBlocker` from `@tanstack/react-router`.

## Basic — block while mutation is pending

```tsx
import { useBlocker } from '@tanstack/react-router'

useBlocker({ shouldBlockFn: () => mutation.isPending })
```

`shouldBlockFn` is called on every attempted navigation. Return `true` to block, `false` to allow. Can be `async`. When no `withResolver`, the browser's native `confirm` dialog is shown automatically.

## Custom dialog with `withResolver`

Set `withResolver: true` to receive `proceed` / `reset` for a custom MUI dialog:

```tsx
import { useBlocker } from '@tanstack/react-router'
import { useStore } from '@tanstack/react-form'

export function FeatureForm() {
  const mutation = useUpdateFeature()
  const isDirty = useStore(form.store, (s) => s.isDirty)

  const { status, proceed, reset } = useBlocker({
    shouldBlockFn: () => isDirty || mutation.isPending,
    withResolver: true,
  })

  return (
    <>
      {/* form fields */}

      <Dialog open={status === 'blocked'} onClose={reset}>
        <DialogTitle>{t('common.unsavedChanges')}</DialogTitle>
        <DialogActions>
          <Button onClick={reset}>{t('common.stayOnPage')}</Button>
          <Button onClick={proceed} color="error">{t('common.leavePage')}</Button>
        </DialogActions>
      </Dialog>
    </>
  )
}
```

## `Block` component (declarative alternative)

```tsx
import { Block } from '@tanstack/react-router'

<Block shouldBlockFn={() => isDirty} withResolver>
  {({ status, proceed, reset }) =>
    status === 'blocked' ? (
      <Dialog open onClose={reset}>
        <DialogTitle>{t('common.unsavedChanges')}</DialogTitle>
        <DialogActions>
          <Button onClick={reset}>{t('common.cancel')}</Button>
          <Button onClick={proceed} color="error">{t('common.leave')}</Button>
        </DialogActions>
      </Dialog>
    ) : null
  }
</Block>
```

## Options

| Option | Type | Default | Description |
|---|---|---|---|
| `shouldBlockFn` | `(args) => boolean \| Promise<boolean>` | — | Return `true` to block |
| `withResolver` | `boolean` | `false` | Enables `proceed` / `reset` for custom UI |
| `disabled` | `boolean` | `false` | Dynamically disable without unmounting |
| `enableBeforeUnload` | `boolean \| (() => boolean)` | `true` | Also shows browser "Leave site?" on hard nav |

## `shouldBlockFn` args

```ts
shouldBlockFn: ({ current, next, action }) => {
  // current / next: { routeId, fullPath, pathname, params, search }
  // action: 'PUSH' | 'POP' | 'REPLACE'

  // Block only when navigating to a different page, not just changing search params
  return current.pathname !== next.pathname && isDirty
}
```
