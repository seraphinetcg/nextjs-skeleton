# AGENTS.md — Next.js Skeleton Instructions

You are an AI coding agent starting a new client web project.
Read this file completely before writing any code.
Then read every file listed under "Required Reading" before generating anything.

---

## Stack

- **Framework:** Next.js 16 (App Router)
- **Language:** TypeScript
- **Styling:** Tailwind CSS + shadcn/ui
- **ORM:** Prisma
- **Database:** PostgreSQL (AlwaysData)
- **Auth:** Auth.js v5 (formerly NextAuth)
- **Deploy:** GitHub Actions → AlwaysData via SSH

Full details in `stack.md`.

---

## Required Reading (in order)

1. `stack.md` — versions, libraries, and non-negotiable decisions
2. `conventions.md` — folder structure, naming rules, patterns to follow
3. `modules/auth.md` — user registration, login, roles, session
4. `modules/crud.md` — generic maintainers (table, search, pagination, form)
5. `modules/layout.md` — admin panel, sidebar, navbar
6. `modules/database.md` — Prisma schema base, connection setup
7. `modules/files.md` — file and image upload, local storage
8. `modules/i18n.md` — multi-language with next-intl (es/en default)
9. `modules/notifications.md` — DB-based notifications with polling
10. `modules/email.md` — SMTP email, provider via .env, templates
11. `modules/design.md` — theme tokens, typography, animations, UI components

---

## What to build at project start

When told to "build the base", generate the following in order:

1. Run `create-next-app` with TypeScript + Tailwind + App Router + src/ directory
2. Install all packages from `stack.md`
3. Set up Prisma with PostgreSQL connection (use DATABASE_URL from .env)
4. Apply base schema from `modules/database.md`
5. Generate Auth.js setup from `modules/auth.md`
6. Generate admin layout from `modules/layout.md`
7. Generate first CRUD maintainer scaffold from `modules/crud.md`
8. Set up i18n from `modules/i18n.md` (es + en, Spanish default)
9. Set up notifications from `modules/notifications.md`
10. Set up file upload from `modules/files.md`
11. Copy `.github/workflows/deploy.yml` into project
12. Create `.env.example` with all required keys (no real values)

---

## Rules you must follow

- Never use API routes for form submissions — use Server Actions
- Never put secrets in code — use environment variables only
- Always validate data on the server, never trust client input
- Use Server Components by default, Client Components only when needed
- Every page that requires auth must check session server-side
- Never skip TypeScript types — no `any`
- Database access only through the Data Access Layer (`src/lib/dal.ts`)

---

## Environment variables required per project

```
DATABASE_URL=
NEXTAUTH_SECRET=
NEXTAUTH_URL=
ALWAYSDATA_HOST=
ALWAYSDATA_USER=
```

These must exist as GitHub Secrets before the first deploy.

---

## When adding a new module

1. Ask the user for the entity name and fields
2. Follow `modules/crud.md` exactly
3. Add the Prisma model to schema.prisma
4. Run `prisma migrate dev`
5. Generate: list page, form page, server actions, DAL function
