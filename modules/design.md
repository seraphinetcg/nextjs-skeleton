# modules/design.md — Design System

Neutral/Corporate aesthetic. Adapts to any client brand by changing CSS variables only.
Covers: theme tokens, dark mode, typography, animations, and extra UI components.

---

## What to generate

1. CSS variables theme → `src/app/globals.css`
2. Tailwind config → `tailwind.config.ts`
3. Font setup → `src/app/layout.tsx`
4. Framer Motion page transitions → `src/components/layout/PageTransition.tsx`
5. Animation variants → `src/lib/animations.ts`
6. Extra components:
   - `src/components/ui/StatCard.tsx`
   - `src/components/ui/EmptyState.tsx`
   - `src/components/ui/Skeleton.tsx`
   - `src/components/ui/Badge.tsx`
   - `src/components/ui/Avatar.tsx`
   - `src/components/ui/PageHeader.tsx`

---

## Install

```bash
npm install framer-motion
npm install @next/font
```

---

## Fonts — Inter + Geist (professional, modern, free)

```typescript
// src/app/layout.tsx
import { Inter, Geist_Mono } from 'next/font/google'
import { cn } from '@/lib/utils'

const inter = Inter({
  subsets: ['latin'],
  variable: '--font-sans',
  display: 'swap',
})

const geistMono = Geist_Mono({
  subsets: ['latin'],
  variable: '--font-mono',
  display: 'swap',
})

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="es" suppressHydrationWarning>
      <body className={cn(inter.variable, geistMono.variable, 'font-sans antialiased')}>
        {children}
      </body>
    </html>
  )
}
```

---

## CSS variables — neutral corporate theme

```css
/* src/app/globals.css */
@import "tailwindcss";

@layer base {
  :root {
    /* Background */
    --background: 0 0% 100%;
    --foreground: 222 47% 11%;

    /* Cards and surfaces */
    --card: 0 0% 100%;
    --card-foreground: 222 47% 11%;
    --popover: 0 0% 100%;
    --popover-foreground: 222 47% 11%;

    /* Brand — change these two per client */
    --primary: 222 47% 11%;
    --primary-foreground: 210 40% 98%;

    /* Secondary / muted */
    --secondary: 210 40% 96%;
    --secondary-foreground: 222 47% 11%;
    --muted: 210 40% 96%;
    --muted-foreground: 215 16% 47%;

    /* Accent */
    --accent: 210 40% 94%;
    --accent-foreground: 222 47% 11%;

    /* Status */
    --destructive: 0 84% 60%;
    --destructive-foreground: 210 40% 98%;
    --success: 142 76% 36%;
    --success-foreground: 0 0% 100%;
    --warning: 38 92% 50%;
    --warning-foreground: 0 0% 100%;

    /* Structure */
    --border: 214 32% 91%;
    --input: 214 32% 91%;
    --ring: 222 47% 11%;
    --radius: 0.5rem;

    /* Sidebar */
    --sidebar: 222 47% 11%;
    --sidebar-foreground: 210 40% 98%;
    --sidebar-muted: 215 25% 27%;
    --sidebar-muted-foreground: 215 16% 65%;
    --sidebar-accent: 215 25% 20%;
    --sidebar-border: 215 25% 20%;
  }

  .dark {
    --background: 222 47% 6%;
    --foreground: 210 40% 98%;
    --card: 222 47% 9%;
    --card-foreground: 210 40% 98%;
    --popover: 222 47% 9%;
    --popover-foreground: 210 40% 98%;
    --primary: 210 40% 98%;
    --primary-foreground: 222 47% 11%;
    --secondary: 217 33% 17%;
    --secondary-foreground: 210 40% 98%;
    --muted: 217 33% 17%;
    --muted-foreground: 215 20% 65%;
    --accent: 217 33% 17%;
    --accent-foreground: 210 40% 98%;
    --destructive: 0 62% 30%;
    --destructive-foreground: 210 40% 98%;
    --border: 217 33% 17%;
    --input: 217 33% 17%;
    --ring: 212 26% 83%;
    --sidebar: 222 47% 6%;
    --sidebar-foreground: 210 40% 98%;
    --sidebar-muted: 217 33% 12%;
    --sidebar-muted-foreground: 215 20% 55%;
    --sidebar-accent: 217 33% 10%;
    --sidebar-border: 217 33% 12%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
  h1 { @apply text-3xl font-bold tracking-tight; }
  h2 { @apply text-2xl font-semibold tracking-tight; }
  h3 { @apply text-xl font-semibold; }
  h4 { @apply text-lg font-medium; }
}
```

---

## Changing brand per client

Only two CSS variables need to change per client.
Find complementary values at: https://www.tints.dev

