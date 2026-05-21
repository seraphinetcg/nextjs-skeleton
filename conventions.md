# conventions.md — Folder Structure & Naming Rules

Follow these conventions in every project. Do not invent new patterns.

---

## Folder structure

```
src/
├── app/
│   ├── (auth)/
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── register/
│   │       └── page.tsx
│   ├── (admin)/
│   │   ├── layout.tsx          ← admin shell with sidebar/navbar
│   │   ├── dashboard/
│   │   │   └── page.tsx
│   │   └── [module]/           ← each CRUD module gets its own folder
│   │       ├── page.tsx        ← list view
│   │       ├── new/
│   │       │   └── page.tsx    ← create form
│   │       └── [id]/
│   │           └── page.tsx    ← edit form
│   ├── api/
│   │   └── auth/
│   │       └── [...nextauth]/
│   │           └── route.ts
│   ├── layout.tsx
│   └── page.tsx
│
├── components/
│   ├── ui/                     ← shadcn/ui components (auto-generated)
│   ├── layout/
│   │   ├── Sidebar.tsx
│   │   ├── Navbar.tsx
│   │   └── AdminShell.tsx
│   └── [module]/               ← components specific to a module
│       ├── [Module]Table.tsx
│       └── [Module]Form.tsx
│
├── lib/
│   ├── dal.ts                  ← Data Access Layer (all DB queries here)
│   ├── auth.ts                 ← Auth.js config
│   ├── prisma.ts               ← Prisma client singleton
│   ├── validations/
│   │   └── [module].ts         ← Zod schemas per module
│   └── utils.ts                ← clsx/twMerge helper + misc
│
├── actions/
│   └── [module].ts             ← Server Actions per module
│
└── types/
    └── index.ts                ← shared TypeScript types
```

---

## Naming rules

| Thing | Convention | Example |
|---|---|---|
| Components | PascalCase | `UserTable.tsx` |
| Server Actions files | camelCase | `actions/users.ts` |
| DAL functions | camelCase, verb first | `getUsers`, `createUser` |
| Zod schemas | PascalCase + Schema | `UserSchema`, `CreateUserSchema` |
| Route groups | lowercase in parens | `(admin)`, `(auth)` |
| Database models | PascalCase singular | `User`, `Product` |
| Database fields | camelCase | `createdAt`, `firstName` |

---

## Patterns

### Server Action pattern
```typescript
// actions/users.ts
'use server'
import { z } from 'zod'
import { revalidatePath } from 'next/cache'
import { CreateUserSchema } from '@/lib/validations/users'
import { createUser } from '@/lib/dal'

export async function createUserAction(formData: FormData) {
  const parsed = CreateUserSchema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return { error: parsed.error.flatten() }
  await createUser(parsed.data)
  revalidatePath('/admin/users')
}
```

### DAL pattern
```typescript
// lib/dal.ts
import { prisma } from '@/lib/prisma'

export async function getUsers() {
  return prisma.user.findMany({ orderBy: { createdAt: 'desc' } })
}

export async function createUser(data: CreateUserInput) {
  return prisma.user.create({ data })
}
```

### Auth guard pattern (server component)
```typescript
import { auth } from '@/lib/auth'
import { redirect } from 'next/navigation'

export default async function AdminPage() {
  const session = await auth()
  if (!session) redirect('/login')
  // render page
}
```

---

## What NOT to do

- No `useEffect` for data fetching — use Server Components
- No `fetch` inside components — use DAL functions
- No direct `prisma` calls outside `dal.ts`
- No inline styles — Tailwind only
- No `any` types
- No API routes for mutations — use Server Actions
