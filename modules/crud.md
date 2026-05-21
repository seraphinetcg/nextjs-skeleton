# modules/crud.md — Generic CRUD Maintainer

When asked to create a maintainer for any entity, follow this spec exactly.
Replace `[Entity]` with the actual entity name (e.g., Product, Client, Category).

---

## What to generate per module

1. Prisma model → add to `schema.prisma`
2. Zod schema → `src/lib/validations/[entity].ts`
3. DAL functions → add to `src/lib/dal.ts`
4. Server Actions → `src/actions/[entity].ts`
5. List page → `src/app/(admin)/[entity]/page.tsx`
6. Create page → `src/app/(admin)/[entity]/new/page.tsx`
7. Edit page → `src/app/(admin)/[entity]/[id]/page.tsx`
8. Table component → `src/components/[entity]/[Entity]Table.tsx`
9. Form component → `src/components/[entity]/[Entity]Form.tsx`

---

## Base Prisma model pattern

```prisma
model [Entity] {
  id        String   @id @default(cuid())
  // ... entity fields here
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

---

## Zod schema pattern

```typescript
// src/lib/validations/[entity].ts
import { z } from 'zod'

export const Create[Entity]Schema = z.object({
  // fields with validation messages in Spanish
})

export const Update[Entity]Schema = Create[Entity]Schema.partial()

export type Create[Entity]Input = z.infer<typeof Create[Entity]Schema>
export type Update[Entity]Input = z.infer<typeof Update[Entity]Schema>
```

---

## DAL functions pattern

```typescript
// add to src/lib/dal.ts

export async function get[Entity]s(search?: string) {
  return prisma.[entity].findMany({
    where: search
      ? { name: { contains: search, mode: 'insensitive' } }
      : undefined,
    orderBy: { createdAt: 'desc' },
  })
}

export async function get[Entity]ById(id: string) {
  return prisma.[entity].findUnique({ where: { id } })
}

export async function create[Entity](data: Create[Entity]Input) {
  return prisma.[entity].create({ data })
}

export async function update[Entity](id: string, data: Update[Entity]Input) {
  return prisma.[entity].update({ where: { id }, data })
}

export async function delete[Entity](id: string) {
  return prisma.[entity].delete({ where: { id } })
}
```

---

## Server Actions pattern

```typescript
// src/actions/[entity].ts
'use server'
import { revalidatePath } from 'next/cache'
import { redirect } from 'next/navigation'
import { auth } from '@/lib/auth'
import { Create[Entity]Schema } from '@/lib/validations/[entity]'
import { create[Entity], update[Entity], delete[Entity] } from '@/lib/dal'

export async function create[Entity]Action(formData: FormData) {
  const session = await auth()
  if (!session) redirect('/login')

  const parsed = Create[Entity]Schema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return { error: parsed.error.flatten() }

  await create[Entity](parsed.data)
  revalidatePath('/admin/[entity]')
  redirect('/admin/[entity]')
}

export async function update[Entity]Action(id: string, formData: FormData) {
  const session = await auth()
  if (!session) redirect('/login')

  const parsed = Create[Entity]Schema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return { error: parsed.error.flatten() }

  await update[Entity](id, parsed.data)
  revalidatePath('/admin/[entity]')
  redirect('/admin/[entity]')
}

export async function delete[Entity]Action(id: string) {
  const session = await auth()
  if (!session) redirect('/login')

  await delete[Entity](id)
  revalidatePath('/admin/[entity]')
}
```

---

## List page pattern

```typescript
// src/app/(admin)/[entity]/page.tsx
import { get[Entity]s } from '@/lib/dal'
import { [Entity]Table } from '@/components/[entity]/[Entity]Table'
import { Button } from '@/components/ui/button'
import Link from 'next/link'

export default async function [Entity]Page({
  searchParams,
}: {
  searchParams: { search?: string }
}) {
  const items = await get[Entity]s(searchParams.search)

  return (
    <div className="space-y-4">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">[Entities]</h1>
        <Button asChild>
          <Link href="/admin/[entity]/new">Nuevo</Link>
        </Button>
      </div>
      <[Entity]Table items={items} />
    </div>
  )
}
```

---

## Table component pattern

Use shadcn/ui `Table` component.
Include columns: main fields + createdAt + actions (Edit, Delete).
Delete button calls the delete Server Action with a confirm dialog.
Search input at the top updates the URL `?search=` param.

---

## Form component pattern

Use shadcn/ui `Card` + `Input` + `Label` + `Button`.
Show validation errors inline below each field in red.
Show a loading state on the submit button.
Cancel button goes back to the list.
Works for both create and edit (receives optional `defaultValues`).

---

## Pagination

When list exceeds 20 items, add URL-based pagination:
- `?page=1` param
- Show prev/next buttons
- Show current page / total pages
- DAL function uses `skip` and `take`
