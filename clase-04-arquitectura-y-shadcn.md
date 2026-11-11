# Clase 4 — Arquitectura de Comarca, shadcn/ui y variables de entorno

**Duración:** 2:30 hs
**Branch de salida:** `clase-04-arquitectura`
**Lo que se llevan:** la estructura de carpetas definitiva del proyecto (la que vamos a respetar hasta la clase 23), un sistema de diseño con Tailwind 4 + shadcn/ui con modo oscuro, fuentes optimizadas, y variables de entorno validadas que rompen el build si falta algo.

**Requisito:** tag `clase-03`.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:10 | El dolor: ¿dónde va este archivo? |
| 0:10 – 0:40 | Las tres estrategias de la doc y la que elegimos |
| 0:40 – 1:15 | Tailwind 4 + shadcn/ui: init, tokens, componentes |
| 1:15 – 1:35 | **En vivo:** header responsive + `ProductCard` con shadcn |
| 1:35 – 1:50 | `next/font` y modo oscuro sin flash |
| 1:50 – 2:15 | Variables de entorno validadas con Zod |
| 2:15 – 2:30 | ESLint flat, Prettier, scripts y tarea |

---

## Bloque 1 — El dolor (10 min)

Preguntá: "¿dónde pondrían `formatPrice`? ¿Y el componente `ProductCard` que se usa en la home, en `/products` y en el panel? ¿Y la función que consulta productos?". Vas a tener cinco respuestas distintas.

> "Next no les impone estructura. La doc lo dice textual: *Next.js is unopinionated about how you organize your project*. Eso es libertad el día uno y caos el día noventa. Hoy decidimos una vez y la respetamos todo el curso."

---

## Bloque 2 — Las tres estrategias y la nuestra (30 min)

Abrí https://nextjs.org/docs/app/getting-started/project-structure y mostrá las tres que propone:

1. **Archivos fuera de `app/`**: `app/` solo tiene rutas; todo lo demás en carpetas hermanas.
2. **Archivos en carpetas top-level dentro de `app/`**: `app/components`, `app/lib`.
3. **Partido por feature o ruta**: lo global arriba, lo específico colocado en la ruta que lo usa.

Elegimos una mezcla de 1 y 3, con `src/`:

```text
comarca-web/
├── docker/                       ← infraestructura local (clase 5)
├── messages/                     ← traducciones (clase 8)
├── prisma/                       ← schema, migraciones, seed (clase 5)
├── public/
├── src/
│   ├── app/                      ← SOLO routing + _components propios de cada ruta
│   │   ├── (frontend)/[locale]/… ← (desde la clase 8)
│   │   ├── (payload)/…           ← (desde la clase 19)
│   │   └── api/…                 ← route handlers
│   ├── components/
│   │   ├── ui/                   ← shadcn (generado, se toca poco)
│   │   └── layout/               ← header, footer, sidebar
│   ├── features/                 ← un módulo por dominio de negocio
│   │   ├── catalog/
│   │   │   ├── components/       ← ProductCard, ProductGrid…
│   │   │   ├── actions.ts        ← Server Actions ('use server')
│   │   │   └── schemas.ts        ← Zod compartido client/server
│   │   ├── cart/
│   │   ├── checkout/
│   │   ├── orders/
│   │   └── producer-dashboard/
│   ├── server/                   ← infraestructura de servidor (server-only)
│   │   ├── db.ts                 ← cliente Prisma (clase 5)
│   │   ├── auth.ts               ← Better Auth (clase 17)
│   │   └── dal/                  ← Data Access Layer: TODAS las lecturas/escrituras
│   │       ├── catalog.ts        ←   (server-only, auth, DTOs, cache) — clases 5, 6, 10
│   ├── i18n/                     ← (clase 8)
│   ├── lib/                      ← utilidades puras sin dependencia de dominio
│   │   ├── utils.ts              ← cn()
│   │   ├── money.ts
│   │   └── env/
│   ├── generated/                ← cliente Prisma generado (no va al repo)
│   └── proxy.ts                  ← (clase 7)
└── next.config.ts
```

### 2.1 — Reglas de dependencia (escribilas en el pizarrón)

