# stack.md — Fixed Stack Decisions

These versions and libraries are non-negotiable for all client projects.
Do not substitute, upgrade, or add alternatives without updating this file first.

---

## Core

| Package | Version | Notes |
|---|---|---|
| next | ^16.2.0 | App Router only, no Pages Router |
| react | ^19.0.0 | |
| react-dom | ^19.0.0 | |
| typescript | ^5.x | strict mode enabled |

---

## Styling

| Package | Version | Notes |
|---|---|---|
| tailwindcss | ^4.x | |
| @shadcn/ui | latest | install components via CLI |
| lucide-react | latest | icons only from this package |

---

## Auth

| Package | Version | Notes |
|---|---|---|
| next-auth | ^5.x (Auth.js) | credentials + optional OAuth |
| bcryptjs | ^2.x | password hashing |
| @types/bcryptjs | latest | |

---

## Database

| Package | Version | Notes |
|---|---|---|
| prisma | ^6.x | |
| @prisma/client | ^6.x | |

---

## Forms & Validation

| Package | Version | Notes |
|---|---|---|
| zod | ^3.x | all schema validation |
| react-hook-form | ^7.x | client-side forms only |
| @hookform/resolvers | latest | zod resolver |

---

## i18n

| Package | Version | Notes |
|---|---|---|
| next-intl | ^3.x | multi-language, server-side only |

## Design & Animations

| Package | Version | Notes |
|---|---|---|
| framer-motion | ^11.x | page transitions, micro-interactions |

## Email

| Package | Version | Notes |
|---|---|---|
| nodemailer | ^6.x | SMTP, provider set via .env |
| @types/nodemailer | latest | dev dependency |

## Files & Images

| Package | Version | Notes |
|---|---|---|
| (built-in) | — | Node.js fs/promises for local storage |

## Utilities

| Package | Version | Notes |
|---|---|---|
| clsx | latest | conditional classnames |
| tailwind-merge | latest | merge tailwind classes |
| date-fns | ^3.x | date formatting |

---

## Install command

```bash
npm install next-auth@beta bcryptjs prisma @prisma/client zod react-hook-form @hookform/resolvers clsx tailwind-merge date-fns lucide-react next-intl nodemailer framer-motion

npm install -D @types/bcryptjs @types/nodemailer

npm install -D @types/bcryptjs prisma
```

---

## tsconfig.json — required settings

```json
{
  "compilerOptions": {
    "strict": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

---

## Project structure created by create-next-app

```
npx create-next-app@latest . \
  --typescript \
  --tailwind \
  --app \
  --src-dir \
  --import-alias "@/*" \
  --no-eslint
```
