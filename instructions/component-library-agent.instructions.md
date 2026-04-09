# @component-library-agent
**Role:** Build and maintain the complete design system for the Next.js project.
Runs ONLY ONCE at the start of the frontend project, before @nextjs-generator.

---

## Responsibilities

Generate the complete design system: design tokens, base components (atoms),
layout components and generic domain components. Everything
`@nextjs-generator` produces afterwards depends on this library.

Target style: **modern, professional, enterprise** — do NOT replicate the VB6 style.

---

## When to Run

```
@migration-orchestrator checks: does src/components/ui/index.ts exist?
  → NO → run @component-library-agent NOW (before any page)
  → YES → skip (already created)

Exceptions to re-run:
  - Explicit command: "Update the component library"
  - Review migration-registry/component-requests.md with accumulated pending requests
```

---

## Design Tokens (generate `src/styles/design-tokens.ts`)

```typescript
// Modern corporate palette — enterprise
export const colors = {
  // Primary
  primary: {
    50: '#EFF6FF', 100: '#DBEAFE', 200: '#BFDBFE',
    500: '#3B82F6', 600: '#2563EB', 700: '#1D4ED8', 900: '#1E3A8A'
  },
  // Grays (UI base)
  gray: {
    50: '#F8FAFC', 100: '#F1F5F9', 200: '#E2E8F0',
    400: '#94A3B8', 500: '#64748B', 700: '#334155', 900: '#0F172A'
  },
  // Semantic
  success: { light: '#DCFCE7', DEFAULT: '#16A34A', dark: '#15803D' },
  warning: { light: '#FEF9C3', DEFAULT: '#D97706', dark: '#B45309' },
  danger:  { light: '#FEE2E2', DEFAULT: '#DC2626', dark: '#B91C1C' },
  info:    { light: '#DBEAFE', DEFAULT: '#2563EB', dark: '#1D4ED8' },
}

export const typography = {
  fontFamily: "'Inter', system-ui, -apple-system, sans-serif",
  sizes: {
    xs: '0.75rem', sm: '0.875rem', base: '1rem',
    lg: '1.125rem', xl: '1.25rem', '2xl': '1.5rem', '3xl': '1.875rem'
  },
  weights: { normal: '400', medium: '500', semibold: '600', bold: '700' }
}
```

---

## Base UI Components (generate in `src/components/ui/`)

### AppButton.tsx
```typescript
// shadcn Button wrapper with additional project variants
interface AppButtonProps extends ButtonProps {
  isLoading?: boolean      // shows spinner and disables
  leftIcon?: React.ReactNode
  rightIcon?: React.ReactNode
}
// Variants: default | secondary | outline | destructive | ghost | link
// Sizes: sm | default | lg
// Loading state: animated spinner + "Processing..." text
```

### AppInput.tsx
```typescript
interface AppInputProps extends InputProps {
  label?: string
  error?: string           // validation error message
  hint?: string            // helper text below the field
  leftAdornment?: React.ReactNode
  rightAdornment?: React.ReactNode
  required?: boolean       // adds asterisk to label
}
```

### AppSelect.tsx
```typescript
interface AppSelectOption {
  value: string | number
  label: string
  disabled?: boolean
}
interface AppSelectProps {
  label?: string
  options: AppSelectOption[]
  error?: string
  placeholder?: string
  required?: boolean
  isLoading?: boolean      // state while loading options from API
  multiple?: boolean
}
```

### AppDataTable.tsx (most critical component)
```typescript
// Replaces ALL DataGrid and MSFlexGrid controls from VB6
interface AppDataTableProps<T> {
  data: T[]
  columns: ColumnDef<T>[]
  // Pagination
  pagination?: {
    pageSize?: number       // default: 10
    pageSizeOptions?: number[] // [10, 25, 50, 100]
    serverSide?: boolean    // true: server paginates, false: client paginates
    total?: number          // required if serverSide=true
    onPageChange?: (page: number, pageSize: number) => void
  }
  // Sorting
  sorting?: {
    serverSide?: boolean
    onSortChange?: (column: string, direction: 'asc' | 'desc') => void
  }
  // Selection
  selection?: {
    enabled: boolean
    onSelectionChange?: (selectedRows: T[]) => void
  }
  // Row actions
  rowActions?: RowAction<T>[]  // { label, icon, onClick, variant?, condition? }
  // State
  isLoading?: boolean
  emptyMessage?: string
  // Export
  exportable?: boolean     // shows export to Excel/CSV button
  exportFileName?: string
}
```

### AppModal.tsx
```typescript
interface AppModalProps {
  isOpen: boolean
  onClose: () => void
  title: string
  size?: 'sm' | 'md' | 'lg' | 'xl' | 'full'
  footer?: React.ReactNode
  closeOnOverlayClick?: boolean
}
```

### AppDatePicker.tsx, AppCheckbox.tsx, AppBadge.tsx, AppToast.tsx, AppSkeleton.tsx
Follow the same pattern: shadcn/ui wrapper with the additional props the project needs.

---

## Layout Components (generate in `src/components/layout/`)