```text
app/        → puede importar features/, components/, lib/, server/
features/X  → puede importar components/, lib/, server/
              NO importa internals de features/Y (si lo necesita, se sube a lib/ o se expone)
components/ → puede importar lib/
lib/        → no importa nada del proyecto
server/     → puede importar lib/; nunca desde un archivo 'use client'
```

> "Si `features/cart` necesita algo de `features/catalog`, primero pregúntense si no es algo genérico que va en `lib/`. Esta regla evita el día en que tocar el carrito rompe el catálogo."

Lo forzamos con ESLint en el bloque 7.

### 2.2 — `app/` como capa delgada

Una `page.tsx` ideal se lee como un índice:

```tsx
// src/app/(shop)/products/page.tsx
import { Suspense } from 'react'
import { ProductGrid, ProductGridSkeleton } from '@/features/catalog/components/product-grid'
import { CategoryNav } from '@/features/catalog/components/category-nav'

export default function ProductsPage(props: PageProps<'/products'>) {
  return (
    <section className="space-y-6">
      <h1 className="text-3xl font-semibold tracking-tight">Productos</h1>
      <CategoryNav />
      <Suspense fallback={<ProductGridSkeleton />}>
        <ProductGrid searchParams={props.searchParams} />
      </Suspense>
    </section>
  )
}
```

> "La página arma; los features hacen. Si una page tiene más de 60 líneas, algo se está cocinando en el lugar equivocado."

Mové en vivo lo que hicimos en las clases 2 y 3 a `features/catalog`.

---

## Bloque 3 — Tailwind 4 + shadcn/ui (35 min)

### 3.1 — Tailwind 4 en dos líneas

`create-next-app` ya lo dejó instalado. Abrí `src/app/globals.css`:

```css
@import "tailwindcss";
```

> "No hay `tailwind.config.js`. En la versión 4 la configuración vive en CSS con `@theme`. Detecta los archivos solo."

### 3.2 — Inicializar shadcn

```bash
npx shadcn@latest init
```

Respuestas: base color **Neutral**, CSS variables **sí**. Lo que hace:

- Crea `components.json` (dónde van los componentes, alias, estilo).
- Crea `src/lib/utils.ts` con `cn()`.
- Agrega a `globals.css` los tokens de color como variables CSS (light y dark) y los mapea con `@theme inline`.

```ts
// src/lib/utils.ts
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

> "shadcn no es una librería que se instala y se actualiza: **copia el código del componente a su repo**. El botón es de ustedes. Si quieren cambiarle el radio a todos los botones, editan `components/ui/button.tsx`. Eso es lo que lo hace bueno para una agencia: cada cliente tiene su sistema de diseño sin pelear contra una lib."

### 3.3 — Componentes que vamos a usar en el curso

```bash
npx shadcn@latest add button card input label textarea select badge \
  dialog sheet dropdown-menu separator skeleton sonner field table tabs avatar
```

Mostrá `src/components/ui/button.tsx`: usa `cva` para variantes y `Slot` para `asChild`.

```tsx
<Button asChild variant="outline">
  <Link href="/products">Ver productos</Link>
</Button>
```

> "`asChild` hace que el botón no renderice un `<button>`, sino que le pase sus estilos al hijo. Un link con aspecto de botón sigue siendo un `<a>`: accesible y con prefetch."

### 3.4 — Marca propia

Comarca tiene un verde oliva como color primario. Editá los tokens en `globals.css`:

```css
:root {
  --primary: oklch(0.45 0.09 130);
  --primary-foreground: oklch(0.98 0.01 130);
  --radius: 0.75rem;
}

.dark {
  --primary: oklch(0.72 0.11 130);
  --primary-foreground: oklch(0.2 0.03 130);
}
```

**Validación:** todos los `<Button>` cambiaron de color sin tocar un componente.

---

## Bloque 4 — En vivo: header y ProductCard (20 min)

### 4.1 — ProductCard (Server Component, usa componentes de shadcn)

```tsx
// src/features/catalog/components/product-card.tsx
import Link from 'next/link'
import Image from 'next/image'
import { Badge } from '@/components/ui/badge'
import { Card, CardContent, CardFooter } from '@/components/ui/card'
import { formatPrice } from '@/lib/money'

