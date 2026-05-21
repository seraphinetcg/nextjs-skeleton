# modules/email.md — Email Module

Compatible with AlwaysData. Uses Nodemailer via SMTP — works with any provider.
Provider is configured entirely through .env — no code changes needed to switch providers.

---

## What to generate

1. Email client config → `src/lib/email.ts`
2. Email templates → `src/lib/email-templates/`
3. Utility functions → `src/lib/mailer.ts`
4. Integration with notifications module (optional)

---

## Install

```bash
npm install nodemailer
npm install -D @types/nodemailer
```

---

## Environment variables

Add these to `.env` and GitHub Secrets:

```env
# SMTP Provider (choose one — see provider configs below)
SMTP_HOST=
SMTP_PORT=
SMTP_SECURE=         # true for port 465, false for 587
SMTP_USER=
SMTP_PASS=

# Sender identity
EMAIL_FROM_NAME=     # e.g. "Mi Empresa"
EMAIL_FROM_ADDRESS=  # e.g. noreply@miempresa.com
```

---

## Provider configurations

Copy the values that match your provider into .env:

### Gmail (personal/testing)
```env
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=tucorreo@gmail.com
SMTP_PASS=tu-app-password   # generate at myaccount.google.com/apppasswords
```

### Resend (recommended for production)
```env
SMTP_HOST=smtp.resend.com
SMTP_PORT=465
SMTP_SECURE=true
SMTP_USER=resend
SMTP_PASS=re_xxxxxxxxxxxx   # API key from resend.com
```

### Brevo (formerly Sendinblue — free tier 300/day)
```env
SMTP_HOST=smtp-relay.brevo.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=tucorreo@brevo.com
SMTP_PASS=xsmtpsib-xxxxxxxxxxxx
```

### Mailgun
```env
SMTP_HOST=smtp.mailgun.org
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=postmaster@mg.tudominio.com
SMTP_PASS=tu-mailgun-password
```

### AlwaysData built-in SMTP
```env
SMTP_HOST=mail.alwaysdata.net
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=tu-usuario@tudominio.com
SMTP_PASS=tu-password-alwaysdata
```

---

## Email client config

```typescript
// src/lib/email.ts
import nodemailer from 'nodemailer'

if (!process.env.SMTP_HOST) throw new Error('SMTP_HOST is not set')
if (!process.env.SMTP_USER) throw new Error('SMTP_USER is not set')
if (!process.env.SMTP_PASS) throw new Error('SMTP_PASS is not set')

export const transporter = nodemailer.createTransport({
  host: process.env.SMTP_HOST,
  port: Number(process.env.SMTP_PORT ?? 587),
  secure: process.env.SMTP_SECURE === 'true',
  auth: {
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS,
  },
})

export const emailFrom = `"${process.env.EMAIL_FROM_NAME}" <${process.env.EMAIL_FROM_ADDRESS}>`
```

---

## Mailer utility

```typescript
// src/lib/mailer.ts
import { transporter, emailFrom } from '@/lib/email'

interface SendEmailParams {
  to: string
  subject: string
  html: string
  text?: string
}

export async function sendEmail({ to, subject, html, text }: SendEmailParams) {
  try {
    const info = await transporter.sendMail({
      from: emailFrom,
      to,
      subject,
      html,
      text: text ?? html.replace(/<[^>]+>/g, ''),
    })
    return { success: true, messageId: info.messageId }
  } catch (error) {
    console.error('Email send error:', error)
    return { success: false, error: 'No se pudo enviar el email' }
  }
}
```

---

## Email templates

```typescript
// src/lib/email-templates/welcome.ts
export function welcomeTemplate({
  name,
  loginUrl,
}: {
  name: string
  loginUrl: string
}) {
  return {
    subject: `Bienvenido, ${name}`,
    html: `
      <div style="font-family:sans-serif;max-width:600px;margin:0 auto;padding:24px">
        <h1 style="color:#111">Bienvenido, ${name}</h1>
        <p style="color:#555">Tu cuenta ha sido creada correctamente.</p>
        <a
          href="${loginUrl}"
          style="display:inline-block;background:#000;color:#fff;padding:12px 24px;border-radius:6px;text-decoration:none;margin-top:16px"
        >
          Iniciar sesión
        </a>
        <p style="color:#999;font-size:12px;margin-top:32px">
          Si no creaste esta cuenta, ignora este mensaje.
        </p>
      </div>
    `,
  }
}
```

