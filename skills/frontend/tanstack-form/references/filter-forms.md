# Filter Forms (URL-Synced) & Reactive Fields

## Filter Form

Filter forms wrap fields in an Accordion and sync with URL search params via `useSearch` + `navigate`.

```tsx
export function FeatureFilters() {
  const { t } = useTranslation()
  const searchParams = useSearch({ from: '/_layout/your-section/feature/' })
  const navigate = Route.useNavigate()
  const [expanded, setExpanded] = useState(false)

  const form = useAppForm({
    defaultValues: {
      search: searchParams.search || '',
      status: searchParams.status || '',
    },
    onSubmit: async ({ value }) => {
      navigate({ search: (prev) => ({ ...prev, page: 1, ...value }) })
      setTimeout(() => setExpanded(false), 0)
    },
  })

  // Sync form when URL changes (browser back/forward)
  useEffect(() => {
    form.setFieldValue('search', searchParams.search || '')
    form.setFieldValue('status', searchParams.status || '')
  }, [searchParams])

  const handleReset = () => {
    form.reset()
    navigate({ search: { page: 1, page_size: searchParams.page_size } })
  }

  return (
    <Accordion expanded={expanded} onChange={(_, isExpanded) => setExpanded(isExpanded)}>
      <AccordionSummary expandIcon={<ExpandMoreIcon />}>
        <Box sx={{ display: 'flex', alignItems: 'center', gap: 1 }}>
          <FilterListIcon />
          {t('feature.filters')}
        </Box>
      </AccordionSummary>
      <AccordionDetails>
        <form.AppForm>
          <Box component="form" onSubmit={(e) => { e.preventDefault(); form.handleSubmit() }} noValidate>
            <Stack spacing={2}>
              <form.AppField name="search">
                {(field) => <field.TextField label={t('common.search')} />}
              </form.AppField>
              <form.AppField name="status">
                {(field) => (
                  <field.Select
                    label={t('feature.status')}
                    options={STATUS_OPTIONS.map(o => ({ value: o.value, label: t(o.label) }))}
                    clearable
                  />
                )}
              </form.AppField>
              <Box sx={{ display: 'flex', gap: 2, justifyContent: 'flex-end' }}>
                <Button variant="outlined" onClick={handleReset}>{t('common.reset')}</Button>
                <form.SubmitButton label={t('common.search')} />
              </Box>
            </Stack>
          </Box>
        </form.AppForm>
      </AccordionDetails>
    </Accordion>
  )
}
```

## Reactive Fields (Cascading Dropdowns)

Watch a field value to drive another field's options:

```tsx
import { useStore } from '@tanstack/react-form'

const categoryId = useStore(form.store, (state) => state.values.category_id)

// Re-fetches when categoryId changes
const { data: subCategories } = useSubCategories({ category_id: categoryId })
```

Reset dependent fields when parent changes:
```tsx
<field.Select
  label="Category"
  options={...}
  onChange={() => form.setFieldValue('sub_category_id', null)}
/>
```