export type ProductCardData = {
  slug: string
  name: string
  producerName: string
  priceCents: number
  imageUrl: string
  stock: number
}

export function ProductCard({ product }: { product: ProductCardData }) {
  return (
    <Card className="overflow-hidden pt-0">
      <Link href={`/products/${product.slug}`} className="block">
        <Image
          src={product.imageUrl}
          alt={product.name}
          width={400}
          height={300}
          className="aspect-[4/3] w-full object-cover"
        />
      </Link>
      <CardContent className="space-y-1">
        <Link href={`/products/${product.slug}`} className="font-medium hover:underline">
          {product.name}
        </Link>
        <p className="text-sm text-muted-foreground">{product.producerName}</p>
      </CardContent>
      <CardFooter className="justify-between">
        <span className="font-semibold">{formatPrice(product.priceCents)}</span>
        {product.stock === 0 && <Badge variant="secondary">Sin stock</Badge>}
      </CardFooter>
    </Card>
  )
}
```

> "Fíjense que shadcn **no** obliga a `'use client'`. `Card`, `Badge` y `Button` son componentes de presentación: funcionan como Server Components. Los que tienen estado (`Dialog`, `DropdownMenu`, `Sheet`) ya traen `'use client'` adentro."

Imágenes de prueba: poné 10 JPG en `public/products/`. Remotas las configuramos en la clase 20.

### 4.2 — Header con menú móvil

```tsx
// src/components/layout/site-header.tsx
import Link from 'next/link'
import { Button } from '@/components/ui/button'
import { MobileNav } from './mobile-nav'
import { ThemeToggle } from './theme-toggle'

const links = [
  { href: '/products', label: 'Productos' },
  { href: '/categories/quesos', label: 'Quesos' },
  { href: '/categories/vinos', label: 'Vinos' },
]

export function SiteHeader() {
  return (
    <header className="sticky top-0 z-40 border-b bg-background/80 backdrop-blur">
      <div className="mx-auto flex h-14 max-w-6xl items-center gap-6 px-4">
        <MobileNav links={links} />
        <Link href="/" className="font-serif text-xl font-semibold">Comarca</Link>
        <nav className="hidden gap-5 text-sm md:flex">
          {links.map((l) => (
            <Link key={l.href} href={l.href} className="text-muted-foreground hover:text-foreground">
              {l.label}
            </Link>
          ))}
        </nav>
        <div className="ml-auto flex items-center gap-2">
          <ThemeToggle />
          <Button asChild size="sm">
            <Link href="/cart">Carrito</Link>
          </Button>
        </div>
      </div>
    </header>
  )
}
```

```tsx
// src/components/layout/mobile-nav.tsx
'use client'

import Link from 'next/link'
import { useState } from 'react'
import { Menu } from 'lucide-react'
import { Button } from '@/components/ui/button'
import { Sheet, SheetContent, SheetTitle, SheetTrigger } from '@/components/ui/sheet'

export function MobileNav({ links }: { links: { href: string; label: string }[] }) {
  const [open, setOpen] = useState(false)
  return (
    <Sheet open={open} onOpenChange={setOpen}>
      <SheetTrigger asChild>
        <Button variant="ghost" size="icon" className="md:hidden" aria-label="Abrir menú">
          <Menu />
        </Button>
      </SheetTrigger>
      <SheetContent side="left">
        <SheetTitle>Comarca</SheetTitle>
        <nav className="mt-6 grid gap-4">
          {links.map((l) => (
            <Link key={l.href} href={l.href} onClick={() => setOpen(false)}>
              {l.label}
            </Link>
          ))}
        </nav>
      </SheetContent>
    </Sheet>
  )
}
```

> "El header es Server Component. La única isla cliente es el menú móvil, que necesita estado. Los `links` cruzan la frontera como array de objetos planos: serializable."

---

## Bloque 5 — Fuentes y modo oscuro (15 min)

### 5.1 — `next/font`

```tsx
// src/app/layout.tsx
import { Inter, Fraunces } from 'next/font/google'

const sans = Inter({ subsets: ['latin'], variable: '--font-sans', display: 'swap' })
const serif = Fraunces({ subsets: ['latin'], variable: '--font-serif', display: 'swap' })

