# Clase 19 — Payload CMS dentro de Next

**Duración:** 2:30 hs
**Branch de salida:** `clase-19-payload`
**Lo que se llevan:** Payload 3 instalado **en la misma app** Next (admin en `/admin`), colecciones para landing, blog de recetas, banners y media, localización alineada con next-intl, contenido leído con la **Local API** (sin HTTP) y cacheado con `'use cache'`, invalidación automática con hooks, y vista previa de borradores con Draft Mode.

**Requisito:** tag `clase-18`. La base `comarca_cms` existe desde la clase 5.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:10 | El dolor: "¿me cambiás el texto del banner?" |
| 0:10 – 0:25 | Qué es Payload y por qué adentro de Next. **Chequeo de compatibilidad** |
| 0:25 – 1:00 | Instalación: route group `(payload)`, config, base separada |
| 1:00 – 1:30 | Colecciones: Media, Pages (bloques), Posts, Banners — localizadas |
| 1:30 – 1:55 | Local API + `'use cache'` + render de rich text |
| 1:55 – 2:15 | Hooks → `revalidateTag`; Draft Mode para borradores |
| 2:15 – 2:30 | Plan B (CMS separado) + tarea |

---

## Bloque 1 — El dolor (10 min)

> "Marketing quiere cambiar el título del hero, publicar una receta por semana y poner un banner de 'envío gratis' el fin de semana largo. Hoy cada uno de esos pedidos es un ticket, un PR y un deploy. Un CMS les da un panel para hacerlo solos, y a nosotros un contrato: **ellos editan contenido, nosotros controlamos estructura y diseño**."

Qué **sí** va al CMS y qué **no**:

| CMS (Payload) | Base de la app (Prisma) |
|---|---|
| Landing, páginas institucionales, blog, banners, FAQ | Productos, stock, pedidos, usuarios, pagos |
| Lo edita marketing | Lo editan los productores desde el panel o el sistema |
| Cambia por publicación | Cambia por transacciones |

---

## Bloque 2 — Payload y compatibilidad (15 min)

Payload 3 es un CMS headless escrito en TypeScript que **se instala dentro de una app Next**: el admin es un route group más, la API REST es un route handler, y desde los Server Components se consulta con la **Local API** (funciones directas, sin HTTP).

Ventajas de tenerlo adentro: un solo deploy, tipos compartidos, cero latencia de red para leer contenido.

### 2.1 — Chequeo de compatibilidad (hacelo antes de cada cohorte)

Nuestra app usa **Next 16 con `cacheComponents: true`**. Estado al 19/10/2026:

- Payload soporta Next 16 desde la línea **3.73** (con Next ≥ 16.2).
- El soporte de **Cache Components** en Payload se mergeó el **16/10/2026** (PR `payloadcms/payload#18702`): el admin pasa a renderizarse por request y se ajusta a que Next mantenga montadas páginas ocultas (`<Activity>`).

```bash
npm view payload version
npm view payload time --json | tail -5
```

Buscá en las release notes de https://github.com/payloadcms/payload/releases que la versión que vas a instalar incluya "cache components" (#18702). **Si la incluye: seguí con esta clase.** Si todavía no salió en una release estable, usá el **Plan B** del bloque 7 (Payload en una app aparte). No mezcles `cacheComponents: true` con una versión de Payload que no lo soporta: el admin se rompe de formas difíciles de diagnosticar.

> "Esto es trabajo real de agencia: el día que vas a integrar dos piezas que evolucionan rápido, primero verificás compatibilidad en la fuente, no en un blog. Hace un año esta clase hubiera sido 'Payload no soporta cacheComponents, apaguen el flag'."

---

## Bloque 3 — Instalación (35 min)

### 3.1 — Paquetes

```bash
npm i payload @payloadcms/next @payloadcms/db-postgres @payloadcms/richtext-lexical sharp graphql
```

(`graphql` es dependencia de Payload aunque no usemos GraphQL.)

### 3.2 — Estructura

La forma más segura de traer los archivos del route group es copiarlos del template oficial en blanco (`npx create-payload-app@latest -t blank` en una carpeta temporal) o dejar que el instalador los agregue al proyecto existente. Lo que tiene que quedar:

```text
src/
├── app/
│   ├── (frontend)/[locale]/…         ← lo nuestro
│   ├── (payload)/                    ← de Payload (root layout propio)
│   │   ├── layout.tsx
│   │   ├── custom.scss
│   │   ├── admin/
│   │   │   ├── [[...segments]]/page.tsx
│   │   │   ├── [[...segments]]/not-found.tsx
│   │   │   └── importMap.js
│   │   └── cms-api/                  ← renombrado desde "api" (ver 3.4)
│   │       └── [...slug]/route.ts
│   └── api/…                         ← nuestros route handlers (auth, v1, webhooks)
├── payload/
│   ├── collections/
│   └── blocks/
└── payload.config.ts
```

> "¿Ven por qué en la clase 8 no dejamos un `app/layout.tsx`? Payload trae **su propio root layout**. Con dos root layouts (frontend y payload), cada mundo tiene su `<html>`. Navegar entre el sitio y el admin hace recarga completa, y está bien."

### 3.3 — `next.config.ts`

```ts
import { withPayload } from '@payloadcms/next/withPayload'
import createNextIntlPlugin from 'next-intl/plugin'

const withNextIntl = createNextIntlPlugin()

// ...nextConfig de siempre...

export default withPayload(withNextIntl(nextConfig))
```

`tsconfig.json`:

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"],
      "@payload-config": ["./src/payload.config.ts"]
    }
  }
}
```

### 3.4 — La config

```ts
// src/payload.config.ts
import path from 'node:path'
import { fileURLToPath } from 'node:url'
import { buildConfig } from 'payload'
import { postgresAdapter } from '@payloadcms/db-postgres'
import { lexicalEditor } from '@payloadcms/richtext-lexical'
import sharp from 'sharp'
import { Users } from './payload/collections/users'
import { Media } from './payload/collections/media'
import { Pages } from './payload/collections/pages'
import { Posts } from './payload/collections/posts'
import { Banners } from './payload/collections/banners'

const dirname = path.dirname(fileURLToPath(import.meta.url))

export default buildConfig({
  secret: process.env.PAYLOAD_SECRET!,
  serverURL: process.env.NEXT_PUBLIC_SITE_URL,

  // La API REST de Payload en /cms-api para no chocar con nuestro /api/*
  // (la carpeta del route handler tiene que llamarse igual: (payload)/cms-api)
  routes: { admin: '/admin', api: '/cms-api' },

  admin: { user: Users.slug },
  collections: [Users, Media, Pages, Posts, Banners],
  editor: lexicalEditor(),

  db: postgresAdapter({
    pool: { connectionString: process.env.PAYLOAD_DATABASE_URL },
  }),

  localization: {
    locales: [
      { code: 'es', label: 'Español' },
      { code: 'en', label: 'English' },
      { code: 'pt', label: 'Português' },
    ],
    defaultLocale: 'es',
    fallback: true, // si falta en 'pt', muestra 'es' (misma política que en la DAL, clase 8)
  },

  sharp,
  typescript: { outputFile: path.resolve(dirname, 'payload-types.ts') },
})
```

```bash
# .env.local
PAYLOAD_SECRET="..."   # openssl rand -base64 32
PAYLOAD_DATABASE_URL="postgresql://comarca:comarca@localhost:5432/comarca_cms"
```

(Sumalas al schema de env.)

> "Fíjense: **otra base** (`comarca_cms`). Prisma no la ve, Payload no ve la de Prisma. Cada ORM con sus migraciones."

```bash
npm run dev
# http://localhost:3000/admin → crear el primer usuario editor
```

En desarrollo, el adapter de Postgres sincroniza el schema solo ("push"). Para producción, Payload usa migraciones propias:

```bash
npx payload migrate:create initial
npx payload migrate        # en el deploy, como prisma migrate deploy
```

**Validación:** `/admin` carga, `/es` sigue andando, `curl localhost:3000/cms-api/posts` devuelve JSON, y `/api/auth/get-session` (Better Auth) sigue respondiendo.

---

## Bloque 4 — Colecciones (30 min)

### 4.1 — Usuarios del CMS (editores)

```ts
// src/payload/collections/users.ts
import type { CollectionConfig } from 'payload'

