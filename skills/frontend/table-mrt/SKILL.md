---
name: table-mrt
description: "How to build data tables using the DataTable component (Material React Table wrapper). Use this whenever creating a list/table view for a feature. Covers column definitions, server-side pagination, row actions menu, row selection for batch operations, custom top toolbar, and loading/error states."
---

# Data Tables with DataTable (MRT Wrapper)

The project wraps Material React Table in a shared `DataTable` component at `@/shared/components/DataTable`. Always use this wrapper — don't instantiate `MaterialReactTable` directly.

## Import

```tsx
import { DataTable } from '@/shared/components/DataTable'
import type { MRT_ColumnDef, MRT_RowSelectionState } from 'material-react-table'
```

## Basic Usage

```tsx
interface FeatureListProps {
  searchParams: FeatureFilterParams
  data?: Feature[]
  total?: number
  isLoading?: boolean
}

export function FeatureList({ searchParams, data = [], total = 0, isLoading }: FeatureListProps) {
  const { t } = useTranslation()
  const navigate = useNavigate({ from: '/your-section/your-feature/' })

  const columns = useMemo<MRT_ColumnDef<Feature>[]>(() => [
    {
      accessorKey: 'id',
      header: '#',
      size: 60,
      Cell: ({ row, table }) => {
        const { pageIndex, pageSize } = table.getState().pagination
        return pageIndex * pageSize + row.index + 1
      },
    },
    {
      accessorKey: 'name',
      header: t('feature.name'),
      size: 200,
      Cell: ({ cell }) => <>{cell.getValue() || '-'}</>,
    },
    // more columns...
  ], [t])

  return (
    <DataTable
      columns={columns}
      data={data}
      rowCount={total}
      isLoading={isLoading}
      page={searchParams.page || 1}
      pageSize={searchParams.page_size || 20}
      onPageChange={(newPage) => navigate({ search: (prev) => ({ ...prev, page: newPage }) })}
      onPageSizeChange={(newPageSize) => navigate({ search: (prev) => ({ ...prev, page_size: newPageSize, page: 1 }) })}
    />
  )
}
```

## Column Definitions

Use `accessorKey` for simple field access, `id` + `Cell` for computed or multi-field columns.

```tsx
const columns = useMemo<MRT_ColumnDef<Feature>[]>(() => [
  // Sequential row number (works with server-side pagination)
  {
    accessorKey: 'id',
    header: '#',
    size: 60,
    Cell: ({ row, table }) => {
      const { pageIndex, pageSize } = table.getState().pagination
      return pageIndex * pageSize + row.index + 1
    },
  },
  // Nullable field with fallback
  {
    accessorKey: 'serial_number',
    header: t('feature.serialNumber'),
    size: 150,
    Cell: ({ cell }) => <>{cell.getValue() || '-'}</>,
  },
  // Nested object
  {
    accessorKey: 'category.name',
    header: t('feature.category'),
    size: 150,
  },
  // Status chip
  {
    accessorKey: 'status',
    header: t('feature.status'),
    size: 120,
    Cell: ({ cell }) => (
      <Chip label={cell.getValue<string>()} color={cell.getValue() === 'active' ? 'success' : 'default'} size="small" />
    ),
  },
], [t])
```

## Row Actions Menu

Define `RowActionMenu` as a component inside `renderRowActions` to use hooks (like `useState`):