export default function RootLayout({ children }: LayoutProps<'/'>) {
  return (
    <html lang="es" className={`${sans.variable} ${serif.variable}`} suppressHydrationWarning>
      <body className="min-h-dvh bg-background font-sans text-foreground antialiased">
        <Providers>{children}</Providers>
      </body>
    </html>
  )
}
```

```css
/* globals.css */
@theme inline {
  --font-sans: var(--font-sans);
  --font-serif: var(--font-serif);
}
```

> "`next/font` descarga la fuente **en build** y la sirve desde su propio dominio. Cero requests a Google en runtime (mejor privacidad, mejor CSP) y cero salto de layout porque calcula el fallback con las mismas métricas."

### 5.2 — Modo oscuro sin flash

Ya hicimos `Providers` con `next-themes` en la clase 3. Dos detalles:

- `suppressHydrationWarning` en `<html>`: `next-themes` pone la clase `dark` antes de que React hidrate. Sin eso, React avisa que el server y el cliente no coinciden.
- El toggle:

```tsx
// src/components/layout/theme-toggle.tsx
'use client'

import { Moon, Sun } from 'lucide-react'
import { useTheme } from 'next-themes'
import { Button } from '@/components/ui/button'

export function ThemeToggle() {
  const { resolvedTheme, setTheme } = useTheme()
  return (
    <Button
      variant="ghost"
      size="icon"
      aria-label="Cambiar tema"
      onClick={() => setTheme(resolvedTheme === 'dark' ? 'light' : 'dark')}
    >
      <Sun className="dark:hidden" />
      <Moon className="hidden dark:block" />
    </Button>
  )
}
```

> "El ícono se decide con CSS (`dark:hidden`), no con JS. Así no hay parpadeo del ícono equivocado mientras hidrata."

---

## Bloque 6 — Variables de entorno validadas (25 min)

### 6.1 — El dolor

```ts
const url = process.env.DATABSE_URL // typo
```

TypeScript no se queja (es `string | undefined`), el build pasa, y explota en producción a las 3 AM.

### 6.2 — Cómo carga Next los `.env`

Orden de prioridad (gana el primero que define la variable):

1. Variables reales del proceso (`DATABASE_URL=... npm start`, Docker, Coolify)
2. `.env.$(NODE_ENV).local`
3. `.env.local` (no se carga en `test`)
4. `.env.$(NODE_ENV)`
5. `.env`

Convención del curso:

- `.env.example` → **va al repo**, sin valores reales, documenta todo.
- `.env.local` → valores de cada dev, **no va al repo**.

### 6.3 — Validación con Zod, separando servidor y cliente

```bash
npm i zod
```

```ts
// src/lib/env/server.ts
import 'server-only'
import { z } from 'zod'

const schema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  DATABASE_URL: z.url(),
  // Se van sumando en las clases que los necesiten:
  // BETTER_AUTH_SECRET: z.string().min(32),
  // PAYLOAD_SECRET: z.string().min(32),
})

const parsed = schema.safeParse(process.env)

if (!parsed.success) {
  console.error('❌ Variables de entorno inválidas:')
  console.error(z.prettifyError(parsed.error))
  throw new Error('Variables de entorno inválidas')
}

export const env = parsed.data
```

```ts
// src/lib/env/client.ts
import { z } from 'zod'

// Next reemplaza process.env.NEXT_PUBLIC_X por su valor en build.
// Por eso hay que nombrar cada variable explícitamente: un process.env
// "dinámico" (process.env[key]) no se reemplaza.
const parsed = z
  .object({
    NEXT_PUBLIC_SITE_URL: z.url(),
  })
  .safeParse({
    NEXT_PUBLIC_SITE_URL: process.env.NEXT_PUBLIC_SITE_URL,
  })

if (!parsed.success) {
  throw new Error(`Variables públicas inválidas: ${z.prettifyError(parsed.error)}`)
}

export const publicEnv = parsed.data
```

Los dos módulos validan cuando alguien los importa. Para que el error salte **al arrancar** (y no cuando alguien visita la primera página que usa la base), validamos también en `next.config.ts`, que Next evalúa después de cargar los `.env*`:

```ts
// next.config.ts
import type { NextConfig } from 'next'
import { z } from 'zod'