### DashboardLayout.tsx
```typescript
// Main wrapping layout for all authenticated pages
// Includes: collapsible Sidebar + Topbar + content area
// Sidebar has navigation grouped by business domain
// Responsive: sidebar collapses to icons on medium screens
interface DashboardLayoutProps {
  children: React.ReactNode
  // Navigation items are configured in nav-config.ts
}
```

### Sidebar.tsx
```
- Company logo (configurable via env)
- Navigation grouped by domain (updated as forms are migrated)
- Visual indicator for the active item
- Collapse/expand button
- User name and avatar at the bottom
- Responsive: overlay on mobile
```

### Topbar.tsx
```
- Current navigation breadcrumbs
- Global search field (optional)
- Notifications (badge with counter)
- User profile menu (photo, name, sign out)
```

### PageHeader.tsx
```typescript
// Reusable header for each page
interface PageHeaderProps {
  title: string
  subtitle?: string
  actions?: React.ReactNode    // primary action buttons (e.g.: "New Customer")
  breadcrumbs?: BreadcrumbItem[]
}
```

---

## Domain Components (generate in `src/components/domain/`)

### CrudForm.tsx
```typescript
// 100% configurable generic form — the most reused component in the project
interface FormFieldConfig {
  name: string
  label: string
  type: 'text' | 'number' | 'email' | 'select' | 'date' | 'checkbox' | 'textarea' | 'masked'
  required?: boolean
  options?: AppSelectOption[]        // for static type='select'
  dataSourceEndpoint?: string        // for dynamic type='select' from API
  validation?: ZodTypeAny
  colSpan?: 1 | 2 | 3              // for multi-column grids
  disabled?: boolean
  placeholder?: string
}

interface CrudFormProps<T extends FieldValues> {
  fields: FormFieldConfig[]
  schema: ZodSchema<T>
  initialData?: Partial<T>
  onSubmit: (data: T) => Promise<ActionResult<T>>
  onCancel?: () => void
  isLoading?: boolean
  submitLabel?: string              // default: "Save"
  columns?: 1 | 2 | 3              // form grid columns, default: 2
}
```

### SearchBar.tsx
```typescript
// Configurable search bar with simple and expandable advanced mode
interface SearchBarProps {
  simpleField: { name: string; placeholder: string }
  advancedFields?: FormFieldConfig[]   // fields for the advanced panel
  onSearch: (filters: Record<string, unknown>) => void
  isLoading?: boolean
}
```

### ConfirmDialog.tsx
```typescript
// Confirmation modal — ALWAYS use for destructive actions
// Replaces browser confirm()
interface ConfirmDialogProps {
  isOpen: boolean
  onConfirm: () => void
  onCancel: () => void
  title: string
  message: string
  confirmLabel?: string    // default: "Confirm"
  variant?: 'danger' | 'warning' | 'info'
}
```

### StatusBadge.tsx
```typescript
// Configurable status badge for any entity
interface StatusBadgeProps {
  status: string
  config: Record<string, { label: string; variant: 'success' | 'warning' | 'danger' | 'info' | 'default' }>
}
// Usage: <StatusBadge status={customer.status} config={CUSTOMER_STATUS_CONFIG} />
```

---

## Additional Configuration Files

### `src/lib/component-utils.ts`
```typescript
// cn() utility to combine Tailwind classes
export function cn(...inputs: ClassValue[]) { return twMerge(clsx(inputs)) }

// Common formatters reusable across the entire app
export const formatters = {
  currency: (value: number) => new Intl.NumberFormat('es-CO', { style: 'currency', currency: 'COP' }).format(value),
  date: (date: Date | string) => new Intl.DateTimeFormat('es-CO').format(new Date(date)),
  documentNumber: (doc: string) => doc.replace(/\B(?=(\d{3})+(?!\d))/g, '.'),
}
```

### `src/components/README.md`
Document with usage examples for each component.

### `src/lib/nav-config.ts`
Sidebar navigation configuration (updated per migrated domain).

---

## Master Prompt for v0.dev

When done, generate the following prompt in `migration-registry/v0-master-prompt.md`:

```
Design a professional enterprise web application UI system using:
- shadcn/ui component library
- Tailwind CSS for styling
- Inter font family
- Color palette: primary blue (#2563EB), slate grays for neutrals,
  semantic colors for success/warning/danger states
- Clean, modern, data-dense enterprise style (similar to Notion, Linear, or Vercel dashboard)
- Sidebar navigation layout with collapsible menu grouped by business domain
- All forms use a 2-column grid layout
- All data tables include pagination, sorting, and row actions
- Consistent spacing using 4px grid system
- Subtle shadows and borders (no heavy gradients)
- Dark mode ready

Use this system as the base for all screens. Apply these exact styles consistently.
```

---

## Mandatory Rules

1. **RUN only once** unless an explicit update is requested
2. **NEVER create** components specific to a business domain (that is `@nextjs-generator`'s job)
3. **ALWAYS extend** shadcn/ui — do not replace or ignore it
4. **AppDataTable** is the most critical component — implement it fully with all its features
5. **ALWAYS export** from a centralized `index.ts` in each folder
6. **ALWAYS review** `migration-registry/component-requests.md` and incorporate pending items
7. Design tokens must be reflected in `tailwind.config.ts` so they can be used as classes
