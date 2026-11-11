# Clase 2 — Routing a fondo

**Duración:** 2:30 hs
**Branch de salida:** `clase-02-routing`
**Lo que se llevan:** el mapa de rutas de Comarca armado (tienda, detalle, carrito, cuenta, panel), dominio de todos los archivos especiales del App Router y tipado de `params`/`searchParams` con los helpers generados.

**Requisito:** tag `clase-01`.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:10 | El dolor: header copiado en cada página |
| 0:10 – 0:35 | Carpetas = URL, `page` y `layout`, jerarquía de archivos especiales |
| 0:35 – 1:05 | Segmentos dinámicos + `PageProps` / `LayoutProps` + `searchParams` |
| 1:05 – 1:30 | `loading`, `error`, `not-found`, `template`, `global-error` |
| 1:30 – 1:55 | Route groups, carpetas privadas, múltiples root layouts |
| 1:55 – 2:20 | Navegación: `Link`, hooks de cliente, `redirect`, `notFound`, redirects en config |
| 2:20 – 2:30 | Tarea |

---

## Bloque 1 — El dolor (10 min)

Pediles que agreguen un header con logo y links a `/`, `/about` y `/categories`. Lo van a copiar en las tres páginas. Después cambiá el logo.

> "Tres archivos tocados para cambiar un logo. Y cuando naveguen entre páginas, el header se destruye y se vuelve a crear, perdiendo cualquier estado que tenga (un buscador a medio escribir, un menú abierto). El layout resuelve las dos cosas."

---

## Bloque 2 — Carpetas = URL (25 min)

### 2.1 — Las reglas

1. **Cada carpeta dentro de `app/` es un segmento de URL.**
2. **Una ruta es pública solo si tiene `page.tsx` (o `route.ts`).** Una carpeta sin `page` no existe para el navegador.
3. **Solo lo que devuelve `page`/`route` se manda al cliente.** Podés poner otros archivos al lado (colocation) sin que sean rutas.

```text
src/app/
├── layout.tsx                  → envuelve TODO
├── page.tsx                    → /
├── products/
│   ├── page.tsx                → /products
│   ├── product-card.tsx        → NO es ruta (no se llama page)
│   └── [slug]/
│       └── page.tsx            → /products/queso-azul
├── cart/
│   └── page.tsx                → /cart
└── dashboard/
    ├── layout.tsx              → envuelve /dashboard/*
    ├── page.tsx                → /dashboard
    └── products/
        └── page.tsx            → /dashboard/products
```

### 2.2 — Layout raíz y layouts anidados

```tsx
// src/app/layout.tsx
import type { Metadata } from 'next'
import Link from 'next/link'
import './globals.css'

export const metadata: Metadata = {
  title: { default: 'Comarca', template: '%s · Comarca' },
  description: 'Productos de productores locales',
}

export default function RootLayout({ children }: LayoutProps<'/'>) {
  return (
    <html lang="es">
      <body className="min-h-dvh bg-white text-neutral-900 antialiased">
        <header className="border-b">
          <nav className="mx-auto flex max-w-5xl items-center gap-6 p-4">
            <Link href="/" className="text-lg font-bold">Comarca</Link>
            <Link href="/products">Productos</Link>
            <Link href="/cart" className="ml-auto">Carrito</Link>
          </nav>
        </header>
        <div className="mx-auto max-w-5xl p-4">{children}</div>
      </body>
    </html>
  )
}
```

Reglas del root layout (preguntalas en voz alta):

- Es **obligatorio** y tiene que renderizar `<html>` y `<body>`.
- No se re-renderiza al navegar: mantiene estado.
- No recibe `searchParams` (para eso, la page).

Layout del panel:

```tsx
// src/app/dashboard/layout.tsx
import Link from 'next/link'

export default function DashboardLayout({ children }: LayoutProps<'/dashboard'>) {
  return (
    <div className="grid grid-cols-[200px_1fr] gap-6">
      <aside className="space-y-2 border-r pr-4">
        <Link href="/dashboard" className="block">Resumen</Link>
        <Link href="/dashboard/products" className="block">Mis productos</Link>
        <Link href="/dashboard/orders" className="block">Pedidos</Link>
      </aside>
      <section>{children}</section>
    </div>
  )
}
```