export const Users: CollectionConfig = {
  slug: 'users',
  auth: true,
  admin: { useAsTitle: 'email' },
  fields: [{ name: 'name', type: 'text' }],
}
```

> "Los editores del CMS **no** son los usuarios de la tienda. Son dos sistemas de auth separados a propósito: un cliente de Comarca nunca debería poder llegar al admin aunque haya un bug. Separación por diseño."

### 4.2 — Media

```ts
// src/payload/collections/media.ts
import type { CollectionConfig } from 'payload'

export const Media: CollectionConfig = {
  slug: 'media',
  access: { read: () => true },
  upload: {
    mimeTypes: ['image/jpeg', 'image/png', 'image/webp'],  // nada de SVG (clase 18)
    imageSizes: [
      { name: 'card', width: 600, height: 450, position: 'centre' },
      { name: 'hero', width: 1920, height: 800, position: 'centre' },
    ],
  },
  fields: [{ name: 'alt', type: 'text', required: true, localized: true }],
}
```

### 4.3 — Posts (recetas)

```ts
// src/payload/collections/posts.ts
import type { CollectionConfig } from 'payload'
import { revalidateCms } from '../hooks/revalidate'

export const Posts: CollectionConfig = {
  slug: 'posts',
  admin: { useAsTitle: 'title', defaultColumns: ['title', 'publishedAt', '_status'] },
  versions: { drafts: true },               // borradores y publicación
  access: {
    read: ({ req }) => (req.user ? true : { _status: { equals: 'published' } }),
  },
  fields: [
    { name: 'title', type: 'text', required: true, localized: true },
    { name: 'slug', type: 'text', required: true, unique: true, index: true },
    { name: 'excerpt', type: 'textarea', localized: true },
    { name: 'cover', type: 'upload', relationTo: 'media', required: true },
    { name: 'content', type: 'richText', localized: true },
    { name: 'productSlugs', type: 'text', hasMany: true, admin: { description: 'Productos de Comarca usados en la receta' } },
    { name: 'publishedAt', type: 'date' },
  ],
  hooks: {
    afterChange: [revalidateCms('posts')],
    afterDelete: [revalidateCms('posts')],
  },
}
```

> "`productSlugs` es el puente entre los dos mundos: la receta vive en Payload, los productos en Prisma. La página de la receta lee la receta de Payload y los productos de nuestra DAL (con su cache). No hay foreign key entre bases: si un producto desaparece, la receta lo omite."

### 4.4 — Pages con bloques (la landing)

```ts
// src/payload/blocks/index.ts
import type { Block } from 'payload'

export const HeroBlock: Block = {
  slug: 'hero',
  fields: [
    { name: 'title', type: 'text', required: true, localized: true },
    { name: 'subtitle', type: 'text', localized: true },
    { name: 'image', type: 'upload', relationTo: 'media' },
    { name: 'ctaLabel', type: 'text', localized: true },
    { name: 'ctaHref', type: 'text' },
  ],
}

export const ProductRailBlock: Block = {
  slug: 'productRail',
  fields: [
    { name: 'title', type: 'text', localized: true },
    { name: 'categorySlug', type: 'text', admin: { description: 'Vacío = destacados' } },
    { name: 'limit', type: 'number', defaultValue: 8, min: 1, max: 20 },
  ],
}

export const RichTextBlock: Block = {
  slug: 'richText',
  fields: [{ name: 'content', type: 'richText', localized: true }],
}
```

```ts
// src/payload/collections/pages.ts
import type { CollectionConfig } from 'payload'
import { HeroBlock, ProductRailBlock, RichTextBlock } from '../blocks'
import { revalidateCms } from '../hooks/revalidate'