```css
/* Example: blue corporate client */
--primary: 217 91% 60%;
--primary-foreground: 0 0% 100%;

/* Example: green client */
--primary: 142 76% 36%;
--primary-foreground: 0 0% 100%;
```

---

## tailwind.config.ts

```typescript
import type { Config } from 'tailwindcss'

const config: Config = {
  darkMode: 'class',
  content: ['./src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      fontFamily: {
        sans: ['var(--font-sans)', 'system-ui', 'sans-serif'],
        mono: ['var(--font-mono)', 'monospace'],
      },
      colors: {
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        card: {
          DEFAULT: 'hsl(var(--card))',
          foreground: 'hsl(var(--card-foreground))',
        },
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))',
        },
        secondary: {
          DEFAULT: 'hsl(var(--secondary))',
          foreground: 'hsl(var(--secondary-foreground))',
        },
        muted: {
          DEFAULT: 'hsl(var(--muted))',
          foreground: 'hsl(var(--muted-foreground))',
        },
        accent: {
          DEFAULT: 'hsl(var(--accent))',
          foreground: 'hsl(var(--accent-foreground))',
        },
        destructive: {
          DEFAULT: 'hsl(var(--destructive))',
          foreground: 'hsl(var(--destructive-foreground))',
        },
        success: {
          DEFAULT: 'hsl(var(--success))',
          foreground: 'hsl(var(--success-foreground))',
        },
        warning: {
          DEFAULT: 'hsl(var(--warning))',
          foreground: 'hsl(var(--warning-foreground))',
        },
        border: 'hsl(var(--border))',
        input: 'hsl(var(--input))',
        ring: 'hsl(var(--ring))',
        sidebar: {
          DEFAULT: 'hsl(var(--sidebar))',
          foreground: 'hsl(var(--sidebar-foreground))',
          muted: 'hsl(var(--sidebar-muted))',
          'muted-foreground': 'hsl(var(--sidebar-muted-foreground))',
          accent: 'hsl(var(--sidebar-accent))',
          border: 'hsl(var(--sidebar-border))',
        },
      },
      borderRadius: {
        lg: 'var(--radius)',
        md: 'calc(var(--radius) - 2px)',
        sm: 'calc(var(--radius) - 4px)',
      },
      keyframes: {
        'fade-in': {
          from: { opacity: '0', transform: 'translateY(8px)' },
          to: { opacity: '1', transform: 'translateY(0)' },
        },
        'slide-in': {
          from: { transform: 'translateX(-100%)' },
          to: { transform: 'translateX(0)' },
        },
        shimmer: {
          '0%': { backgroundPosition: '-200% 0' },
          '100%': { backgroundPosition: '200% 0' },
        },
      },
      animation: {
        'fade-in': 'fade-in 0.3s ease-out',
        'slide-in': 'slide-in 0.2s ease-out',
        shimmer: 'shimmer 1.5s infinite linear',
      },
    },
  },
  plugins: [],
}

export default config
```

---

## Animation variants (Framer Motion)

```typescript
// src/lib/animations.ts
export const fadeIn = {
  initial: { opacity: 0, y: 8 },
  animate: { opacity: 1, y: 0 },
  exit: { opacity: 0, y: -8 },
  transition: { duration: 0.2, ease: 'easeOut' },
}

export const slideInLeft = {
  initial: { opacity: 0, x: -16 },
  animate: { opacity: 1, x: 0 },
  exit: { opacity: 0, x: -16 },
  transition: { duration: 0.2, ease: 'easeOut' },
}

export const staggerContainer = {
  animate: {
    transition: {
      staggerChildren: 0.07,
    },
  },
}

export const staggerItem = {
  initial: { opacity: 0, y: 12 },
  animate: { opacity: 1, y: 0 },
  transition: { duration: 0.2, ease: 'easeOut' },
}

export const scaleIn = {
  initial: { opacity: 0, scale: 0.95 },
  animate: { opacity: 1, scale: 1 },
  exit: { opacity: 0, scale: 0.95 },
  transition: { duration: 0.15, ease: 'easeOut' },
}
```

---

## Page transition wrapper

```typescript
// src/components/layout/PageTransition.tsx
'use client'
import { motion } from 'framer-motion'
import { fadeIn } from '@/lib/animations'

export function PageTransition({ children }: { children: React.ReactNode }) {
  return (
    <motion.div {...fadeIn}>
      {children}
    </motion.div>
  )
}
```

Wrap every admin page content:
```typescript
export default async function SomePage() {
  return (
    <PageTransition>
      <div className="space-y-6">
        ...
      </div>
    </PageTransition>
  )
}
```