```tsx
<DataTable
  // ...
  enableRowActions
  positionActionsColumn="last"
  getRowId={(row: Feature) => String(row.id)}
  renderRowActions={({ row }) => {
    const RowActionMenu = () => {
      const { hasPermission } = usePermissions()
      const [anchorEl, setAnchorEl] = useState<null | HTMLElement>(null)
      const open = Boolean(anchorEl)

      const canView = hasPermission('feature-index')
      const canEdit = hasPermission('feature-edit')
      if (!canView && !canEdit) return null

      return (
        <>
          <Tooltip title={t('common.actions')} arrow>
            <IconButton size="small" onClick={(e) => setAnchorEl(e.currentTarget)}>
              <MoreVertIcon fontSize="small" />
            </IconButton>
          </Tooltip>
          <Menu anchorEl={anchorEl} open={open} onClose={() => setAnchorEl(null)}
            anchorOrigin={{ vertical: 'bottom', horizontal: 'right' }}
            transformOrigin={{ vertical: 'top', horizontal: 'right' }}
          >
            {canView && (
              <MenuItem onClick={() => { navigate({ to: '/your-section/feature/$featureId', params: { featureId: String(row.original.id) } }); setAnchorEl(null) }}>
                <ListItemIcon><VisibilityIcon fontSize="small" /></ListItemIcon>
                <ListItemText>{t('feature.viewDetails')}</ListItemText>
              </MenuItem>
            )}
            {canEdit && (
              <MenuItem onClick={() => { navigate({ to: '/your-section/feature/$featureId/edit', params: { featureId: String(row.original.id) } }); setAnchorEl(null) }}>
                <ListItemIcon><EditIcon fontSize="small" /></ListItemIcon>
                <ListItemText>{t('common.edit')}</ListItemText>
              </MenuItem>
            )}
          </Menu>
        </>
      )
    }
    return <RowActionMenu />
  }}
/>
```

## Row Selection for Batch Operations

```tsx
const [rowSelection, setRowSelection] = useState<MRT_RowSelectionState>({})

const selectedItems = useMemo(() => {
  const selectedIds = Object.keys(rowSelection).filter((key) => rowSelection[key])
  return data.filter((item) => selectedIds.includes(String(item.id)))
}, [rowSelection, data])

<DataTable
  // ...
  enableRowSelection={(row) => row.original.status !== 'exported'} // or just `true`
  getRowId={(row) => String(row.id)}
  rowSelection={rowSelection}
  onRowSelectionChange={setRowSelection}
  renderTopToolbarCustomActions={() => (
    <Box sx={{ display: 'flex', gap: 1, alignItems: 'center', width: '100%', justifyContent: 'space-between' }}>
      <Box sx={{ display: 'flex', gap: 1, alignItems: 'center' }}>
        {selectedItems.length > 0 && (
          <>
            <Chip label={t('feature.selected', { count: selectedItems.length })} color="primary" size="small" />
            <Button variant="contained" size="small" onClick={handleBatchAction}>
              {t('feature.batchAction', { count: selectedItems.length })}
            </Button>
          </>
        )}
      </Box>
      <Button variant="outlined" size="small" onClick={handleExport}>
        {t('feature.exportToExcel')}
      </Button>
    </Box>
  )}
/>
```

## DataTable Props Reference

| Prop | Type | Default | Purpose |
|---|---|---|---|
| `columns` | `MRT_ColumnDef<T>[]` | required | Column definitions |
| `data` | `T[]` | required | Row data |
| `rowCount` | `number` | `0` | Total rows (for server pagination) |
| `isLoading` | `boolean` | `false` | Skeleton loading state |
| `page` | `number` | `1` | Current page (1-indexed) |
| `pageSize` | `number` | `10` | Rows per page |
| `onPageChange` | `(p: number) => void` | — | Called on page change |
| `onPageSizeChange` | `(ps: number) => void` | — | Called on page size change |
| `enableRowActions` | `boolean` | `false` | Show actions column |
| `renderRowActions` | `fn` | — | Action cell renderer |
| `positionActionsColumn` | `'first' \| 'last'` | `'last'` | Actions column position |
| `enableRowSelection` | `boolean \| fn` | `false` | Enable row checkboxes |
| `getRowId` | `(row: T) => string` | — | Required when using row selection |
| `rowSelection` | `Record<string, boolean>` | — | Controlled selection state |
| `onRowSelectionChange` | `fn` | — | Selection change handler |
| `renderTopToolbarCustomActions` | `fn` | — | Custom toolbar content |

## Pagination

Pagination is server-side (`manualPagination`). The component converts 1-indexed pages from your URL params to 0-indexed for MRT internally. Always pass `page` and `pageSize` from `searchParams`, and update the URL on change via `navigate`.
