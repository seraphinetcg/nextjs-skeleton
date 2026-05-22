# modules/auth.md — Authentication Module

Generate this module exactly as described. No variations.

---

## What to generate

1. Auth.js v5 config (`src/lib/auth.ts`)
2. Prisma adapter models (add to schema.prisma)
3. Login page (`src/app/(auth)/login/page.tsx`)
4. Register page (`src/app/(auth)/register/page.tsx`)
5. Verify email page (`src/app/(auth)/verify-email/page.tsx`)
6. Forgot password page (`src/app/(auth)/forgot-password/page.tsx`)
7. Reset password page (`src/app/(auth)/reset-password/page.tsx`)
8. Auth API route (`src/app/api/auth/[...nextauth]/route.ts`)
9. Server Actions (`src/actions/auth.ts`)
10. Middleware for route protection (`middleware.ts`)

---

## Auth.js config

```typescript
// src/lib/auth.ts
import NextAuth from 'next-auth'
import { PrismaAdapter } from '@auth/prisma-adapter'
import Credentials from 'next-auth/providers/credentials'
import bcrypt from 'bcryptjs'
import { prisma } from '@/lib/prisma'
import { LoginSchema } from '@/lib/validations/auth'

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(prisma),
  session: { strategy: 'jwt' },
  pages: {
    signIn: '/login',
  },
  providers: [
    Credentials({
      async authorize(credentials) {
        const parsed = LoginSchema.safeParse(credentials)
        if (!parsed.success) return null
        const user = await prisma.user.findUnique({
          where: { email: parsed.data.email },
        })
        if (!user || !user.password) return null
        const valid = await bcrypt.compare(parsed.data.password, user.password)
        if (!valid) return null
        return user
      },
    }),
  ],
  callbacks: {
    jwt({ token, user }) {
      if (user) {
        token.id = user.id
        token.role = (user as any).role
      }
      return token
    },
    session({ session, token }) {
      session.user.id = token.id as string
      session.user.role = token.role as string
      return session
    },
  },
})
```

---

## Prisma schema additions

```prisma
model User {
  id            String    @id @default(cuid())
  name          String?
  email         String    @unique
  password      String?
  role          String    @default("user")
  emailVerified DateTime?
  image         String?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  accounts      Account[]
  sessions      Session[]
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
```

---

## Zod schemas

```typescript
// src/lib/validations/auth.ts
import { z } from 'zod'

export const LoginSchema = z.object({
  email: z.string().email('Email inválido'),
  password: z.string().min(6, 'Mínimo 6 caracteres'),
})

export const RegisterSchema = z.object({
  name: z.string().min(2, 'Mínimo 2 caracteres'),
  email: z.string().email('Email inválido'),
  password: z.string().min(6, 'Mínimo 6 caracteres'),
  confirmPassword: z.string(),
}).refine(data => data.password === data.confirmPassword, {
  message: 'Las contraseñas no coinciden',
  path: ['confirmPassword'],
})
```

---

## Register Server Action

```typescript
// src/actions/auth.ts
'use server'
import bcrypt from 'bcryptjs'
import { prisma } from '@/lib/prisma'
import { RegisterSchema } from '@/lib/validations/auth'
import { redirect } from 'next/navigation'

export async function registerAction(formData: FormData) {
  const parsed = RegisterSchema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return { error: parsed.error.flatten() }

  const existing = await prisma.user.findUnique({
    where: { email: parsed.data.email },
  })
  if (existing) return { error: { formErrors: ['El email ya está registrado'] } }

  const hashed = await bcrypt.hash(parsed.data.password, 10)
  await prisma.user.create({
    data: {
      name: parsed.data.name,
      email: parsed.data.email,
      password: hashed,
    },
  })
  redirect('/login')
}
```

---

## Middleware

```typescript
// middleware.ts
import { auth } from '@/lib/auth'
import { NextResponse } from 'next/server'

export default auth((req) => {
  const isAdminRoute = req.nextUrl.pathname.startsWith('/admin')
  if (isAdminRoute && !req.auth) {
    return NextResponse.redirect(new URL('/login', req.url))
  }
})

export const config = {
  matcher: ['/admin/:path*'],
}
```

---

## Roles

Default roles: `user`, `admin`