**Validación:** escribí algo en un `<input>` dentro del aside del panel, navegá entre `/dashboard` y `/dashboard/products`: **el texto sigue ahí**. El layout no se desmontó.

### 2.3 — La jerarquía de archivos especiales

Dibujala. Es el orden real en que Next anida los archivos de un mismo segmento:

```tsx
<Layout>
  <Template>
    <ErrorBoundary fallback={<Error />}>
      <Suspense fallback={<Loading />}>
        <NotFoundBoundary fallback={<NotFound />}>
          <Page />           {/* o el layout del segmento hijo */}
        </NotFoundBoundary>
      </Suspense>
    </ErrorBoundary>
  </Template>
</Layout>
```

> "Fíjense que `error.tsx` está **adentro** del layout. Por eso un `error.tsx` no atrapa errores de su propio `layout.tsx`: los atrapa el `error.tsx` del segmento padre. Esta pregunta la toman en entrevistas."

---

## Bloque 3 — Segmentos dinámicos y tipos generados (30 min)

### 3.1 — `[slug]`, `[...slug]`, `[[...slug]]`

| Carpeta | Matchea | `params` |
|---|---|---|
| `products/[slug]` | `/products/queso-azul` | `{ slug: 'queso-azul' }` |
| `shop/[...path]` | `/shop/a`, `/shop/a/b` | `{ path: ['a', 'b'] }` |
| `docs/[[...path]]` | `/docs`, `/docs/a/b` | `{ path: undefined }` o `['a','b']` |

### 3.2 — `params` es una promesa

```tsx
// src/app/products/[slug]/page.tsx
import { notFound } from 'next/navigation'
import { getProductBySlug, getProducts } from '@/data/products'

// Le decimos a Next qué slugs existen en build → se prerenderizan.
// A fondo en la clase 9. Sin esto, con Cache Components, leer params
// cuenta como dato de request y Next nos pide un <Suspense>.
export function generateStaticParams() {
  return getProducts().map((p) => ({ slug: p.slug }))
}

export default async function ProductPage(props: PageProps<'/products/[slug]'>) {
  const { slug } = await props.params
  const product = getProductBySlug(slug)

  if (!product) notFound()

  return (
    <article>
      <h1 className="text-3xl font-bold">{product.name}</h1>
      <p className="text-neutral-500">{product.producer}</p>
    </article>
  )
}
```

Movemos la lectura del JSON a un módulo reutilizable:

```ts
// src/data/products.ts
import { readFileSync } from 'node:fs'
import path from 'node:path'

export type Product = {
  id: string
  slug: string
  name: string
  priceCents: number
  producer: string
  category: string
}

function readAll(): Product[] {
  return JSON.parse(readFileSync(path.join(process.cwd(), 'src/data/products.json'), 'utf-8'))
}

export function getProducts() {
  return readAll()
}

export function getProductBySlug(slug: string) {
  return readAll().find((p) => p.slug === slug) ?? null
}

export function getProductsByCategory(category: string) {
  return readAll().filter((p) => p.category === category)
}
```

### 3.3 — ¿De dónde sale `PageProps`?

No lo importamos. Es un tipo **global generado** a partir de las carpetas:

```bash
npx next typegen
```

Genera tipos en `.next/types` (también se generan solos con `next dev` y `next build`). Mostrá el autocompletado:

```tsx
export default async function Page(props: PageProps<'/products/[slug]'>) {
  const { slug } = await props.params   // slug: string  ✅
  const { sku } = await props.params    // ❌ error de TS: no existe
}
```

> "Si renombran la carpeta `[slug]` a `[id]`, TypeScript les marca en rojo todas las páginas que la usan. Antes de esto, el tipo de `params` lo escribía cada uno a mano y mentía."

Equivalentes: `LayoutProps<'/ruta'>` (incluye `children` y los slots paralelos) y `RouteContext<'/ruta'>` para `route.ts`.

### 3.4 — `searchParams` (solo en `page`)

Primero la versión "ingenua":

```tsx
// src/app/products/page.tsx  (versión 1)
export default async function ProductsPage(props: PageProps<'/products'>) {
  const { category } = await props.searchParams
  // ...
}
```

Corré `npm run dev`: el overlay muestra un insight **blocking-route**. Leer `searchParams` es un dato que solo existe cuando hay un request, y lo estamos esperando en la raíz de la página: **toda** la página queda bloqueada esperando el request. Next nos pide aislarlo. Versión correcta:

```tsx
// src/app/products/page.tsx  (versión 2)
import { Suspense } from 'react'
import Link from 'next/link'
import { getProducts } from '@/data/products'

export default function ProductsPage(props: PageProps<'/products'>) {
  return (
    <div className="space-y-4">
      <nav className="flex gap-3 text-sm">
        <Link href="/products">Todos</Link>
        <Link href="/products?category=quesos">Quesos</Link>
        <Link href="/products?category=miel">Miel</Link>
        <Link href="/products?category=vinos">Vinos</Link>
      </nav>
      <Suspense fallback={<p>Cargando…</p>}>
        <ProductList searchParams={props.searchParams} />
      </Suspense>
    </div>
  )
}

async function ProductList({ searchParams }: Pick<PageProps<'/products'>, 'searchParams'>) {
  const { category } = await searchParams
  const all = getProducts()
  const products = typeof category === 'string' ? all.filter((p) => p.category === category) : all

  return (
    <ul className="grid grid-cols-3 gap-4">
      {products.map((p) => (
        <li key={p.id} className="rounded border p-3">
          <Link href={`/products/${p.slug}`}>{p.name}</Link>
        </li>
      ))}
    </ul>
  )
}
```

> "La página ya no es `async`. Le pasa la **promesa** de `searchParams` al hijo, y el hijo la espera adentro de un Suspense. Resultado: el título y los filtros salen en el HTML estático, y solo la grilla espera al request. Este patrón —pasar la promesa hacia abajo y esperarla lo más profundo posible— lo vamos a repetir todo el curso."

Señalá el tipo: `searchParams` es `Promise<Record<string, string | string[] | undefined>>`. `?category=a&category=b` da un **array**. Por eso el `typeof category === 'string'`.

> "Todo lo que viene en la URL es input del usuario. `params` y `searchParams` son tan peligrosos como un body de POST. En la clase 18 los validamos con Zod; hoy ya se acostumbran a no confiar en el tipo."

**Validación:** el overlay quedó limpio. `npm run build` muestra `/products` con `◐`.

---

## Bloque 4 — Archivos de estado: loading, error, not-found, template (25 min)

### 4.1 — `loading.tsx`

```tsx
// src/app/products/[slug]/loading.tsx
export default function Loading() {
  return <div className="h-40 animate-pulse rounded bg-neutral-100" />
}
```

Para verlo, simulá lentitud en la page:

```tsx
await new Promise((r) => setTimeout(r, 1500))
```

> "`loading.tsx` no es magia: es un `<Suspense>` que Next pone alrededor de la page. Todo lo que sepan de Suspense aplica. Y como cualquier Suspense, es grueso: tapa la página entera. En la clase 6 vamos a poner boundaries más finos adentro de la page, que es casi siempre mejor."

### 4.2 — `error.tsx`

```tsx
// src/app/products/error.tsx
'use client'

export default function ProductsError({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div className="rounded border border-red-200 bg-red-50 p-4">
      <p className="font-medium">No pudimos cargar los productos.</p>
      <p className="text-xs text-neutral-500">Código: {error.digest}</p>
      <button onClick={reset} className="mt-2 underline">Reintentar</button>
    </div>
  )
}
```

Puntos clave:

- **Tiene que ser Client Component** (es un error boundary de React, usa estado).
- En producción, `error.message` de un error del servidor **no llega** al cliente: llega un mensaje genérico y un `digest`. Con el digest buscás el error real en los logs del servidor. Esto es una medida de seguridad, no un bug.
- `reset()` vuelve a renderizar el segmento en el cliente. En la clase 21 vemos `catchError` (16.3), que además puede **reintentar los Server Components**.

Provocalo: `throw new Error('DB caída')` en `products/page.tsx`. Mostralo en `npm run dev` (mensaje real) y en `npm run build && npm run start` (mensaje genérico + digest).

### 4.3 — `not-found.tsx` y `notFound()`

```tsx
// src/app/products/[slug]/not-found.tsx
import Link from 'next/link'

export default function ProductNotFound() {
  return (
    <div>
      <h2 className="text-xl font-semibold">Ese producto no existe</h2>
      <Link href="/products" className="underline">Ver todos</Link>
    </div>
  )
}
```

