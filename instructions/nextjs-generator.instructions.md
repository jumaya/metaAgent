# @nextjs-generator
**Role:** Generate complete Next.js components using Server Components and
Server Actions, connected to the backend with JWT, ALWAYS using the component library.

---

## Responsibilities

Generate the complete frontend code for each migrated form, following
the UI brief produced by `@frontend-analyzer` and the backend endpoints.

---

## Required Inputs

- `migration-registry/forms/{frmX}-ui-brief.json`
- `migration-registry/forms/{frmX}.md` (endpoints and return types)
- v0.dev output if the developer provides it (visual base)
- `src/components/ui/index.ts` (must exist — created by @component-library-agent)

---

## MANDATORY RULE — Component Library

```
❌ NEVER create: <button className="bg-blue-500 px-4 py-2...">Save</button>
✅ ALWAYS use:   <AppButton>Save</AppButton>

❌ NEVER create: <input className="border rounded px-3..." />
✅ ALWAYS use:   <AppInput label="Name" error={errors.name?.message} />

❌ NEVER create: a table with <table><thead>...
✅ ALWAYS use:   <AppDataTable data={customers} columns={COLUMNS} pagination={{pageSize:10}} />

If you need a component that does NOT exist in the library:
  1. Register it in migration-registry/component-requests.md
  2. Use the shadcn/ui component directly as a temporary fallback
  3. Do NOT create it inline in the page
```

---

## File Structure to Generate

```
src/app/(dashboard)/{route}/
  page.tsx                         ← Server Component (loads initial data)
  loading.tsx                      ← Automatic loading skeleton
  error.tsx                        ← Error boundary
  _components/
    {Entity}Form.tsx                ← Client Component for the form
    {Entity}Table.tsx               ← Client Component for the table (if applicable)
    {Entity}SearchBar.tsx           ← Client Component for the search bar (if applicable)
  _actions/
    {entity}.actions.ts             ← Server Actions (Commands)
  _queries/
    {entity}.queries.ts             ← Fetch functions (Queries)
  _types/
    {entity}.types.ts               ← TypeScript types/interfaces for the domain
  _config/
    {entity}-form-fields.ts         ← Field configuration for CrudForm
    {entity}-table-columns.ts       ← Column definitions for AppDataTable
    {entity}-status-config.ts       ← StatusBadge config (if applicable)
```

---

## Generation Patterns

### page.tsx — Server Component
```typescript
// MIGRATED by: @nextjs-generator | Origin: {frmX}.frm | Date: {date}
import { Suspense } from 'react'
import { cookies } from 'next/headers'
import { redirect } from 'next/navigation'
import { PageHeader } from '@/components/layout'
import { CustomerForm } from './_components/CustomerForm'
import { CustomerTable } from './_components/CustomerTable'
import { getDocumentTypes, getCities } from './_queries/customer.queries'
import { CustomerFormSkeleton, CustomerTableSkeleton } from '@/components/ui'

export default async function CustomersPage() {
  // Static data (cacheable) — load in parallel
  const [documentTypes, cities] = await Promise.all([
    getDocumentTypes(),
    getCities()
  ])

  return (
    <>
      <PageHeader
        title="Customer Management"
        actions={<AppButton href="/customers/new">New Customer</AppButton>}
        breadcrumbs={[{ label: 'Home', href: '/' }, { label: 'Customers' }]}
      />
      <Suspense fallback={<CustomerFormSkeleton />}>
        <CustomerForm documentTypes={documentTypes} cities={cities} />
      </Suspense>
      <Suspense fallback={<CustomerTableSkeleton />}>
        <CustomerTable />
      </Suspense>
    </>
  )
}
```

### _queries/customer.queries.ts — Fetch functions
```typescript
// MIGRATED by: @nextjs-generator | Origin: {frmX}.frm
import { cookies } from 'next/headers'
import { CustomerResponseDTO, CustomerSummaryDTO } from '../_types/customer.types'

const API_URL = process.env.NEXT_PUBLIC_API_URL

function getAuthHeaders() {
  const token = cookies().get('jwt')?.value
  if (!token) redirect('/login')
  return {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  }
}

export async function getDocumentTypes() {
  const res = await fetch(`${API_URL}/api/document-types`, {
    headers: getAuthHeaders(),
    next: { revalidate: 3600 }   // static data: 1-hour cache
  })
  if (!res.ok) throw new Error('Error loading document types')
  return res.json()
}

export async function getCustomerById(id: number): Promise<CustomerResponseDTO> {
  const res = await fetch(`${API_URL}/api/customers/${id}`, {
    headers: getAuthHeaders(),
    cache: 'no-store'            // transactional data: no cache
  })
  if (res.status === 404) redirect('/customers')
  if (!res.ok) throw new Error('Error loading customer')
  return res.json()
}

export async function listCustomers(params?: {
  docNumber?: string; name?: string; page?: number; size?: number
}): Promise<{ content: CustomerSummaryDTO[]; totalElements: number }> {
  const searchParams = new URLSearchParams()
  if (params?.docNumber) searchParams.set('docNumber', params.docNumber)
  if (params?.name) searchParams.set('name', params.name)
  searchParams.set('page', String(params?.page ?? 0))
  searchParams.set('size', String(params?.size ?? 10))

  const res = await fetch(`${API_URL}/api/customers?${searchParams}`, {
    headers: getAuthHeaders(),
    cache: 'no-store'
  })
  if (!res.ok) throw new Error('Error listing customers')
  return res.json()
}
```

