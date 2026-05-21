# modules/database.md — Database Setup

---

## Prisma setup

```bash
npx prisma init --datasource-provider postgresql
```

---

## schema.prisma base

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// Auth models — always included (see modules/auth.md)
// Add project models below this line
```

---

## Prisma client singleton

```typescript
// src/lib/prisma.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const prisma =
  globalForPrisma.prisma ?? new PrismaClient({ log: ['error'] })

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma
```

---

## DATABASE_URL format for AlwaysData PostgreSQL

```
DATABASE_URL="postgresql://USER:PASSWORD@postgresql-USER.alwaysdata.net:5432/USER_dbname"
```

Replace USER, PASSWORD, and dbname with values from AlwaysData panel.

---

## Migration commands

```bash
# Create and apply migration
npx prisma migrate dev --name init

# Apply migrations in production (CI/CD)
npx prisma migrate deploy

# Open Prisma Studio (local only)
npx prisma studio
```

---

## package.json scripts to add

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
- Never run `prisma migrate dev` on the production database
- DATABASE_URL must be set as a GitHub Secret
- Prisma generates the client automatically during `npm run build`