**Validación:** `/products/no-existe` → se ve este componente **y el status HTTP es 404** (miralo en Network). Un "no encontrado" que devuelve 200 es un problema de SEO.

### 4.4 — `global-error.tsx`

Atrapa errores del root layout. Como reemplaza al layout raíz, **tiene que traer su propio `<html>` y `<body>`**:

```tsx
// src/app/global-error.tsx
'use client'

export default function GlobalError({ reset }: { error: Error; reset: () => void }) {
  return (
    <html lang="es">
      <body>
        <h1>Algo salió muy mal</h1>
        <button onClick={reset}>Reintentar</button>
      </body>
    </html>
  )
}
```

### 4.5 — `template.tsx`: el layout que SÍ se remonta

Igual que un layout, pero crea una instancia nueva en cada navegación. Casos reales: animación de entrada por página, resetear un formulario al cambiar de ruta, registrar una vista por navegación con un `useEffect`.

```tsx
// src/app/dashboard/template.tsx
export default function DashboardTemplate({ children }: { children: React.ReactNode }) {
  return <div className="animate-in fade-in">{children}</div>
}
```

> "Regla: si no saben si necesitan `template`, no lo necesitan. Y para animaciones entre páginas, en la clase 14 vemos View Transitions de React 19.3, que es la forma moderna."

---

## Bloque 5 — Organizar sin tocar la URL (25 min)

### 5.1 — Route groups `(nombre)`

Queremos: la tienda con header de tienda; el panel con su propio chrome; las páginas de login centradas sin header. Las URLs **no** deben cambiar.

```text
src/app/
├── layout.tsx                 ← root: solo <html>, <body>, fuentes
├── (shop)/
│   ├── layout.tsx             ← header de tienda + footer
│   ├── page.tsx               → /
│   ├── products/...           → /products
│   └── cart/page.tsx          → /cart
├── (auth)/
│   ├── layout.tsx             ← tarjeta centrada
│   ├── sign-in/page.tsx       → /sign-in
│   └── sign-up/page.tsx       → /sign-up
└── dashboard/                 ← tiene su propio layout
```

Movelo en vivo. El root layout queda mínimo y el header se va a `(shop)/layout.tsx`.

**Validación:** las URLs son las mismas que antes. `/sign-in` no tiene el header de la tienda.

### 5.2 — Dos grupos no pueden resolver la misma URL

Creá `(shop)/about/page.tsx` y `(auth)/about/page.tsx`. Next tira error: dos páginas para `/about`. Borrá una.

### 5.3 — Carpetas privadas `_nombre`

```text
src/app/(shop)/products/
├── _components/
│   ├── product-card.tsx
│   └── category-filter.tsx
├── page.tsx
└── [slug]/page.tsx
```

Todo lo que está en `_components` queda fuera del routing aunque algún día alguien le ponga un `page.tsx` adentro. Lo usamos para lo que es propio de una ruta. Lo compartido entre rutas va fuera de `app/` (clase 4).

### 5.4 — Múltiples root layouts (adelanto)

Si sacás `app/layout.tsx` y ponés un `layout.tsx` con `<html>` en cada grupo, cada grupo es un mundo aparte (navegar entre ellos hace recarga completa). Lo vamos a necesitar dos veces:

- **Clase 8:** el root layout pasa a ser `app/(frontend)/[locale]/layout.tsx` para poner `<html lang={locale}>`.
- **Clase 19:** Payload trae su propio root layout en `app/(payload)/`.

> "No lo hagan hoy. Solo sepan que existe, porque cuando lo veamos van a entender por qué no hay un `app/layout.tsx`."

---

## Bloque 6 — Navegación (25 min)

### 6.1 — `<Link>`

```tsx
<Link href="/products/queso-azul">Queso azul</Link>
<Link href={{ pathname: '/products', query: { category: 'quesos' } }}>Quesos</Link>
<Link href="/cart" replace>Carrito</Link>         {/* no agrega entrada al historial */}
<Link href="/products" scroll={false}>Productos</Link>
```

El prefetch automático lo vemos a fondo en la clase 12. Por ahora: los links visibles en pantalla precargan lo necesario para navegar rápido, **solo en producción**.

### 6.2 — Hooks de cliente (en Client Components)

