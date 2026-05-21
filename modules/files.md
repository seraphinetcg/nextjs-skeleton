# modules/files.md — File & Image Management

Compatible with AlwaysData: files stored on the server filesystem.
No external CDN required. Optional Cloudinary upgrade path included at the end.

---

## Strategy for AlwaysData

Files are stored in `/public/uploads/` on the server.
Next.js serves them as static files automatically.
AlwaysData persistent filesystem makes this reliable.

---

## What to generate

1. Upload Server Action → `src/actions/files.ts`
2. DAL functions → add to `src/lib/dal.ts`
3. Prisma model → add to `schema.prisma`
4. Upload component → `src/components/ui/FileUpload.tsx`
5. Image display component → `src/components/ui/AppImage.tsx`
6. API route for serving protected files → `src/app/api/files/[id]/route.ts`

---

## Prisma model

```prisma
model File {
  id        String   @id @default(cuid())
  filename  String
  mimetype  String
  size      Int
  path      String
  url       String
  uploadedBy String
  createdAt DateTime @default(now())
}
```

---

## Upload Server Action

```typescript
// src/actions/files.ts
'use server'
import { writeFile, mkdir } from 'fs/promises'
import { join } from 'path'
import { auth } from '@/lib/auth'
import { redirect } from 'next/navigation'
import { prisma } from '@/lib/prisma'

const MAX_SIZE_MB = 5
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp', 'image/gif', 'application/pdf']

export async function uploadFileAction(formData: FormData) {
  const session = await auth()
  if (!session) redirect('/login')

  const file = formData.get('file') as File
  if (!file) return { error: 'No se seleccionó archivo' }

  if (!ALLOWED_TYPES.includes(file.type)) {
    return { error: 'Tipo de archivo no permitido' }
  }

  if (file.size > MAX_SIZE_MB * 1024 * 1024) {
    return { error: `El archivo no puede superar ${MAX_SIZE_MB}MB` }
  }

  const bytes = await file.arrayBuffer()
  const buffer = Buffer.from(bytes)

  const ext = file.name.split('.').pop()
  const filename = `${Date.now()}-${Math.random().toString(36).slice(2)}.${ext}`
  const uploadDir = join(process.cwd(), 'public', 'uploads')

  await mkdir(uploadDir, { recursive: true })
  await writeFile(join(uploadDir, filename), buffer)

  const record = await prisma.file.create({
    data: {
      filename: file.name,
      mimetype: file.type,
      size: file.size,
      path: join('public', 'uploads', filename),
      url: `/uploads/${filename}`,
      uploadedBy: session.user.id,
    },
  })

  return { success: true, file: record }
}

export async function deleteFileAction(id: string) {
  const session = await auth()
  if (!session) redirect('/login')

  const { unlink } = await import('fs/promises')
  const record = await prisma.file.findUnique({ where: { id } })
  if (!record) return { error: 'Archivo no encontrado' }

  try {
    await unlink(join(process.cwd(), record.path))
  } catch {
    // file already gone from disk, continue
  }

  await prisma.file.delete({ where: { id } })
  return { success: true }
}
```

---

## DAL functions

```typescript
// add to src/lib/dal.ts

export async function getFiles(uploadedBy?: string) {
  return prisma.file.findMany({
    where: uploadedBy ? { uploadedBy } : undefined,
    orderBy: { createdAt: 'desc' },
  })
}

export async function getFileById(id: string) {
  return prisma.file.findUnique({ where: { id } })
}
```

---

## FileUpload component

```typescript
// src/components/ui/FileUpload.tsx
'use client'
import { useRef, useState } from 'react'
import { uploadFileAction } from '@/actions/files'
import { Button } from '@/components/ui/button'
import { Upload, X } from 'lucide-react'

interface FileUploadProps {
  onUploaded?: (url: string, fileId: string) => void
  accept?: string
}

export function FileUpload({ onUploaded, accept = 'image/*' }: FileUploadProps) {
  const inputRef = useRef<HTMLInputElement>(null)
  const [preview, setPreview] = useState<string | null>(null)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  async function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0]
    if (!file) return

    if (file.type.startsWith('image/')) {
      setPreview(URL.createObjectURL(file))
    }

    setLoading(true)
    setError(null)
    const formData = new FormData()
    formData.append('file', file)

    const result = await uploadFileAction(formData)
    setLoading(false)

    if (result.error) {
      setError(result.error)
      return
    }
    if (result.success && result.file) {
      onUploaded?.(result.file.url, result.file.id)
    }
  }

  return (
    <div className="space-y-2">
      <div
        className="border-2 border-dashed rounded-lg p-6 text-center cursor-pointer hover:bg-gray-50 transition-colors"
        onClick={() => inputRef.current?.click()}
      >
        {preview ? (
          <img src={preview} alt="preview" className="mx-auto max-h-32 object-contain" />
        ) : (
          <div className="flex flex-col items-center gap-2 text-gray-500">
            <Upload className="h-8 w-8" />
            <span className="text-sm">Clic para subir archivo</span>
            <span className="text-xs">JPG, PNG, WebP, PDF — máx 5MB</span>
          </div>
        )}
      </div>
      <input
        ref={inputRef}
        type="file"
        accept={accept}
        onChange={handleChange}
        className="hidden"
      />
      {loading && <p className="text-sm text-gray-500">Subiendo...</p>}
      {error && <p className="text-sm text-red-500">{error}</p>}
    </div>
  )
}
```

---

## AppImage component

```typescript
// src/components/ui/AppImage.tsx
import Image from 'next/image'
import { cn } from '@/lib/utils'

interface AppImageProps {
  src: string
  alt: string
  className?: string
  width?: number
  height?: number
}

export function AppImage({ src, alt, className, width = 400, height = 300 }: AppImageProps) {
  return (
    <Image
      src={src}
      alt={alt}
      width={width}
      height={height}
      className={cn('rounded-md object-cover', className)}
    />
  )
}
```

---

## next.config.ts — required for local images

```typescript
const nextConfig = {
  images: {
    localPatterns: [
      {
        pathname: '/uploads/**',
      },
    ],
  },
}
```

---

## .gitignore additions

```
public/uploads/
```

Never commit uploaded files to git.

---

## AlwaysData notes

- Uploaded files persist between deploys because AlwaysData uses a persistent filesystem
- The `public/uploads/` directory must be created manually once on the server
- During deploy, do NOT delete `public/uploads/` — only deploy code changes

---

## Optional upgrade: Cloudinary

When the project grows and needs CDN, replace `uploadFileAction` with:
```bash
npm install cloudinary
```
Add `CLOUDINARY_URL` to secrets and update the action to upload to Cloudinary instead of local disk.
The rest of the code (components, DAL) stays the same — only the action changes.
