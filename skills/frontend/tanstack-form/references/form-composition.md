# Form Composition — Setup & Usage

TanStack Form v1 composition API splits form infrastructure into two layers: shared setup files in `src/shared/components/form/` and per-feature form usage. Set up the shared layer once; all features consume it.

---

## Folder Structure

```
src/shared/components/form/
├── createAppForm.tsx           ← Step 1: create contexts
├── hooks/
│   └── useAppForm.ts           ← Step 3: wire up global hook
├── components/
│   ├── AppTextField.tsx        ← Step 2: field components (useFieldContext)
│   ├── AppTextFieldNumber.tsx
│   ├── AppSelect.tsx
│   ├── AppAutocomplete.tsx
│   ├── AppDatePicker.tsx
│   ├── AppDateTimePicker.tsx
│   ├── AppCheckbox.tsx
│   ├── AppImageUploadField.tsx
│   ├── AppRichTextEditor.tsx
│   └── AppSubmitButton.tsx     ← Step 2b: form-level component (useFormContext)
└── index.ts                    ← Step 4: barrel exports
```

---

## Step 1 — Create Contexts (`createAppForm.tsx`)

```tsx
// src/shared/components/form/createAppForm.tsx
import { createFormHookContexts } from '@tanstack/react-form'

export const { fieldContext, useFieldContext, formContext, useFormContext } =
  createFormHookContexts()
```

`fieldContext` / `formContext` are passed into `createFormHook`.  
`useFieldContext` / `useFormContext` are used inside field/form components to read state.

---

## Step 2 — Create Field Components (`components/App*.tsx`)

Each field component calls `useFieldContext<T>()` to get the field API.

```tsx
// src/shared/components/form/components/AppTextField.tsx
import { useFieldContext } from '../createAppForm'

export function AppTextField({ label, required, ...props }: TextFieldProps & { required?: boolean }) {
  const field = useFieldContext<string>()

  return (
    <TextField
      label={label}
      value={field.state.value ?? ''}
      onChange={(e) => field.handleChange(e.target.value)}
      onBlur={field.handleBlur}
      error={field.state.meta.isTouched && !field.state.meta.isValid}
      helperText={
        field.state.meta.isTouched ? field.state.meta.errors[0]?.message : undefined
      }
      required={required}
      fullWidth
      {...props}
    />
  )
}
```

Form-level components (like a submit button) use `useFormContext()` instead:

```tsx
// src/shared/components/form/components/AppSubmitButton.tsx
import { useFormContext } from '../createAppForm'

export function AppSubmitButton({ label, loading }: { label: string; loading?: boolean }) {
  const form = useFormContext()
  return (
    <form.Subscribe selector={(s) => s.isSubmitting}>
      {(isSubmitting) => (
        <Button type="submit" loading={loading || isSubmitting} variant="contained">
          {label}
        </Button>
      )}
    </form.Subscribe>
  )
}
```

---

## Step 3 — Wire Up the Global Hook (`hooks/useAppForm.ts`)

Register all field and form components once. This exports `useAppForm` and `withForm`.

```tsx
// src/shared/components/form/hooks/useAppForm.ts
import { createFormHook } from '@tanstack/react-form'
import { fieldContext, formContext } from '../createAppForm'
import { AppTextField } from '../components/AppTextField'
import { AppTextFieldNumber } from '../components/AppTextFieldNumber'
import { AppSelect } from '../components/AppSelect'
import { AppAutocomplete } from '../components/AppAutocomplete'
import { AppDatePicker } from '../components/AppDatePicker'
import { AppDateTimePicker } from '../components/AppDateTimePicker'
import { AppCheckbox } from '../components/AppCheckbox'
import { AppImageUploadField } from '../components/AppImageUploadField'
import { AppRichTextEditor } from '../components/AppRichTextEditor'
import { AppSubmitButton } from '../components/AppSubmitButton'

export const { useAppForm, withForm } = createFormHook({
  fieldComponents: {
    TextField: AppTextField,
    TextFieldNumber: AppTextFieldNumber,
    Select: AppSelect,
    Autocomplete: AppAutocomplete,
    DatePicker: AppDatePicker,
    DateTimePicker: AppDateTimePicker,
    Checkbox: AppCheckbox,
    ImageUploadField: AppImageUploadField,
    RichTextEditor: AppRichTextEditor,
  },
  formComponents: {
    SubmitButton: AppSubmitButton,
  },
  fieldContext,
  formContext,
})
```