To check role in a server component:
```typescript
const session = await auth()
if (session?.user.role !== 'admin') redirect('/')
```

---

---

## Email Verification Flow

### Prisma model additions
```prisma
model VerificationToken {
  id         String   @id @default(cuid())
  email      String
  token      String   @unique
  type       String   @default("email_verification") // email_verification | password_reset
  expiresAt  DateTime
  createdAt  DateTime @default(now())
  @@unique([email, type])
}
```

Update User model — add field:
```prisma
emailVerified DateTime?
```

### Token utility
```typescript
// src/lib/tokens.ts
import { prisma } from '@/lib/prisma'
import crypto from 'crypto'

export async function generateToken(email: string, type: 'email_verification' | 'password_reset') {
  // delete any existing token of this type for this email
  await prisma.verificationToken.deleteMany({ where: { email, type } })

  const token = crypto.randomBytes(32).toString('hex')
  const expiresAt = new Date(Date.now() + 1000 * 60 * 60) // 1 hour

  await prisma.verificationToken.create({
    data: { email, token, type, expiresAt },
  })

  return token
}

export async function validateToken(token: string, type: 'email_verification' | 'password_reset') {
  const record = await prisma.verificationToken.findUnique({ where: { token } })

  if (!record) return { error: 'Token inválido' }
  if (record.type !== type) return { error: 'Token inválido' }
  if (record.expiresAt < new Date()) return { error: 'El token ha expirado' }

  return { success: true, email: record.email }
}

export async function deleteToken(token: string) {
  await prisma.verificationToken.deleteMany({ where: { token } })
}
```

---

## Updated Register Server Action (with email verification)

```typescript
// src/actions/auth.ts — registerAction
'use server'
import bcrypt from 'bcryptjs'
import { prisma } from '@/lib/prisma'
import { RegisterSchema } from '@/lib/validations/auth'
import { generateToken } from '@/lib/tokens'
import { sendEmail } from '@/lib/mailer'
import { verifyEmailTemplate } from '@/lib/email-templates/verify-email'
import { redirect } from 'next/navigation'

export async function registerAction(formData: FormData) {
  const parsed = RegisterSchema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return { error: parsed.error.flatten() }

  const existing = await prisma.user.findUnique({ where: { email: parsed.data.email } })
  if (existing) return { error: { formErrors: ['El email ya está registrado'] } }

  const hashed = await bcrypt.hash(parsed.data.password, 10)
  await prisma.user.create({
    data: {
      name: parsed.data.name,
      email: parsed.data.email,
      password: hashed,
    },
  })

  // generate verification token and send email
  const token = await generateToken(parsed.data.email, 'email_verification')
  const verifyUrl = `${process.env.NEXTAUTH_URL}/verify-email?token=${token}`

  const { subject, html } = verifyEmailTemplate({ name: parsed.data.name, verifyUrl })
  await sendEmail({ to: parsed.data.email, subject, html })

  redirect('/verify-email?sent=true')
}
```

---

## Verify Email Server Action

```typescript
// add to src/actions/auth.ts
import { validateToken, deleteToken } from '@/lib/tokens'

export async function verifyEmailAction(token: string) {
  const result = await validateToken(token, 'email_verification')
  if (result.error) return { error: result.error }

  await prisma.user.update({
    where: { email: result.email },
    data: { emailVerified: new Date() },
  })

  await deleteToken(token)
  redirect('/login?verified=true')
}
```

---

## Forgot Password Server Action

```typescript
// add to src/actions/auth.ts
import { forgotPasswordTemplate } from '@/lib/email-templates/forgot-password'

export async function forgotPasswordAction(formData: FormData) {
  const email = formData.get('email') as string
  if (!email) return { error: 'Email requerido' }

  const user = await prisma.user.findUnique({ where: { email } })

  // always return success — never reveal if email exists
  if (!user) return { success: true }

  const token = await generateToken(email, 'password_reset')
  const resetUrl = `${process.env.NEXTAUTH_URL}/reset-password?token=${token}`

  const { subject, html } = forgotPasswordTemplate({ name: user.name ?? email, resetUrl })
  await sendEmail({ to: email, subject, html })

  return { success: true }
}
```

---

## Reset Password Server Action

