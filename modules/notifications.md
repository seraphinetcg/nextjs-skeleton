# modules/notifications.md — Notifications

Compatible with AlwaysData: notifications stored in PostgreSQL.
No external service, no WebSockets, no Pusher required.
Uses database polling via React — simple and reliable on shared/VPS hosting.

---

## Strategy

Notifications are stored in the DB and fetched by the client every 30 seconds.
This works perfectly on AlwaysData without needing persistent WebSocket connections.
Upgrade path to real-time (SSE or Pusher) is included at the end.

---

## What to generate

1. Prisma model → add to `schema.prisma`
2. DAL functions → add to `src/lib/dal.ts`
3. Server Actions → `src/actions/notifications.ts`
4. API route for polling → `src/app/api/notifications/route.ts`
5. Notification bell component → `src/components/ui/NotificationBell.tsx`
6. Utility to create notifications → `src/lib/notify.ts`

---

## Prisma model

```prisma
model Notification {
  id        String   @id @default(cuid())
  userId    String
  title     String
  message   String
  type      String   @default("info")  // info | success | warning | error
  read      Boolean  @default(false)
  link      String?  // optional URL to navigate on click
  createdAt DateTime @default(now())
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

Add to User model:
```prisma
notifications Notification[]
```

---

## DAL functions

```typescript
// add to src/lib/dal.ts

export async function getNotifications(userId: string) {
  return prisma.notification.findMany({
    where: { userId },
    orderBy: { createdAt: 'desc' },
    take: 20,
  })
}

export async function getUnreadCount(userId: string) {
  return prisma.notification.count({
    where: { userId, read: false },
  })
}

export async function markAsRead(id: string, userId: string) {
  return prisma.notification.update({
    where: { id, userId },
    data: { read: true },
  })
}

export async function markAllAsRead(userId: string) {
  return prisma.notification.updateMany({
    where: { userId, read: false },
    data: { read: true },
  })
}

export async function deleteNotification(id: string, userId: string) {
  return prisma.notification.delete({
    where: { id, userId },
  })
}
```

---

## Notify utility (create notifications from anywhere)

```typescript
// src/lib/notify.ts
import { prisma } from '@/lib/prisma'

interface CreateNotificationParams {
  userId: string
  title: string
  message: string
  type?: 'info' | 'success' | 'warning' | 'error'
  link?: string
}

export async function notify({
  userId,
  title,
  message,
  type = 'info',
  link,
}: CreateNotificationParams) {
  return prisma.notification.create({
    data: { userId, title, message, type, link },
  })
}

export async function notifyAll({
  title,
  message,
  type = 'info',
  link,
}: Omit<CreateNotificationParams, 'userId'>) {
  const users = await prisma.user.findMany({ select: { id: true } })
  return prisma.notification.createMany({
    data: users.map((u) => ({ userId: u.id, title, message, type, link })),
  })
}
```

Usage from any Server Action:
```typescript
import { notify } from '@/lib/notify'

await notify({
  userId: session.user.id,
  title: 'Producto creado',
  message: 'El producto "iPhone 15" fue creado correctamente',
  type: 'success',
  link: '/admin/products',
})
```

---

## Server Actions

```typescript
// src/actions/notifications.ts
'use server'
import { auth } from '@/lib/auth'
import { redirect } from 'next/navigation'
import { markAsRead, markAllAsRead, deleteNotification } from '@/lib/dal'
import { revalidatePath } from 'next/cache'

export async function markAsReadAction(id: string) {
  const session = await auth()
  if (!session) redirect('/login')
  await markAsRead(id, session.user.id)
  revalidatePath('/')
}

export async function markAllAsReadAction() {
  const session = await auth()
  if (!session) redirect('/login')
  await markAllAsRead(session.user.id)
  revalidatePath('/')
}

export async function deleteNotificationAction(id: string) {
  const session = await auth()
  if (!session) redirect('/login')
  await deleteNotification(id, session.user.id)
  revalidatePath('/')
}
```

---

## API route for polling

```typescript
// src/app/api/notifications/route.ts
import { auth } from '@/lib/auth'
import { getNotifications, getUnreadCount } from '@/lib/dal'
import { NextResponse } from 'next/server'