```typescript
// src/lib/email-templates/reset-password.ts
export function resetPasswordTemplate({
  name,
  resetUrl,
}: {
  name: string
  resetUrl: string
}) {
  return {
    subject: 'Restablecer contraseña',
    html: `
      <div style="font-family:sans-serif;max-width:600px;margin:0 auto;padding:24px">
        <h1 style="color:#111">Restablecer contraseña</h1>
        <p style="color:#555">Hola ${name}, recibimos una solicitud para restablecer tu contraseña.</p>
        <a
          href="${resetUrl}"
          style="display:inline-block;background:#000;color:#fff;padding:12px 24px;border-radius:6px;text-decoration:none;margin-top:16px"
        >
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

```typescript
// src/lib/email-templates/notification.ts
export function notificationTemplate({
  title,
  message,
  actionUrl,
  actionLabel = 'Ver detalle',
}: {
  title: string
  message: string
  actionUrl?: string
  actionLabel?: string
}) {
  return {
    subject: title,
    html: `
      <div style="font-family:sans-serif;max-width:600px;margin:0 auto;padding:24px">
        <h2 style="color:#111">${title}</h2>
        <p style="color:#555">${message}</p>
        ${actionUrl ? `
          <a
            href="${actionUrl}"
            style="display:inline-block;background:#000;color:#fff;padding:12px 24px;border-radius:6px;text-decoration:none;margin-top:16px"
          >
            ${actionLabel}
          </a>
        ` : ''}
        <p style="color:#999;font-size:12px;margin-top:32px">
          Este es un mensaje automático, no respondas este correo.
        </p>
      </div>
    `,
  }
}
```

---

## Usage in Server Actions

### Send welcome email on register
```typescript
// add to src/actions/auth.ts after creating the user
import { sendEmail } from '@/lib/mailer'
import { welcomeTemplate } from '@/lib/email-templates/welcome'

const { subject, html } = welcomeTemplate({
  name: parsed.data.name,
  loginUrl: `${process.env.NEXTAUTH_URL}/login`,
})

await sendEmail({ to: parsed.data.email, subject, html })
```

### Send email + in-app notification together
```typescript
import { sendEmail } from '@/lib/mailer'
import { notify } from '@/lib/notify'
import { notificationTemplate } from '@/lib/email-templates/notification'

const title = 'Pedido aprobado'
const message = 'Tu pedido #1234 fue aprobado y está en camino'

// in-app notification
await notify({ userId, title, message, type: 'success', link: '/orders/1234' })

// email notification
const { subject, html } = notificationTemplate({ title, message, actionUrl: `${process.env.NEXTAUTH_URL}/orders/1234` })
await sendEmail({ to: userEmail, subject, html })
```

---

## Folder structure for templates

```
src/lib/email-templates/
├── welcome.ts
├── reset-password.ts
└── notification.ts
```

Add a new file per template — one function per file.
Templates return `{ subject, html }` — always this shape.

---

## .env.example additions

```env
SMTP_HOST=
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=
SMTP_PASS=
EMAIL_FROM_NAME=
EMAIL_FROM_ADDRESS=
```

---

## GitHub Secrets to add

| Secret | Value |
|---|---|
| `SMTP_HOST` | SMTP server host |
| `SMTP_PORT` | 587 or 465 |
| `SMTP_SECURE` | true or false |
| `SMTP_USER` | SMTP username |
| `SMTP_PASS` | SMTP password or API key |
| `EMAIL_FROM_NAME` | Sender display name |
| `EMAIL_FROM_ADDRESS` | Sender email address |

---

## AlwaysData notes

- AlwaysData includes SMTP on paid plans — use the built-in config above to avoid external dependencies
- For better deliverability (avoid spam), use Resend or Brevo with your own domain
- Never send emails from `localhost` — always test with a real SMTP provider