export const Pages: CollectionConfig = {
  slug: 'pages',
  admin: { useAsTitle: 'title' },
  versions: { drafts: true },
  access: { read: ({ req }) => (req.user ? true : { _status: { equals: 'published' } }) },
  fields: [
    { name: 'title', type: 'text', required: true, localized: true },
    { name: 'slug', type: 'text', required: true, unique: true, index: true }, // 'home' para la landing
    { name: 'blocks', type: 'blocks', blocks: [HeroBlock, ProductRailBlock, RichTextBlock] },
  ],
  hooks: { afterChange: [revalidateCms('pages')], afterDelete: [revalidateCms('pages')] },
}
```

`Banners`: `text` localizado, `href`, `startsAt`, `endsAt`, `active`. (Tarea.)

```bash
npx payload generate:types       # src/payload-types.ts
npx payload generate:importmap   # si agregan componentes custom al admin
```

---

## Bloque 5 — Leer contenido: Local API + cache (25 min)

### 5.1 — El acceso a Payload también es DAL

```ts
// src/server/dal/cms.ts
import 'server-only'
import { getPayload } from 'payload'
import config from '@payload-config'
import { cacheLife, cacheTag } from 'next/cache'
import { locale as rootLocale } from 'next/root-params'
import { tags } from '@/server/cache-tags'
import type { Page, Post } from '@/payload-types'

const payload = () => getPayload({ config })

export async function getPageBySlug(slug: string): Promise<Page | null> {
  'use cache'
  cacheLife('cms')
  cacheTag(tags.cms('pages'), tags.cms(`pages:${slug}`))

  const lang = await rootLocale()
  const res = await (await payload()).find({
    collection: 'pages',
    where: { slug: { equals: slug } },
    locale: lang as 'es' | 'en' | 'pt',
    depth: 1,          // resuelve uploads (imágenes) un nivel
    limit: 1,
    draft: false,
    overrideAccess: false, // respeta 'read': solo publicados
  })
  return res.docs[0] ?? null
}

export async function listPosts(page = 1) {
  'use cache'
  cacheLife('cms')
  cacheTag(tags.cms('posts'))
  const lang = await rootLocale()
  return (await payload()).find({
    collection: 'posts',
    locale: lang as 'es' | 'en' | 'pt',
    sort: '-publishedAt',
    page,
    limit: 12,
    depth: 1,
    draft: false,
    overrideAccess: false,
    select: { title: true, slug: true, excerpt: true, cover: true, publishedAt: true },
  })
}
```

> "Local API = llamar a Payload como una función, en el mismo proceso. Cero HTTP, cero serialización JSON intermedia, tipos generados. Y como es una función async común, `'use cache'` la cachea igual que a Prisma. Ojo con `overrideAccess`: la Local API por defecto **saltea** las reglas de acceso (es 'de confianza'). Para contenido público, `overrideAccess: false` para que respete `read`."

### 5.2 — La landing desde el CMS

```tsx
// src/app/(frontend)/[locale]/(shop)/page.tsx
import { notFound } from 'next/navigation'
import { setRequestLocale } from 'next-intl/server'
import { getPageBySlug } from '@/server/dal/cms'
import { RenderBlocks } from '@/features/cms/render-blocks'

export default async function HomePage(props: PageProps<'/[locale]'>) {
  const { locale } = await props.params
  setRequestLocale(locale)
  const page = await getPageBySlug('home')
  if (!page) notFound()
  return <RenderBlocks blocks={page.blocks ?? []} />
}
```

```tsx
// src/features/cms/render-blocks.tsx
import type { Page } from '@/payload-types'
import { HeroBlock } from './blocks/hero-block'
import { ProductRailBlock } from './blocks/product-rail-block'
import { RichTextBlock } from './blocks/rich-text-block'

type Block = NonNullable<Page['blocks']>[number]

export function RenderBlocks({ blocks }: { blocks: Block[] }) {
  return (
    <>
      {blocks.map((block) => {
        switch (block.blockType) {
          case 'hero': return <HeroBlock key={block.id} {...block} />
          case 'productRail': return <ProductRailBlock key={block.id} {...block} />
          case 'richText': return <RichTextBlock key={block.id} {...block} />
          default: return null // bloque nuevo en el CMS que el front todavía no conoce: no rompe
        }
      })}
    </>
  )
}
```

```tsx
// src/features/cms/blocks/product-rail-block.tsx
import { listProducts } from '@/server/dal/catalog'   // ← Prisma, cacheado
import { ProductCard } from '@/features/catalog/components/product-card'