---

## StatCard component

```typescript
// src/components/ui/StatCard.tsx
'use client'
import { motion } from 'framer-motion'
import { Card, CardContent } from '@/components/ui/card'
import { staggerItem } from '@/lib/animations'
import { cn } from '@/lib/utils'
import type { LucideIcon } from 'lucide-react'

interface StatCardProps {
  title: string
  value: string | number
  description?: string
  icon: LucideIcon
  trend?: { value: number; label: string }
  variant?: 'default' | 'primary'
}

export function StatCard({
  title,
  value,
  description,
  icon: Icon,
  trend,
  variant = 'default',
}: StatCardProps) {
  return (
    <motion.div {...staggerItem}>
      <Card className={cn(
        'relative overflow-hidden',
        variant === 'primary' && 'bg-primary text-primary-foreground'
      )}>
        <CardContent className="p-6">
          <div className="flex items-center justify-between">
            <div className="space-y-1">
              <p className={cn(
                'text-sm font-medium',
                variant === 'primary' ? 'text-primary-foreground/70' : 'text-muted-foreground'
              )}>
                {title}
              </p>
              <p className="text-3xl font-bold tracking-tight">{value}</p>
              {description && (
                <p className={cn(
                  'text-xs',
                  variant === 'primary' ? 'text-primary-foreground/60' : 'text-muted-foreground'
                )}>
                  {description}
                </p>
              )}
              {trend && (
                <p className={cn(
                  'text-xs font-medium',
                  trend.value >= 0 ? 'text-success' : 'text-destructive'
                )}>
                  {trend.value >= 0 ? '↑' : '↓'} {Math.abs(trend.value)}% {trend.label}
                </p>
              )}
            </div>
            <div className={cn(
              'rounded-xl p-3',
              variant === 'primary' ? 'bg-primary-foreground/10' : 'bg-muted'
            )}>
              <Icon className="h-6 w-6" />
            </div>
          </div>
        </CardContent>
      </Card>
    </motion.div>
  )
}
```

Usage on Dashboard:
```typescript
import { motion } from 'framer-motion'
import { staggerContainer } from '@/lib/animations'
import { StatCard } from '@/components/ui/StatCard'
import { Users, ShoppingBag, TrendingUp, Activity } from 'lucide-react'

<motion.div
  className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4"
  variants={staggerContainer}
  initial="initial"
  animate="animate"
>
  <StatCard title="Usuarios" value={124} icon={Users} trend={{ value: 12, label: 'este mes' }} />
  <StatCard title="Pedidos" value={38} icon={ShoppingBag} description="Últimos 30 días" />
  <StatCard title="Ingresos" value="$4,200" icon={TrendingUp} variant="primary" />
  <StatCard title="Activos" value={97} icon={Activity} trend={{ value: -3, label: 'vs ayer' }} />
</motion.div>
```

---

## EmptyState component

```typescript
// src/components/ui/EmptyState.tsx
import { cn } from '@/lib/utils'
import type { LucideIcon } from 'lucide-react'
import { Button } from '@/components/ui/button'
import Link from 'next/link'

interface EmptyStateProps {
  icon: LucideIcon
  title: string
  description: string
  action?: { label: string; href: string }
  className?: string
}

export function EmptyState({ icon: Icon, title, description, action, className }: EmptyStateProps) {
  return (
    <div className={cn(
      'flex flex-col items-center justify-center py-16 px-4 text-center',
      className
    )}>
      <div className="rounded-full bg-muted p-4 mb-4">
        <Icon className="h-8 w-8 text-muted-foreground" />
      </div>
      <h3 className="font-semibold text-lg mb-1">{title}</h3>
      <p className="text-muted-foreground text-sm max-w-sm mb-6">{description}</p>
      {action && (
        <Button asChild>
          <Link href={action.href}>{action.label}</Link>
        </Button>
      )}
    </div>
  )
}
```

---

## Skeleton component

```typescript
// src/components/ui/Skeleton.tsx
import { cn } from '@/lib/utils'

export function Skeleton({ className }: { className?: string }) {
  return (
    <div className={cn(
      'rounded-md bg-muted animate-shimmer bg-gradient-to-r from-muted via-muted/50 to-muted bg-[length:200%_100%]',
      className
    )} />
  )
}

export function TableSkeleton({ rows = 5 }: { rows?: number }) {
  return (
    <div className="space-y-3">
      <Skeleton className="h-10 w-full" />
      {Array.from({ length: rows }).map((_, i) => (
        <Skeleton key={i} className="h-14 w-full" />
      ))}
    </div>
  )
}

export function CardSkeleton() {
  return (
    <div className="rounded-xl border p-6 space-y-3">
      <Skeleton className="h-4 w-24" />
      <Skeleton className="h-8 w-16" />
      <Skeleton className="h-3 w-32" />
    </div>
  )
}
```