export async function GET() {
  const session = await auth()
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })

  const [notifications, unreadCount] = await Promise.all([
    getNotifications(session.user.id),
    getUnreadCount(session.user.id),
  ])

  return NextResponse.json({ notifications, unreadCount })
}
```

---

## NotificationBell component

```typescript
// src/components/ui/NotificationBell.tsx
'use client'
import { useEffect, useState } from 'react'
import { Bell } from 'lucide-react'
import { Button } from '@/components/ui/button'
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
  DropdownMenuSeparator,
} from '@/components/ui/dropdown-menu'
import { cn } from '@/lib/utils'
import { markAsReadAction, markAllAsReadAction } from '@/actions/notifications'

interface Notification {
  id: string
  title: string
  message: string
  type: string
  read: boolean
  link?: string | null
  createdAt: string
}

const typeColors: Record<string, string> = {
  info: 'bg-blue-50 border-blue-200',
  success: 'bg-green-50 border-green-200',
  warning: 'bg-yellow-50 border-yellow-200',
  error: 'bg-red-50 border-red-200',
}

export function NotificationBell() {
  const [notifications, setNotifications] = useState<Notification[]>([])
  const [unreadCount, setUnreadCount] = useState(0)

  async function fetchNotifications() {
    try {
      const res = await fetch('/api/notifications')
      if (!res.ok) return
      const data = await res.json()
      setNotifications(data.notifications)
      setUnreadCount(data.unreadCount)
    } catch {}
  }

  useEffect(() => {
    fetchNotifications()
    const interval = setInterval(fetchNotifications, 30000) // poll every 30s
    return () => clearInterval(interval)
  }, [])

  async function handleRead(id: string) {
    await markAsReadAction(id)
    fetchNotifications()
  }

  async function handleReadAll() {
    await markAllAsReadAction()
    fetchNotifications()
  }

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" size="icon" className="relative">
          <Bell className="h-5 w-5" />
          {unreadCount > 0 && (
            <span className="absolute -top-1 -right-1 h-5 w-5 rounded-full bg-red-500 text-white text-xs flex items-center justify-center">
              {unreadCount > 9 ? '9+' : unreadCount}
            </span>
          )}
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end" className="w-80">
        <div className="flex items-center justify-between px-3 py-2">
          <span className="font-semibold text-sm">Notificaciones</span>
          {unreadCount > 0 && (
            <button
              onClick={handleReadAll}
              className="text-xs text-blue-600 hover:underline"
            >
              Marcar todas como leídas
            </button>
          )}
        </div>
        <DropdownMenuSeparator />
        {notifications.length === 0 ? (
          <div className="px-3 py-6 text-center text-sm text-gray-500">
            Sin notificaciones
          </div>
        ) : (
          notifications.map((n) => (
            <DropdownMenuItem
              key={n.id}
              className={cn(
                'flex flex-col items-start gap-1 p-3 cursor-pointer border-l-4 mb-1',
                typeColors[n.type] ?? typeColors.info,
                !n.read && 'font-medium'
              )}
              onClick={() => !n.read && handleRead(n.id)}
              asChild={!!n.link}
            >
              {n.link ? (
                <a href={n.link}>
                  <span className="text-sm">{n.title}</span>
                  <span className="text-xs text-gray-500 font-normal">{n.message}</span>
                </a>
              ) : (
                <>
                  <span className="text-sm">{n.title}</span>
                  <span className="text-xs text-gray-500 font-normal">{n.message}</span>
                </>
              )}
            </DropdownMenuItem>
          ))
        )}
      </DropdownMenuContent>
    </DropdownMenu>
  )
}
```

Add `<NotificationBell />` to `Navbar.tsx` next to the language switcher.

---

## Notification types

| Type | Color | When to use |
|---|---|---|
| `info` | Blue | Informational messages |
| `success` | Green | Successful operations |
| `warning` | Yellow | Warnings, expiring items |
| `error` | Red | Failed operations |

---

## AlwaysData notes

- Polling every 30 seconds works fine on AlwaysData — lightweight DB query
- No persistent connections needed
- For high-traffic sites, increase polling interval to 60s

---

## Optional upgrade: Server-Sent Events (SSE)

When you need real-time without polling, replace the API route with SSE:

```typescript
// src/app/api/notifications/stream/route.ts
import { auth } from '@/lib/auth'

export async function GET() {
  const session = await auth()
  if (!session) return new Response('Unauthorized', { status: 401 })

  const stream = new ReadableStream({
    start(controller) {
      const interval = setInterval(async () => {
        const count = await getUnreadCount(session.user.id)
        controller.enqueue(`data: ${JSON.stringify({ unreadCount: count })}\n\n`)
      }, 10000)

      return () => clearInterval(interval)
    },
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
    },
  })
}
```

Note: SSE requires AlwaysData to support long-lived connections — verify with their support before using.