// Falla rápido en `next dev` y `next build` si falta algo esencial.
const required = z.object({
  DATABASE_URL: z.url(),
  NEXT_PUBLIC_SITE_URL: z.url(),
})
const check = required.safeParse(process.env)
if (!check.success) {
  console.error('❌ Variables de entorno inválidas:\n' + z.prettifyError(check.error))
  process.exit(1)
}

const nextConfig: NextConfig = {
  cacheComponents: true,
  partialPrefetching: true,
}

export default nextConfig
```

> "Sí, el schema está duplicado con `env/server.ts`. Es a propósito: `next.config.ts` lo evalúa Node fuera del bundler y no queremos que importe un archivo con `'server-only'`. Son cuatro líneas; si crece, se extrae a un `.mjs` compartido."

```bash
# .env.example
DATABASE_URL="postgresql://comarca:comarca@localhost:5432/comarca"
NEXT_PUBLIC_SITE_URL="http://localhost:3000"
```

**Validación:** borrá `DATABASE_URL` de `.env.local` y corré `npm run dev`: el proceso termina con un mensaje claro. Volvé a ponerla.

### 6.4 — La trampa con el prerender

> "Ojo con esto que les va a pegar en Docker (clase 22): un componente que se **prerenderiza en build** y lee `process.env.ALGO` captura el valor **del momento del build**. Si después cambian la variable en el servidor, ese HTML ya tiene el valor viejo. Para valores que deben leerse en runtime, la doc indica llamar a `await connection()` antes de leer `process.env`, o leerlos solo en código que corre por request. Para las `NEXT_PUBLIC_*` no hay escapatoria: se pegan en el JS en build."

---

## Bloque 7 — ESLint, Prettier y scripts (15 min)

```bash
npm i -D prettier prettier-plugin-tailwindcss
```

```json
// .prettierrc
{ "semi": false, "singleQuote": true, "printWidth": 100, "plugins": ["prettier-plugin-tailwindcss"] }
```

Regla de dependencias en ESLint (flat config):

```js
// eslint.config.mjs (agregar al array exportado)
{
  files: ['src/lib/**/*.{ts,tsx}'],
  rules: {
    'no-restricted-imports': ['error', { patterns: ['@/features/*', '@/server/*', '@/app/*'] }],
  },
},
{
  files: ['src/components/**/*.{ts,tsx}'],
  rules: {
    'no-restricted-imports': ['error', { patterns: ['@/features/*', '@/server/*'] }],
  },
},
```

Scripts:

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint .",
    "typecheck": "next typegen && tsc --noEmit",
    "format": "prettier --write ."
  }
}
```

> "`next build` ya no corre el linter (se sacó en la 16). Si quieren lint en CI, lo corren ustedes. Lo agregamos al pipeline en la clase 22."

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| Flash de tema claro al cargar en oscuro | Falta `suppressHydrationWarning` o `attribute="class"` | Revisar `<html>` y `ThemeProvider` |
| `Invalid src prop ... hostname is not configured` | Imagen remota sin `remotePatterns` | Por ahora usar `/public`; clase 20 |
| Las clases de Tailwind no se aplican en un archivo nuevo | Archivo fuera de lo que detecta | Verificar que esté en `src/` |
| `NEXT_PUBLIC_X` es `undefined` en el cliente | Se lee con `process.env[nombre]` dinámico | Nombrarla literalmente |

---

## Tarea

1. Migrá todo el código de las clases 1–3 a la estructura nueva (`features/catalog`, `components/layout`, `lib/money.ts`).
2. Home con: hero (texto + botón con `asChild`), grilla de 6 `ProductCard`, sección "Categorías" con 3 cards.
3. `ProductGridSkeleton` hecho con `Skeleton` de shadcn, con la misma grilla que la real (sin salto de layout cuando carga).
4. `.env.example` completo y validación funcionando.
5. `npm run lint && npm run typecheck && npm run build` en verde.

**Terminado cuando:** ningún archivo de `src/lib` ni `src/components` importa de `features/` (ESLint lo confirma) y el modo oscuro no parpadea al recargar.
