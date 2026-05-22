# modules/database.md — Database Setup

---

## Prisma setup

```bash
npx prisma init --datasource-provider postgresql
```

---

## schema.prisma — complete base schema

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ─── AUTH (Auth.js required) ──────────────────────────────

model User {
  id            String         @id @default(cuid())
  name          String?
  email         String         @unique
  password      String?
  role          String         @default("user")
  emailVerified DateTime?
  image         String?
  createdAt     DateTime       @default(now())
  updatedAt     DateTime       @updatedAt
  accounts      Account[]
  sessions      Session[]
  notifications Notification[]
  auditLogs     AuditLog[]
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String?
  access_token      String?
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?
  session_state     String?
  user              User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  id        String   @id @default(cuid())
  email     String
  token     String   @unique
  type      String   @default("email_verification")
  expiresAt DateTime
  createdAt DateTime @default(now())
  @@unique([email, type])
}

// ─── AUDIT LOG ────────────────────────────────────────────

model AuditLog {
  id        String   @id @default(cuid())
  userId    String?
  action    String   // CREATE | UPDATE | DELETE | LOGIN | LOGOUT
  entity    String   // model name: "User", "Product", etc.
  entityId  String?
  changes   Json?    // { before: {}, after: {} }
  ip        String?
  userAgent String?
  createdAt DateTime @default(now())
  user      User?    @relation(fields: [userId], references: [id], onDelete: SetNull)
}

// ─── SETTINGS ─────────────────────────────────────────────

model Setting {
  id        String   @id @default(cuid())
  key       String   @unique
  value     String
  updatedAt DateTime @updatedAt
}

// ─── NOTIFICATIONS ────────────────────────────────────────

model Notification {
  id        String   @id @default(cuid())
  userId    String
  title     String
  message   String
  type      String   @default("info")
  read      Boolean  @default(false)
  link      String?
  createdAt DateTime @default(now())
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

// ─── FILES ────────────────────────────────────────────────

model File {
  id         String   @id @default(cuid())
  filename   String
  mimetype   String
  size       Int
  path       String
  url        String
  uploadedBy String
  createdAt  DateTime @default(now())
}

// ─── PROJECT MODELS ───────────────────────────────────────
// Add client-specific models below this line
```

---

## Prisma client with automatic AuditLog

This is the most important file. The Prisma Extension intercepts every
create, update, and delete across ALL models and logs them automatically.
No need to manually call the audit log in every Server Action.

```typescript
// src/lib/prisma.ts
import { PrismaClient } from '@prisma/client'
import { headers } from 'next/headers'
import { auth } from '@/lib/auth'

// Models excluded from audit logging
const EXCLUDED_MODELS = ['AuditLog', 'Session', 'Account']

// Actions to intercept
const AUDIT_ACTIONS = ['create', 'update', 'delete', 'updateMany', 'deleteMany']

function createPrismaClient() {
  const client = new PrismaClient({ log: ['error'] })

  return client.$extends({
    query: {
      $allModels: {
        async $allOperations({ model, operation, args, query }) {
          // run the original query first
          const result = await query(args)

          // skip excluded models and non-mutating operations
          if (
            !model ||
            EXCLUDED_MODELS.includes(model) ||
            !AUDIT_ACTIONS.includes(operation)
          ) {
            return result
          }

          try {
            // get current session (may not exist in some contexts)
            const session = await auth().catch(() => null)
            const userId = session?.user?.id ?? null

            // get IP and user agent from request headers
            const headersList = await headers().catch(() => null)
            const ip = headersList?.get('x-forwarded-for') ?? null
            const userAgent = headersList?.get('user-agent') ?? null

            // determine action label
            const actionMap: Record<string, string> = {
              create: 'CREATE',
              update: 'UPDATE',
              updateMany: 'UPDATE',
              delete: 'DELETE',
              deleteMany: 'DELETE',
            }

            // extract entityId when possible
            const entityId =
              (result as any)?.id ??
              (args as any)?.where?.id ??
              null

            // build changes object
            const changes = (args as any)?.data
              ? { after: (args as any).data }
              : null

            await client.auditLog.create({
              data: {
                userId,
                action: actionMap[operation] ?? operation.toUpperCase(),
                entity: model,
                entityId: entityId ? String(entityId) : null,
                changes,
                ip,
                userAgent,
              },
            })
          } catch {
            // never block the main operation if audit fails
          }

          return result
        },
      },
    },
  })
}

const globalForPrisma = globalThis as unknown as {
  prisma: ReturnType<typeof createPrismaClient>
}

export const prisma = globalForPrisma.prisma ?? createPrismaClient()

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma
```

---

## AuditLog DAL functions

```typescript
// add to src/lib/dal.ts

export async function getAuditLogs({
  entity,
  entityId,
  userId,
  limit = 50,
}: {
  entity?: string
  entityId?: string
  userId?: string
  limit?: number
} = {}) {
  return prisma.auditLog.findMany({
    where: {
      ...(entity && { entity }),
      ...(entityId && { entityId }),
      ...(userId && { userId }),
    },
    include: { user: { select: { name: true, email: true } } },
    orderBy: { createdAt: 'desc' },
    take: limit,
  })
}
```

---

## AuditLog admin page

Generate a read-only page at `/admin/audit`:
- Table with columns: Date, User, Action, Entity, Entity ID, Changes
- Filter by entity (dropdown) and date range
- Color badges per action: CREATE=green, UPDATE=blue, DELETE=red
- No edit or delete buttons — audit log is read-only forever
- Only accessible by users with role `admin`

---

## Manual audit for special actions (LOGIN, LOGOUT)

For actions not covered by Prisma (login, logout, failed attempts),
call the audit log directly:

```typescript
// usage in auth callbacks or server actions
await prisma.auditLog.create({
  data: {
    userId: user.id,
    action: 'LOGIN',
    entity: 'User',
    entityId: user.id,
    ip: headers().get('x-forwarded-for'),
  },
})
```

---

## Settings DAL functions

```typescript
// add to src/lib/dal.ts

export async function getSetting(key: string) {
  const setting = await prisma.setting.findUnique({ where: { key } })
  return setting?.value ?? null
}

export async function setSetting(key: string, value: string) {
  return prisma.setting.upsert({
    where: { key },
    update: { value },
    create: { key, value },
  })
}

export async function getAllSettings() {
  return prisma.setting.findMany({ orderBy: { key: 'asc' } })
}
```

Default settings to seed on first deploy:
```typescript
await setSetting('site_name', 'Mi Empresa')
await setSetting('maintenance_mode', 'false')
await setSetting('allow_registration', 'true')
```

---

## DATABASE_URL format for AlwaysData PostgreSQL

```
DATABASE_URL="postgresql://USER:PASSWORD@postgresql-USER.alwaysdata.net:5432/USER_dbname"
```

---

## Migration commands

```bash
# Create and apply migration (local)
npx prisma migrate dev --name init

# Apply in production (CI/CD only)
npx prisma migrate deploy

# Prisma Studio (local only)
npx prisma studio
```

---

## package.json scripts

```json
{
  "scripts": {
    "db:migrate": "prisma migrate dev",
    "db:deploy": "prisma migrate deploy",
    "db:studio": "prisma studio",
    "db:generate": "prisma generate"
  }
}
```

---

## Production notes

- Run `prisma migrate deploy` (not `dev`) in CI/CD
- Never run `prisma migrate dev` on production
- DATABASE_URL must be set as a GitHub Secret
- AuditLog table grows over time — add a cleanup job after 6 months if needed
- Never delete AuditLog records manually — they are the source of truth