export async function ProductRailBlock({ title, categorySlug, limit }: { title?: string | null; categorySlug?: string | null; limit?: number | null }) {
  const products = (await listProducts(categorySlug ?? undefined)).slice(0, limit ?? 8)
  return (
    <section className="py-8">
      {title && <h2 className="mb-4 text-2xl font-semibold">{title}</h2>}
      <div className="grid grid-cols-2 gap-4 md:grid-cols-4">
        {products.map((p) => <ProductCard key={p.slug} product={p} />)}
      </div>
    </section>
  )
}
```

> "Este bloque es el punto donde se juntan los dos mundos: marketing decide **qué** riel y **dónde**; los datos del producto salen de nuestra DAL. La home sigue entera en el static shell: CMS cacheado + catálogo cacheado."

### 5.3 — Rich text

```tsx
// src/features/cms/blocks/rich-text-block.tsx
import { RichText } from '@payloadcms/richtext-lexical/react'
import type { SerializedEditorState } from '@payloadcms/richtext-lexical/lexical'

export function RichTextBlock({ content }: { content?: SerializedEditorState | null }) {
  if (!content) return null
  return <RichText data={content} className="prose dark:prose-invert max-w-none" />
}
```

> "El rich text de Lexical se guarda como JSON estructurado, no como HTML. El componente lo convierte a React: no hay `dangerouslySetInnerHTML`, no hay XSS por contenido del CMS."

### 5.4 — Imágenes del CMS

Las imágenes se sirven desde `/cms-api/media/file/...` (mismo origen). `next/image` las optimiza sin configurar `remotePatterns`. Si las mueven a S3 (adapter de storage de Payload), agregan ese host a `remotePatterns`.

---

## Bloque 6 — Invalidación y borradores (20 min)

### 6.1 — Hook → `revalidateTag`

```ts
// src/payload/hooks/revalidate.ts
import { revalidateTag } from 'next/cache'
import type { PayloadRequest } from 'payload'
import { tags } from '@/server/cache-tags'

type HookArgs = { doc: { slug?: string | null } | null; req: PayloadRequest }

// Sirve para afterChange y afterDelete: los dos reciben { doc, req } y devuelven el doc.
export const revalidateCms = (collection: 'pages' | 'posts' | 'banners') =>
  ({ doc, req }: HookArgs) => {
    // En seeds o scripts (fuera de un request de Next) se puede desactivar con context
    if (req.context?.disableRevalidate) return doc
    revalidateTag(tags.cms(collection), 'max')
    if (doc?.slug) revalidateTag(tags.cms(`${collection}:${doc.slug}`), 'max')
    return doc
  }
```

> "Los hooks de Payload corren dentro del route handler del admin, o sea, dentro de Next: `revalidateTag` funciona directo. `updateTag` no (no es una Server Action)… y tampoco hace falta: el editor no está mirando la tienda, y con `'max'` el próximo visitante dispara la regeneración."

**Validación:** cambiá el título del hero en `/admin`, publicá, recargá `/es` (en `npm run start`): título nuevo. En `/en`: también, si el campo en inglés estaba vacío (fallback) o con su propio valor.

### 6.2 — Draft Mode: ver borradores antes de publicar

```ts
// src/app/api/preview/route.ts
import { draftMode } from 'next/headers'
import { redirect } from 'next/navigation'
import { timingSafeEqual } from 'node:crypto'