To add a new field component later, create `components/AppXxx.tsx` using `useFieldContext`, then add it to `fieldComponents` here and export it from `index.ts`.

---

## Step 4 — Barrel Exports (`index.ts`)

```ts
// src/shared/components/form/index.ts
export { fieldContext, formContext, useFieldContext, useFormContext } from './createAppForm'
export { useAppForm, withForm } from './hooks/useAppForm'
export { AppTextField } from './components/AppTextField'
export { AppTextFieldNumber } from './components/AppTextFieldNumber'
export { AppSelect } from './components/AppSelect'
export { AppAutocomplete } from './components/AppAutocomplete'
export { AppDatePicker } from './components/AppDatePicker'
export { AppDateTimePicker } from './components/AppDateTimePicker'
export { AppCheckbox } from './components/AppCheckbox'
export { AppImageUploadField } from './components/AppImageUploadField'
export { AppRichTextEditor } from './components/AppRichTextEditor'
export { AppSubmitButton } from './components/AppSubmitButton'
export { FormSchemaDebugger } from './components/FormSchemaDebugger'
```

All feature imports use the single alias:
```ts
import { useAppForm, withForm } from '@/shared/components/form'
```

---

## Usage — `withForm` (reusable section receiving the full form)

Defined **at module level** — not inside a component. The result IS the component.

```tsx
// contact-section.tsx
import { withForm } from '@/shared/components/form'

export const ContactSection = withForm({
  render: ({ form }) => (
    <form.AppForm>
      <Stack spacing={2}>
        <form.AppField name="email">
          {(field) => <field.TextField label={t('feature.email')} required />}
        </form.AppField>
        <form.AppField name="phone">
          {(field) => <field.TextField label={t('feature.phone')} />}
        </form.AppField>
      </Stack>
    </form.AppForm>
  ),
})

// In parent:
<ContactSection form={form} />
```

Pass extra props via `props`:
```tsx
export const ContactSection = withForm({
  props: {} as { readonly: boolean },
  render: ({ form, readonly }) => (
    <form.AppForm>
      <form.AppField name="email">
        {(field) => <field.TextField label="Email" disabled={readonly} />}
      </form.AppField>
    </form.AppForm>
  ),
})

<ContactSection form={form} readonly={!canEdit} />
```

---

## Usage — `useFormContext` (last resort — no prop drilling possible)

Use only when a child cannot receive `form` as a prop (e.g. TanStack Router `<Outlet />`). Always prefer `withForm` first.

```tsx
import { useAppForm, useFormContext } from '@/shared/components/form'

function ParentRoute() {
  const form = useAppForm({ defaultValues: { name: '' }, onSubmit: handleSubmit })
  return (
    <form.AppForm>
      <Outlet />
    </form.AppForm>
  )
}

function ChildRoute() {
  const form = useFormContext()
  return (
    <form.AppField name="name">
      {(field) => <field.TextField label="Name" />}
    </form.AppField>
  )
}
```

---

## When to use which

| Situation | API |
|---|---|
| Split one form into tab/accordion sections | `withForm` |
| Deep child needs form state, no prop possible | `useFormContext` |
| Build a new reusable field component | `useFieldContext<T>()` in `components/` |
| Build a new form-level component (buttons, etc.) | `useFormContext()` in `components/` |
