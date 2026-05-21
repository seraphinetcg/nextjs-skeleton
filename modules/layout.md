# modules/layout.md — Admin Panel Layout

Generate this layout as the shell for all admin pages.

---

## What to generate

1. Admin layout → `src/app/(admin)/layout.tsx`
2. AdminShell component → `src/components/layout/AdminShell.tsx`
3. Sidebar component → `src/components/layout/Sidebar.tsx`
4. Navbar component → `src/components/layout/Navbar.tsx`
5. Dashboard page → `src/app/(admin)/dashboard/page.tsx`

---

## Admin layout

```typescript
// src/app/(admin)/layout.tsx
import { auth } from '@/lib/auth'
import { redirect } from 'next/navigation'
import { AdminShell } from '@/components/layout/AdminShell'

export default async function AdminLayout({
  children,
}: {
  children: React.ReactNode
}) {
  const session = await auth()
  if (!session) redirect('/login')

  return <AdminShell user={session.user}>{children}</AdminShell>
}
```

---

## AdminShell component

```typescript
// src/components/layout/AdminShell.tsx
'use client'
import { useState } from 'react'
import { Sidebar } from './Sidebar'
import { Navbar } from './Navbar'

interface AdminShellProps {
  children: React.ReactNode
  user: { name?: string | null; email?: string | null; role?: string }
}

export function AdminShell({ children, user }: AdminShellProps) {
  const [sidebarOpen, setSidebarOpen] = useState(true)

  return (
    <div className="flex h-screen bg-gray-50">
      <Sidebar open={sidebarOpen} />
      <div className="flex flex-col flex-1 overflow-hidden">
        <Navbar
          user={user}
          onToggleSidebar={() => setSidebarOpen(!sidebarOpen)}
        />
        <main className="flex-1 overflow-y-auto p-6">
          {children}
        </main>
      </div>
    </div>
  )
}
```

---

## Sidebar component

```typescript
// src/components/layout/Sidebar.tsx
'use client'
import Link from 'next/link'
import { usePathname } from 'next/navigation'
import { cn } from '@/lib/utils'
import {
  LayoutDashboard,
  Users,
  Settings,
  // add module icons here
} from 'lucide-react'

const navItems = [
  { label: 'Dashboard', href: '/admin/dashboard', icon: LayoutDashboard },
  { label: 'Usuarios', href: '/admin/users', icon: Users },
  { label: 'Configuración', href: '/admin/settings', icon: Settings },
  // add module links here
]

export function Sidebar({ open }: { open: boolean }) {
  const pathname = usePathname()

  if (!open) return null

  return (
    <aside className="w-64 bg-white border-r flex flex-col">
      <div className="h-16 flex items-center px-6 border-b">
        <span className="font-bold text-lg">Admin</span>
      </div>
      <nav className="flex-1 p-4 space-y-1">
        {navItems.map((item) => (
          <Link
            key={item.href}
            href={item.href}
            className={cn(
              'flex items-center gap-3 px-3 py-2 rounded-md text-sm transition-colors',
              pathname === item.href
                ? 'bg-gray-100 font-medium text-gray-900'
                : 'text-gray-600 hover:bg-gray-50 hover:text-gray-900'
            )}
          >
            <item.icon className="h-4 w-4" />
            {item.label}
          </Link>
        ))}
      </nav>
    </aside>
  )
}
```

---

## Navbar component

```typescript
// src/components/layout/Navbar.tsx
'use client'
import { signOut } from 'next-auth/react'
import { Menu, LogOut, User } from 'lucide-react'
import { Button } from '@/components/ui/button'
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu'

interface NavbarProps {
  user: { name?: string | null; email?: string | null }
  onToggleSidebar: () => void
}

export function Navbar({ user, onToggleSidebar }: NavbarProps) {
  return (
    <header className="h-16 bg-white border-b flex items-center justify-between px-6">
      <Button variant="ghost" size="icon" onClick={onToggleSidebar}>
        <Menu className="h-5 w-5" />
      </Button>

      <DropdownMenu>
        <DropdownMenuTrigger asChild>
          <Button variant="ghost" className="flex items-center gap-2">
            <User className="h-4 w-4" />
            <span className="text-sm">{user.name ?? user.email}</span>
          </Button>
        </DropdownMenuTrigger>
        <DropdownMenuContent align="end">
          <DropdownMenuItem onClick={() => signOut({ callbackUrl: '/login' })}>
            <LogOut className="h-4 w-4 mr-2" />
            Cerrar sesión
          </DropdownMenuItem>
        </DropdownMenuContent>
      </DropdownMenu>
    </header>
  )
}
```

---

## Dashboard page

Generate a simple dashboard with:
- Welcome message with user name
- 3-4 stat cards (counts from DB — users, main module records)
- Use shadcn/ui `Card` component
- All data fetched server-side

---

## When adding a new module to the sidebar

Add the new route and icon to `navItems` in `Sidebar.tsx`.
Always use an icon from `lucide-react`.
Keep the order: Dashboard first, then modules alphabetically, Settings last.