---

## Badge component

```typescript
// src/components/ui/Badge.tsx
import { cn } from '@/lib/utils'

type BadgeVariant = 'default' | 'success' | 'warning' | 'error' | 'info' | 'outline'

interface BadgeProps {
  children: React.ReactNode
  variant?: BadgeVariant
  className?: string
}

const variants: Record<BadgeVariant, string> = {
  default: 'bg-secondary text-secondary-foreground',
  success: 'bg-success/10 text-success border border-success/20',
  warning: 'bg-warning/10 text-warning border border-warning/20',
  error: 'bg-destructive/10 text-destructive border border-destructive/20',
  info: 'bg-blue-50 text-blue-700 border border-blue-200',
  outline: 'border border-border text-foreground',
}

export function Badge({ children, variant = 'default', className }: BadgeProps) {
  return (
    <span className={cn(
      'inline-flex items-center rounded-full px-2.5 py-0.5 text-xs font-medium',
      variants[variant],
      className
    )}>
      {children}
    </span>
  )
}
```

---

## Avatar component

```typescript
// src/components/ui/Avatar.tsx
import { cn } from '@/lib/utils'
import Image from 'next/image'

interface AvatarProps {
  src?: string | null
  name?: string | null
  size?: 'sm' | 'md' | 'lg'
  className?: string
}

const sizes = { sm: 'h-7 w-7 text-xs', md: 'h-9 w-9 text-sm', lg: 'h-12 w-12 text-base' }
const imgSizes = { sm: 28, md: 36, lg: 48 }

function getInitials(name?: string | null) {
  if (!name) return '?'
  return name.split(' ').map(n => n[0]).slice(0, 2).join('').toUpperCase()
}

export function Avatar({ src, name, size = 'md', className }: AvatarProps) {
  return (
    <div className={cn(
      'rounded-full bg-muted flex items-center justify-center font-medium text-muted-foreground overflow-hidden shrink-0',
      sizes[size],
      className
    )}>
      {src ? (
        <Image
          src={src}
          alt={name ?? 'avatar'}
          width={imgSizes[size]}
          height={imgSizes[size]}
          className="object-cover w-full h-full"
        />
      ) : (
        getInitials(name)
      )}
    </div>
  )
}
```

---

## PageHeader component

```typescript
// src/components/ui/PageHeader.tsx
import { cn } from '@/lib/utils'

interface PageHeaderProps {
  title: string
  description?: string
  children?: React.ReactNode  // for action buttons
  className?: string
}

export function PageHeader({ title, description, children, className }: PageHeaderProps) {
  return (
    <div className={cn('flex items-start justify-between gap-4', className)}>
      <div>
        <h1 className="text-2xl font-bold tracking-tight">{title}</h1>
        {description && (
          <p className="text-muted-foreground mt-1 text-sm">{description}</p>
        )}
      </div>
      {children && (
        <div className="flex items-center gap-2 shrink-0">
          {children}
        </div>
      )}
    </div>
  )
}
```

Usage on any list page:
```typescript
<PageHeader
  title="Productos"
  description="Gestiona el catálogo de productos"
>
  <Button asChild>
    <Link href="/admin/products/new">Nuevo producto</Link>
  </Button>
</PageHeader>
```

---

## Dark mode toggle

```typescript
// src/components/ui/ThemeToggle.tsx
'use client'
import { Moon, Sun } from 'lucide-react'
import { Button } from '@/components/ui/button'
import { useEffect, useState } from 'react'

export function ThemeToggle() {
  const [dark, setDark] = useState(false)

  useEffect(() => {
    const saved = localStorage.getItem('theme')
    const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches
    const isDark = saved === 'dark' || (!saved && prefersDark)
    setDark(isDark)
    document.documentElement.classList.toggle('dark', isDark)
  }, [])

  function toggle() {
    const next = !dark
    setDark(next)
    document.documentElement.classList.toggle('dark', next)
    localStorage.setItem('theme', next ? 'dark' : 'light')
  }

  return (
    <Button variant="ghost" size="icon" onClick={toggle}>
      {dark ? <Sun className="h-4 w-4" /> : <Moon className="h-4 w-4" />}
    </Button>
  )
}
```

Add `<ThemeToggle />` to `Navbar.tsx`.

---

## shadcn/ui components to install

Run these after project creation:

```bash
npx shadcn@latest init
npx shadcn@latest add button card input label table dropdown-menu dialog alert-dialog sheet tabs badge avatar skeleton separator
```