```typescript
// add to src/actions/auth.ts
import { ResetPasswordSchema } from '@/lib/validations/auth'

export async function resetPasswordAction(token: string, formData: FormData) {
  const parsed = ResetPasswordSchema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return { error: parsed.error.flatten() }

  const result = await validateToken(token, 'password_reset')
  if (result.error) return { error: result.error }

  const hashed = await bcrypt.hash(parsed.data.password, 10)

  await prisma.user.update({
    where: { email: result.email },
    data: { password: hashed },
  })

  await deleteToken(token)
  redirect('/login?reset=true')
}
```

---

## Updated Zod schemas

```typescript
// add to src/lib/validations/auth.ts

export const ForgotPasswordSchema = z.object({
  email: z.string().email('Email inválido'),
})

export const ResetPasswordSchema = z.object({
  password: z.string().min(6, 'Mínimo 6 caracteres'),
  confirmPassword: z.string(),
}).refine(data => data.password === data.confirmPassword, {
  message: 'Las contraseñas no coinciden',
  path: ['confirmPassword'],
})
```

---

## Email templates for auth flows

```typescript
// src/lib/email-templates/verify-email.ts
export function verifyEmailTemplate({ name, verifyUrl }: { name: string; verifyUrl: string }) {
  return {
    subject: 'Verifica tu correo electrónico',
    html: `
      <div style="font-family:sans-serif;max-width:600px;margin:0 auto;padding:24px">
        <h1 style="color:#111">Verifica tu correo</h1>
        <p style="color:#555">Hola ${name}, haz clic en el botón para verificar tu cuenta.</p>
        <a href="${verifyUrl}"
          style="display:inline-block;background:#000;color:#fff;padding:12px 24px;border-radius:6px;text-decoration:none;margin-top:16px">
          Verificar correo
        </a>
        <p style="color:#999;font-size:12px;margin-top:32px">
          Este enlace expira en 1 hora. Si no creaste esta cuenta, ignora este mensaje.
        </p>
      </div>
    `,
  }
}
```

```typescript
// src/lib/email-templates/forgot-password.ts
export function forgotPasswordTemplate({ name, resetUrl }: { name: string; resetUrl: string }) {
  return {
    subject: 'Restablecer tu contraseña',
    html: `
      <div style="font-family:sans-serif;max-width:600px;margin:0 auto;padding:24px">
        <h1 style="color:#111">¿Olvidaste tu contraseña?</h1>
        <p style="color:#555">Hola ${name}, haz clic en el botón para crear una nueva contraseña.</p>
        <a href="${resetUrl}"
          style="display:inline-block;background:#000;color:#fff;padding:12px 24px;border-radius:6px;text-decoration:none;margin-top:16px">
          Restablecer contraseña
        </a>
        <p style="color:#999;font-size:12px;margin-top:32px">
          Este enlace expira en 1 hora. Si no solicitaste esto, ignora este mensaje.
        </p>
      </div>
    `,
  }
}
```

---

## Auth pages behavior

### `/verify-email?sent=true`
- Muestra mensaje: "Revisa tu correo, te enviamos un enlace de verificación"
- Si tiene `?token=` en la URL: llama `verifyEmailAction(token)` automáticamente
- Si el token es inválido: muestra error con botón para reenviar

### `/forgot-password`
- Formulario con solo campo email
- Al enviar: muestra "Si el correo existe, recibirás un enlace en minutos"
- Nunca revelar si el email existe o no

### `/reset-password?token=XXX`
- Valida el token al cargar la página (server-side)
- Si inválido/expirado: muestra error con link a `/forgot-password`
- Si válido: muestra formulario con nueva contraseña + confirmar

### `/login` feedback messages
- `?verified=true` → mostrar banner verde: "Email verificado, ya puedes iniciar sesión"
- `?reset=true` → mostrar banner verde: "Contraseña actualizada correctamente"

---

## Login guard — block unverified users

```typescript
// in Auth.js authorize callback
if (!user.emailVerified) {
  throw new Error('EMAIL_NOT_VERIFIED')
}
```

Handle the error in the login page to show: "Debes verificar tu correo antes de iniciar sesión".

---

## Login and Register pages

Generate clean forms using shadcn/ui components:
- `Card`, `CardContent`, `CardHeader`
- `Input`, `Label`, `Button`
- Show field errors inline below each input
- Show form-level errors at the top of the card
- Login page links to register, register page links to login
