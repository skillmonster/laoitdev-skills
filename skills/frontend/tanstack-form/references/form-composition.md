# Form Composition (Multi-Section / Nested Forms)

TanStack Form v1 has three complementary APIs for splitting large forms. All three are exported from `@/shared/components/form`.

## `withForm` — reusable section that receives the full form

Defined **at module level**, not inside a component. The result IS the component.

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

## `withFieldGroup` — reusable section that lenses into a nested object

Field paths inside the group are **relative to the nested type**, not the root form.

```tsx
// address-section.tsx
import { withFieldGroup } from '@/shared/components/form'

interface AddressData {
  street: string
  city: string
  country: string
}

export const AddressSection = withFieldGroup<AddressData>({
  render: ({ group }) => (
    <group.AppForm>
      <Stack spacing={2}>
        <group.AppField name="street">
          {(field) => <field.TextField label={t('common.street')} />}
        </group.AppField>
        <group.AppField name="city">
          {(field) => <field.TextField label={t('common.city')} />}
        </group.AppField>
        <group.AppField name="country">
          {(field) => <field.TextField label={t('common.country')} />}
        </group.AppField>
      </Stack>
    </group.AppForm>
  ),
})

// Parent form must have `homeAddress: AddressData`
<AddressSection form={form} fields="homeAddress" />
```

Use a `FieldsMap` when field names don't match 1:1:
```tsx
import type { FieldsMap } from '@tanstack/react-form'

<AddressSection
  form={form}
  fields={{ street: 'billingStreet', city: 'billingCity', country: 'billingCountry' } satisfies FieldsMap<ParentFormData, AddressData>}
/>
```

## `useTypedAppFormContext` — read form from context without prop drilling

The form is set in context by `form.AppForm` / `group.AppForm`. Options are **for type inference only — never executed at runtime**.

```tsx
// deep-nested-field.tsx — no form prop needed
import { useTypedAppFormContext } from '@/shared/components/form'

function NameField() {
  const form = useTypedAppFormContext({
    defaultValues: { name: '' as string },
  })

  return (
    <form.AppField name="name">
      {(field) => <field.TextField label={t('feature.name')} />}
    </form.AppField>
  )
}

// Must be rendered inside a <form.AppForm> or <group.AppForm>
```

## When to use which

| Situation | API |
|---|---|
| Split one form into tab/accordion sections | `withForm` |
| Reusable nested-object block (address, contact info) shared across forms | `withFieldGroup` |
| Deep child needs form state without prop drilling | `useTypedAppFormContext` |
| Programmatic lensing (loop, conditional) | `useFieldGroup` hook |

## Export from shared module

```ts
// src/shared/components/form/index.ts
export { useAppForm, withForm, withFieldGroup, useTypedAppFormContext } from './hooks/useAppForm'
```
