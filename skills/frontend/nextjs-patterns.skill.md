# Skill: Next.js Patterns — Server Components + Server Actions
**Used by:** `@nextjs-generator` | **Layer:** Frontend

---

## Page Structure by Domain

```
src/app/(dashboard)/{domain}/
  page.tsx                    ← Server Component — listing page
  [id]/
    page.tsx                  ← Server Component — detail/edit page
    loading.tsx               ← Skeleton while loading
    error.tsx                 ← Domain error boundary
  nuevo/
    page.tsx                  ← Creation page
  _actions/
    {domain}.actions.ts       ← Server Actions (mutations)
  _queries/
    {domain}.queries.ts       ← Server Queries (reads)
  _components/
    {Domain}Table.tsx         ← Domain table component
    {Domain}Form.tsx          ← Domain form component
    {Domain}SearchBar.tsx     ← Search bar component
```

---

## Server Component — Listing Page

```typescript
// page.tsx — Server Component (no 'use client')
// Fetches data directly on the server — no useEffect, no client-side fetch
import { Suspense } from 'react'
import { PageHeader } from '@/components/layout/PageHeader'
import { AppButton } from '@/components/ui/AppButton'
import { AdmisionTable } from './_components/AdmisionTable'
import { AdmisionTableSkeleton } from './_components/AdmisionTableSkeleton'
import { listarAdmisiones } from './_queries/admision.queries'

interface PageProps {
  searchParams: { page?: string; size?: string; estado?: string; fecha?: string }
}

export default async function AdmisionesPage({ searchParams }: PageProps) {
  const page = Number(searchParams.page ?? 0)
  const size = Number(searchParams.size ?? 20)

  return (
    <div className="space-y-6">
      <PageHeader
        title="Admissions"
        subtitle="Patient admission management"
        actions={
          <AppButton href="/admisiones/nuevo">
            New Admission
          </AppButton>
        }
      />

      {/* Suspense shows skeleton while loading — streaming */}
      <Suspense fallback={<AdmisionTableSkeleton />}>
        <AdmisionTableWrapper
          page={page}
          size={size}
          estado={searchParams.estado}
        />
      </Suspense>
    </div>
  )
}

// Nested async component — enables granular Suspense
async function AdmisionTableWrapper({ page, size, estado }: {
  page: number; size: number; estado?: string
}) {
  // Server-side fetch — no useEffect, no useState, no manual loading
  const data = await listarAdmisiones({ page, size, estado })

  return <AdmisionTable data={data} />
}
```

---

## Server Query — Data Reads

```typescript
// _queries/admision.queries.ts
// Runs on the server only — can call the backend directly
'use server'

import { getAuthToken } from '@/lib/auth'

const API_BASE = process.env.API_URL!   // server-side env var (no NEXT_PUBLIC_ prefix)

export interface AdmisionListResponse {
  content: AdmisionSummary[]
  totalElements: number
  totalPages: number
  number: number
  size: number
}

export async function listarAdmisiones(params: {
  page?: number
  size?: number
  estado?: string
  fechaDesde?: string
}): Promise<AdmisionListResponse> {

  const token = await getAuthToken()
  const searchParams = new URLSearchParams()

  if (params.page != null) searchParams.set('page', String(params.page))
  if (params.size != null) searchParams.set('size', String(params.size))
  if (params.estado) searchParams.set('estado', params.estado)
  if (params.fechaDesde) searchParams.set('fechaDesde', params.fechaDesde)

  const res = await fetch(`${API_BASE}/api/admisiones?${searchParams}`, {
    headers: { Authorization: `Bearer ${token}` },
    // next: { revalidate: 30 }   // revalidate cache every 30s if applicable
  })

  if (!res.ok) {
    throw new Error(`Error fetching admissions: ${res.status}`)
  }

  return res.json()
}

export async function obtenerAdmision(id: number): Promise<AdmisionDetail> {
  const token = await getAuthToken()
  const res = await fetch(`${API_BASE}/api/admisiones/${id}`, {
    headers: { Authorization: `Bearer ${token}` },
    cache: 'no-store'   // detail always fresh
  })

  if (res.status === 404) return null
  if (!res.ok) throw new Error(`Error: ${res.status}`)
  return res.json()
}
```

---

## Server Action — Mutations