export async function GET(request: Request) {
  const url = new URL(request.url)
  const secret = url.searchParams.get('secret') ?? ''
  const path = url.searchParams.get('path') ?? '/es'
  const expected = process.env.PREVIEW_SECRET ?? ''

  const ok = secret.length === expected.length && timingSafeEqual(Buffer.from(secret), Buffer.from(expected))
  if (!ok || !path.startsWith('/') || path.startsWith('//')) return new Response('invalid', { status: 401 })

  ;(await draftMode()).enable()
  redirect(path)
}
```

En la DAL, una variante sin cache para borradores:

```ts
export async function getPageForPreview(slug: string) {
  // sin 'use cache': los borradores nunca se cachean ni se comparten
  const { isEnabled } = await draftMode()
  if (!isEnabled) return null
  const res = await (await payload()).find({
    collection: 'pages', where: { slug: { equals: slug } }, draft: true, depth: 1, limit: 1,
    locale: (await rootLocale()) as 'es' | 'en' | 'pt',
  })
  return res.docs[0] ?? null
}
```

En la página: `draftMode()` es una runtime API → el chequeo va dentro de un `<Suspense>`; si está activo se renderiza la versión borrador, si no, la cacheada. En Payload, configurá `admin.preview` (o Live Preview) de la colección para que el botón "Preview" arme la URL `/api/preview?secret=…&path=/es/blog/<slug>`.

> "Draft mode pone una cookie de bypass. Mientras esté, ese navegador ve contenido fresco y sin cache. Cuidado con el secreto: quien lo tenga ve borradores (por ejemplo, la campaña de Black Friday antes de tiempo)."

---

## Bloque 7 — Plan B: el CMS en una app aparte (15 min)

Si la versión de Payload no soporta Cache Components, o si el equipo prefiere separar deploys:

```text
cms.comarca.com   → app Next mínima con solo Payload (sin cacheComponents)
comarca.com       → nuestra app, consume Payload por REST
```

Cambios:

1. `src/server/dal/cms.ts` usa `fetch('https://cms.comarca.com/cms-api/pages?where[slug][equals]=home&locale=es&depth=1')` dentro de `'use cache'` (en vez de `getPayload`).
2. El hook de Payload, en vez de `revalidateTag`, llama a un webhook nuestro:

```ts
afterChange: [async ({ doc }) => {
  await fetch(`${process.env.STOREFRONT_URL}/api/webhooks/cms`, {
    method: 'POST',
    headers: { 'content-type': 'application/json', 'x-webhook-secret': process.env.CMS_WEBHOOK_SECRET! },
    body: JSON.stringify({ collection: 'pages', slug: doc.slug }),
  })
}]
```

3. `/api/webhooks/cms` en Comarca hace `revalidateTag(tags.cms(...), 'max')` (mismo patrón que `catalog-sync` de la clase 11).
4. Tipos: `payload generate:types` en el repo del CMS y copiarlos (o un paquete compartido en un monorepo).

> "Es exactamente el patrón de la doc de Next para 'contenido de un CMS': `cacheLife('max')` + `cacheTag` + webhook que llama a `revalidateTag`. Pierden la Local API y el deploy único; ganan independencia."

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| `/admin` muestra el layout de la tienda | Quedó un `app/layout.tsx` raíz | Solo root layouts por grupo |
| El proxy redirige `/admin` a `/es/admin` | Matcher sin excluir `admin`/`cms-api` | Matcher de la clase 8 |
| `/api/auth/...` responde Payload | API de Payload en `/api` | `routes.api: '/cms-api'` + carpeta renombrada |
| La landing no cambia al publicar | Falta el hook o el tag no coincide | `revalidateCms` + `tags.cms(...)` |
| Contenido en borrador visible en la tienda | Local API con `overrideAccess` por defecto | `overrideAccess: false` + `draft: false` |
| Errores raros en el admin con `cacheComponents` | Versión de Payload sin soporte | Bloque 2.1 / Plan B |

---

## Tarea

1. Colección `Banners` (texto localizado, link, fechas, activo) y componente `PromoBanner` en el layout de la tienda que muestre el banner vigente. Pensá: ¿la vigencia por fecha es estática o dinámica? (Pista: `cacheLife` con `revalidate` corto o calcular la vigencia al guardar.)
2. Blog de recetas: `/[locale]/blog` (listado paginado) y `/[locale]/blog/[slug]` con los productos de `productSlugs` como rail comprable.
3. Landing 100 % desde el CMS (hero + 2 rails + texto), en los tres idiomas.
4. Preview de borradores funcionando desde el botón del admin.

**Terminado cuando:** marketing (vos con otro usuario del CMS) cambia el hero en portugués, publica, y `/pt` lo muestra sin deploy; y una receta en borrador no aparece en `/es/blog`.