```tsx
// src/app/(shop)/products/_components/category-filter.tsx
'use client'

import { usePathname, useRouter, useSearchParams } from 'next/navigation'

const categories = ['quesos', 'miel', 'vinos']

export function CategoryFilter() {
  const router = useRouter()
  const pathname = usePathname()
  const searchParams = useSearchParams()
  const current = searchParams.get('category')

  function select(category: string | null) {
    const params = new URLSearchParams(searchParams)
    if (category) params.set('category', category)
    else params.delete('category')
    router.push(`${pathname}?${params.toString()}`)
  }

  return (
    <div className="flex gap-2">
      <button onClick={() => select(null)} aria-pressed={!current}>Todos</button>
      {categories.map((c) => (
        <button key={c} onClick={() => select(c)} aria-pressed={current === c}>
          {c}
        </button>
      ))}
    </div>
  )
}
```

Usalo en la page envuelto en Suspense:

```tsx
<Suspense fallback={<div className="h-8" />}>
  <CategoryFilter />
</Suspense>
```

> "¿Por qué el Suspense? `useSearchParams` en el servidor no sabe los query params del build. Si no lo envuelven, ese componente arrastra a todo su padre a renderizarse por request. Con Suspense, solo el filtro espera."

| Hook | Para qué |
|---|---|
| `useRouter()` | `push`, `replace`, `back`, `refresh`, `prefetch` |
| `usePathname()` | ruta actual (marcar link activo) |
| `useSearchParams()` | leer query (solo lectura) |
| `useParams()` | params dinámicos desde un client component |
| `useSelectedLayoutSegment()` | segmento hijo activo de un layout (tabs) |

### 6.3 — Navegar desde el servidor

```ts
import { notFound, redirect, permanentRedirect } from 'next/navigation'

if (!product) notFound()                          // 404
if (product.slug !== slug) permanentRedirect(`/products/${product.slug}`) // 308
if (!user) redirect('/sign-in')                   // 307
```

Las tres **cortan la ejecución lanzando una excepción especial**. Por eso:

```ts
try {
  redirect('/x')
} catch (e) {
  // ❌ acá atrapan el redirect y lo rompen
}
```

> "Nunca envuelvan `redirect` o `notFound` en un try/catch genérico. Si necesitan try/catch, hagan el redirect afuera."

### 6.4 — Redirects que no necesitan código

```ts
// next.config.ts
const nextConfig: NextConfig = {
  cacheComponents: true,
  partialPrefetching: true,
  async redirects() {
    return [
      { source: '/tienda', destination: '/products', permanent: true },
      { source: '/producto/:slug', destination: '/products/:slug', permanent: true },
    ]
  },
}
```

Se ejecutan **antes** del proxy y del render (acordate del diagrama de la clase 1). Para migraciones de URLs son lo más barato que hay.

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| `You cannot have two parallel pages that resolve to the same path` | Dos route groups con la misma página | Una sola `page` por URL |
| El `error.tsx` no atrapa un error del layout | Está adentro del layout | Ponerlo en el segmento padre o `global-error` |
| El redirect no redirige | Está dentro de un `try/catch` | Sacarlo del try |
| `useSearchParams() should be wrapped in a suspense boundary` | Falta Suspense | Envolver el componente |
| Tipos de `PageProps` desactualizados | Renombraste carpetas | `npx next typegen` o reiniciar `next dev` |

---

## Tarea

1. Armá el árbol completo de rutas de Comarca (aunque las páginas tengan solo un `<h1>`):
   `/`, `/products`, `/products/[slug]`, `/categories/[slug]`, `/cart`, `/checkout`, `/account`, `/account/orders`, `/account/orders/[id]`, `/dashboard`, `/dashboard/products`, `/dashboard/products/new`, `/dashboard/products/[id]/edit`, `/dashboard/orders`, `/sign-in`, `/sign-up`.
2. Tres route groups: `(shop)`, `(auth)` y el panel con su layout.
3. `not-found.tsx` en `/products/[slug]` y `/categories/[slug]`, con 404 real. `/categories/[slug]` lleva su `generateStaticParams` (las categorías salen del JSON). Las rutas dinámicas del panel y de la cuenta que todavía son un `<h1>` no leen `params`, así que no lo necesitan.
4. Marcá el link activo del sidebar del panel usando `usePathname` en un componente cliente chico (no conviertas el layout entero en cliente).

**Terminado cuando:** todas las URLs responden, `/products/xxx` da 404, y en `npm run build` aparecen todas las rutas listadas.