```typescript
// _actions/admision.actions.ts
'use server'

import { revalidatePath } from 'next/cache'
import { redirect } from 'next/navigation'
import { z } from 'zod'
import { getAuthToken } from '@/lib/auth'

const API_BASE = process.env.API_URL!

// Zod validation schema — mirrors the backend Bean Validation
const CrearAdmisionSchema = z.object({
  pacienteId:       z.number({ required_error: 'Patient is required' }),
  tipoAdmisionId:   z.number({ required_error: 'Admission type is required' }),
  medicoTratanteId: z.number({ required_error: 'Doctor is required' }),
  fechaIngreso:     z.string().min(1, 'Admission date is required'),
  observaciones:    z.string().optional()
})

export type ActionResult<T = void> = {
  success: true; data?: T
} | {
  success: false; error: string; fieldErrors?: Record<string, string[]>
}

// Server Action — called from forms with action={} or from Client Components
export async function crearAdmision(
  formData: FormData
): Promise<ActionResult<{ id: number }>> {

  // 1. Validate on the server (never trust the client alone)
  const parsed = CrearAdmisionSchema.safeParse({
    pacienteId:       Number(formData.get('pacienteId')),
    tipoAdmisionId:   Number(formData.get('tipoAdmisionId')),
    medicoTratanteId: Number(formData.get('medicoTratanteId')),
    fechaIngreso:     formData.get('fechaIngreso'),
    observaciones:    formData.get('observaciones') || undefined
  })

  if (!parsed.success) {
    return {
      success: false,
      error: 'Validation error',
      fieldErrors: parsed.error.flatten().fieldErrors as Record<string, string[]>
    }
  }

  // 2. Call the backend
  const token = await getAuthToken()
  const res = await fetch(`${API_BASE}/api/admisiones`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${token}`
    },
    body: JSON.stringify(parsed.data)
  })

  if (!res.ok) {
    const errorBody = await res.json().catch(() => ({}))
    return { success: false, error: errorBody.message ?? 'Error creating admission' }
  }

  const id = await res.json()

  // 3. Invalidate cache and redirect
  revalidatePath('/admisiones')
  return { success: true, data: { id } }
}
```

---

## Client Component — Form with Server Action

```typescript
// _components/AdmisionForm.tsx
'use client'   // ← required for useState, useFormState, interactivity

import { useFormState, useFormStatus } from 'react-dom'
import { crearAdmision } from '../_actions/admision.actions'
import { AppButton } from '@/components/ui/AppButton'
import { AppInput } from '@/components/ui/AppInput'

const initialState = { success: false as const, error: '' }

export function AdmisionForm() {
  const [state, formAction] = useFormState(crearAdmision, initialState)

  return (
    <form action={formAction} className="grid grid-cols-2 gap-4">
      <AppInput
        name="pacienteId"
        label="Patient"
        required
        error={state.success === false ? state.fieldErrors?.pacienteId?.[0] : undefined}
      />
      <AppInput
        name="fechaIngreso"
        label="Admission Date"
        type="date"
        required
        error={state.success === false ? state.fieldErrors?.fechaIngreso?.[0] : undefined}
      />

      {/* General error message */}
      {state.success === false && state.error && (
        <div className="col-span-2 p-3 bg-red-50 border border-red-200 rounded text-red-700 text-sm">
          {state.error}
        </div>
      )}

      <div className="col-span-2 flex justify-end gap-3">
        <AppButton type="button" variant="outline" href="/admisiones">
          Cancel
        </AppButton>
        <SubmitButton />
      </div>
    </form>
  )
}

// Separate component to access useFormStatus
function SubmitButton() {
  const { pending } = useFormStatus()
  return (
    <AppButton type="submit" isLoading={pending}>
      Save Admission
    </AppButton>
  )
}
```

---

## Next.js Checklist — Before Generating

```
□ Are listing pages Server Components (no 'use client')?
□ Do Server Queries use 'use server' and the server-side token?
□ Do Server Actions validate with Zod before calling the backend?
□ Do forms use useFormState + useFormStatus for pending state?
□ Do pages have a loading.tsx with Skeleton for Suspense?
□ Are JWT tokens stored in httpOnly cookies (not localStorage)?
□ Do backend env vars NOT have the NEXT_PUBLIC_ prefix?
□ Is revalidatePath() called after successful mutations?
□ Are backend errors shown to the user with a readable message?