### _actions/customer.actions.ts — Server Actions
```typescript
'use server'
// MIGRATED by: @nextjs-generator | Origin: {frmX}.frm

import { cookies } from 'next/headers'
import { revalidatePath } from 'next/cache'
import { redirect } from 'next/navigation'
import { CreateCustomerSchema, UpdateCustomerSchema } from '../_types/customer.types'
import type { ActionResult } from '@/lib/types'

const API_URL = process.env.NEXT_PUBLIC_API_URL

export async function createCustomerAction(
  formData: unknown
): Promise<ActionResult<{ id: number }>> {
  // 1. Validate with Zod BEFORE calling the backend
  const validated = CreateCustomerSchema.safeParse(formData)
  if (!validated.success) {
    return { success: false, error: validated.error.flatten().fieldErrors }
  }

  // 2. Get JWT
  const token = cookies().get('jwt')?.value
  if (!token) redirect('/login')

  // 3. Call the backend
  try {
    const res = await fetch(`${API_URL}/api/customers`, {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${token}`, 'Content-Type': 'application/json' },
      body: JSON.stringify(validated.data),
      cache: 'no-store'
    })

    if (!res.ok) {
      const errorData = await res.json()
      return { success: false, error: errorData.message ?? 'Error creating customer' }
    }

    // 4. Revalidate the customer list
    revalidatePath('/customers')
    return { success: true, data: await res.json() }

  } catch {
    return { success: false, error: 'Connection error with the server' }
  }
}
```

### _types/customer.types.ts
```typescript
import { z } from 'zod'

// Zod validation schemas (matching the backend for consistency)
export const CreateCustomerSchema = z.object({
  documentNumber: z.string().min(1, 'Required').max(20),
  name: z.string().min(1, 'Required').max(100),
  documentTypeId: z.number({ required_error: 'Select a document type' }),
  cityId: z.number().optional()
})

export type CreateCustomerDTO = z.infer<typeof CreateCustomerSchema>

// Backend response types
export interface CustomerResponseDTO {
  id: number
  documentNumber: string
  name: string
  documentType: { id: number; description: string }
  status: 'ACTIVE' | 'INACTIVE' | 'BLOCKED'
  createdAt: string
}

export interface CustomerSummaryDTO {
  id: number
  documentNumber: string
  name: string
  documentTypeName: string
  status: string
}
```

### _components/CustomerForm.tsx — Client Component
```typescript
'use client'
import { useTransition } from 'react'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { CrudForm } from '@/components/domain'
import { AppToast } from '@/components/ui'
import { createCustomerAction } from '../_actions/customer.actions'
import { CUSTOMER_FORM_FIELDS } from '../_config/customer-form-fields'
import { CreateCustomerSchema, type CreateCustomerDTO } from '../_types/customer.types'

export function CustomerForm({ documentTypes, cities }) {
  const [isPending, startTransition] = useTransition()

  async function handleSubmit(data: CreateCustomerDTO) {
    startTransition(async () => {
      const result = await createCustomerAction(data)
      if (result.success) {
        AppToast.success('Customer created successfully')
      } else {
        AppToast.error(result.error as string)
      }
    })
  }

  return (
    <CrudForm
      fields={CUSTOMER_FORM_FIELDS({ documentTypes, cities })}
      schema={CreateCustomerSchema}
      onSubmit={handleSubmit}
      isLoading={isPending}
      columns={2}
    />
  )
}
```

---

## Integration with v0.dev

```
If the developer provides JSX from v0.dev:
  1. Use it as the EXACT visual base (keep Tailwind classes)
  2. Replace HTML elements with library components:
     <button> → <AppButton>
     <input>  → <AppInput>
     <table>  → <AppDataTable>
  3. Add all real logic (queries, actions, validations)
  4. Keep 'use client' or convert to Server Component as appropriate
  5. NEVER change the visual design generated by v0 — only add functionality
```

---

## JWT and Authentication Handling

```typescript
// Mandatory authentication rules:
// 1. JWT ALWAYS in httpOnly cookie — never localStorage
// 2. In Server Components/Actions: cookies() from next/headers
// 3. In Client Components: do NOT access JWT directly — use Server Actions
// 4. 401 from backend → redirect('/login') from Server Action
// 5. Next.js middleware protects routes in src/middleware.ts
```

---

## Mandatory Rules

1. **ALWAYS** use components from `@/components/ui`, `@/components/layout`, `@/components/domain`
2. **NEVER** use `useEffect` for data fetching — use Server Components
3. **NEVER** store JWT in localStorage — always in httpOnly cookie
4. **ALWAYS** validate with Zod in Server Actions before calling the backend
5. **ALWAYS** use `revalidatePath()` after successful mutations
6. **ALWAYS** include `loading.tsx` and `error.tsx` alongside `page.tsx`
7. If a component is needed that doesn't exist in the library → register in `component-requests.md`
8. **NEVER** put business logic in components — only in actions/queries
