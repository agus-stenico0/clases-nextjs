# Clase 3 — Server Components vs Client Components

**Duración:** 2:30 hs
**Branch de salida:** `clase-03-rsc`
**Lo que se llevan:** saber dónde poner la frontera `'use client'`, qué puede cruzarla y qué no, el patrón de composición que permite meter Server Components dentro de Client Components, y cómo blindar código de servidor con `server-only`.

**Requisito:** tag `clase-02`.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:15 | El dolor: un botón con `onClick` en la página de producto |
| 0:15 – 0:45 | Qué significa realmente `'use client'` (frontera de módulo) |
| 0:45 – 1:10 | Qué cruza la frontera: serialización |
| 1:10 – 1:35 | Composición: Server dentro de Client vía `children` |
| 1:35 – 1:55 | Providers, librerías de terceros y `window` |
| 1:55 – 2:15 | `server-only`, `client-only` y envenenamiento de entorno |
| 2:15 – 2:30 | Promesas al cliente con `use()` + tarea |

---

## Bloque 1 — El dolor (15 min)

En `products/[slug]/page.tsx` agregá:

```tsx
<button onClick={() => alert('agregado')}>Agregar al carrito</button>
```

Error:

```text
Event handlers cannot be passed to Client Component props.
  <button onClick={function onClick} children=...>
                  ^^^^^^^^^^^^^^^^^^
If you need interactivity, consider converting part of this to a Client Component.
```

Un alumno va a proponer poner `'use client'` arriba de la página. Hacelo. Funciona. Ahora mostrá:

1. `getProductBySlug` usa `node:fs` → **rompe**: `Module not found: Can't resolve 'fs'`.
2. Aunque lo arregles, la página ya no puede ser `async`.

> "Pusimos `'use client'` para un botón y perdimos el servidor para toda la página. Es el error número uno del curso. La solución es al revés: la página sigue siendo servidor y el botón es una **isla** de cliente."

---

## Bloque 2 — Qué significa realmente `'use client'` (30 min)

### 2.1 — Es una frontera de módulos, no una etiqueta de componente

```text
page.tsx (server)
 ├── import ProductGallery   (server)
 ├── import ProductInfo      (server)
 └── import AddToCartButton  ← 'use client' acá
        ├── import QuantityPicker   → cliente (heredado)
        └── import formatPrice      → cliente (heredado)
```

> "`'use client'` dice: **este archivo y todo lo que importe** entran al bundle del navegador. No hace falta ponerlo en `QuantityPicker`; ya es cliente porque lo importa un archivo cliente. La directiva marca **la puerta de entrada**, no cada habitación."

### 2.2 — La isla

```tsx
// src/features/cart/components/add-to-cart-button.tsx
'use client'

import { useState } from 'react'

export function AddToCartButton({ productId, stock }: { productId: string; stock: number }) {
  const [qty, setQty] = useState(1)
  const [added, setAdded] = useState(false)

  return (
    <div className="flex items-center gap-3">
      <input
        type="number"
        min={1}
        max={stock}
        value={qty}
        onChange={(e) => setQty(Number(e.target.value))}
        className="w-16 rounded border px-2 py-1"
      />
      <button
        onClick={() => setAdded(true)}
        disabled={stock === 0}
        className="rounded bg-neutral-900 px-4 py-2 text-white disabled:opacity-40"
      >
        {added ? 'Agregado ✓' : 'Agregar al carrito'}
      </button>
    </div>
  )
}
```

> "Hoy el botón no guarda nada; en la clase 13 lo conectamos a una Server Action. Lo importante es que el resto de la página no cambió."

Página:

```tsx
// src/app/(shop)/products/[slug]/page.tsx
import { notFound } from 'next/navigation'
import { getProductBySlug } from '@/data/products'
import { AddToCartButton } from '@/features/cart/components/add-to-cart-button'
import { formatPrice } from '@/lib/money'

export default async function ProductPage(props: PageProps<'/products/[slug]'>) {
  const { slug } = await props.params
  const product = getProductBySlug(slug)
  if (!product) notFound()

  return (
    <article className="space-y-4">
      <h1 className="text-3xl font-bold">{product.name}</h1>
      <p className="text-2xl">{formatPrice(product.priceCents)}</p>
      <AddToCartButton productId={product.id} stock={product.stock} />
    </article>
  )
}
```

```ts
// src/lib/money.ts
const ars = new Intl.NumberFormat('es-AR', { style: 'currency', currency: 'ARS' })
export const formatPrice = (cents: number) => ars.format(cents / 100)
```

(Agregá `stock` al JSON.)

### 2.3 — ¿Un Client Component corre en el servidor?

**Sí**, en la primera carga: se renderiza en el servidor para generar HTML, y después se **hidrata** en el navegador. Por eso:

```tsx
'use client'
export function Bad() {
  const width = window.innerWidth // 💥 ReferenceError: window is not defined (en el server)
  return <p>{width}</p>
}
```

Formas correctas:

```tsx
'use client'
import { useEffect, useState } from 'react'

export function ViewportWidth() {
  const [width, setWidth] = useState<number | null>(null)
  useEffect(() => setWidth(window.innerWidth), [])
  return <p>{width ?? '—'}</p>
}
```

O, si la lib entera toca `window` al importarse (mapas, editores):

```tsx
'use client'
import dynamic from 'next/dynamic'

const Map = dynamic(() => import('./leaflet-map'), { ssr: false, loading: () => <p>Cargando mapa…</p> })
```

> "`ssr: false` solo se puede usar dentro de un Client Component. Si lo ponen en un Server Component, error."

### 2.4 — Medir la diferencia

Instalá una lib pesada para la descripción en markdown:

```bash
npm i marked
```

```tsx
// en la page (server)
import { marked } from 'marked'
// ...
<div className="prose" dangerouslySetInnerHTML={{ __html: await marked.parse(product.description) }} />
```

`npm run build && npm run start`, Network → JS: **`marked` no aparece** en ningún chunk. Ahora mové el render del markdown a un componente `'use client'`: el chunk crece ~40 KB.

> "Mismo resultado en pantalla, 40 KB menos para el usuario. Multipliquen eso por cada lib de fechas, de markdown, de sanitización. Por eso el default es servidor."

(Sobre `dangerouslySetInnerHTML` con contenido de usuario: en la clase 18 sanitizamos. Hoy la descripción la escribimos nosotros.)

---

## Bloque 3 — Qué cruza la frontera (25 min)

Las props de un Server Component a un Client Component se **serializan** (viajan en el RSC payload). Reglas:

| Se puede pasar | No se puede pasar |
|---|---|
| string, number, boolean, null, undefined | Funciones comunes |
| Arrays y objetos planos | Instancias de clases (con métodos) |
| `Date`, `Map`, `Set`, `BigInt`, typed arrays | Símbolos no registrados |
| Promesas (se resuelven en el cliente con `use`) | Conexiones, streams de Node, el cliente de Prisma |
| JSX (incluso Server Components ya renderizados) | |
| **Server Functions** (`'use server'`) | |

Probalo en vivo:

```tsx
// server
<AddToCartButton productId={product.id} stock={product.stock} onAdded={() => console.log('x')} />
```

```text
Error: Functions cannot be passed directly to Client Components unless you explicitly expose it by marking it with "use server".
```

### 3.1 — El problema de seguridad escondido

```tsx
// ❌ server
const user = await db.user.findUnique({ where: { id } }) // trae passwordHash, email, etc.
return <UserMenu user={user} />                          // UserMenu es 'use client'
```

Todo `user` viaja en el payload, **aunque `UserMenu` solo muestre el nombre**. Abrí el RSC payload en Network y buscalo.

> "La prop de un Client Component es un endpoint público. Lo que pasen ahí lo puede leer cualquiera con DevTools. Regla: los Client Components reciben **solo los campos que muestran**. En la clase 6 formalizamos esto con DTOs."

```tsx
// ✅
<UserMenu name={user.name} avatarUrl={user.image} />
```

---

## Bloque 4 — Composición: Server dentro de Client (25 min)

Un Client Component **no puede importar** un Server Component (si lo importa, lo convierte en cliente). Pero **sí puede recibirlo ya renderizado** como `children` o como prop.

Caso real: tabs del detalle de producto (Descripción / Productor / Reseñas). Los tabs necesitan estado (cliente). El contenido de cada tab lee datos (servidor).

```tsx
// src/components/tabs.tsx
'use client'

import { useState, type ReactNode } from 'react'

type Tab = { id: string; label: string; content: ReactNode }

export function Tabs({ tabs }: { tabs: Tab[] }) {
  const [active, setActive] = useState(tabs[0]?.id)
  return (
    <div>
      <div role="tablist" className="flex gap-4 border-b">
        {tabs.map((t) => (
          <button
            key={t.id}
            role="tab"
            aria-selected={active === t.id}
            onClick={() => setActive(t.id)}
            className="pb-2 aria-selected:border-b-2 aria-selected:border-neutral-900"
          >
            {t.label}
          </button>
        ))}
      </div>
      {tabs.map((t) => (
        <div key={t.id} role="tabpanel" hidden={active !== t.id} className="pt-4">
          {t.content}
        </div>
      ))}
    </div>
  )
}
```

```tsx
// en la page (server)
<Tabs
  tabs={[
    { id: 'desc', label: 'Descripción', content: <ProductDescription markdown={product.description} /> },
    { id: 'producer', label: 'Productor', content: <ProducerCard name={product.producer} /> },
    { id: 'reviews', label: 'Reseñas', content: <ProductReviews productId={product.id} /> },
  ]}
/>
```

`ProductDescription`, `ProducerCard` y `ProductReviews` son **Server Components async**. Se renderizan en el servidor y viajan como resultado. `Tabs` solo decide cuál mostrar.

> "Piensen en el Client Component como un marco con botones. El cuadro lo pinta el servidor. El marco no necesita saber cómo se pintó."

Dibujá el árbol:

```text
Page (S)
└── Tabs (C)
    ├── ProductDescription (S)  ← pasado como prop, renderizado en el server
    ├── ProducerCard (S)
    └── ProductReviews (S)
```

---

## Bloque 5 — Providers, terceros y contexto (20 min)

### 5.1 — Context solo existe en el cliente

```tsx
// src/components/providers.tsx
'use client'

import { ThemeProvider } from 'next-themes'
import type { ReactNode } from 'react'

export function Providers({ children }: { children: ReactNode }) {
  return (
    <ThemeProvider attribute="class" defaultTheme="system" enableSystem>
      {children}
    </ThemeProvider>
  )
}
```

```tsx
// src/app/layout.tsx (server)
<body>
  <Providers>{children}</Providers>
</body>
```

> "¿El layout se volvió cliente? No. `Providers` es cliente, pero `children` le llega ya renderizado desde el servidor (bloque 4). Todo lo que está adentro sigue siendo Server Component."

Regla: **poné los providers lo más abajo posible**. Si un contexto solo lo usa el carrito, el provider va en el layout de `(shop)`, no en el raíz.

### 5.2 — Librerías que no dicen `'use client'`

Muchas libs usan hooks pero no tienen la directiva. Importarlas en un Server Component rompe con `useState only works in Client Components`. Solución: re-exportarlas desde un archivo cliente.

```tsx
// src/components/carousel.tsx
'use client'
export { Carousel } from 'some-carousel-lib'
```

---

## Bloque 6 — `server-only` y envenenamiento de entorno (20 min)

### 6.1 — El accidente

```ts
// src/lib/payments.ts
export async function chargeCard(amount: number) {
  return fetch('https://api.pagos.com/charge', {
    headers: { Authorization: `Bearer ${process.env.PAYMENTS_SECRET}` },
    // ...
  })
}
```

Alguien la importa desde un Client Component para "probar rápido". ¿Qué pasa?

- `process.env.PAYMENTS_SECRET` en el cliente es `undefined` (solo las `NEXT_PUBLIC_*` se inyectan). Funciona mal en silencio.
- Peor: la **lógica** y la URL interna terminan en el bundle público.

### 6.2 — El candado

```bash
npm i server-only
```

```ts
// src/lib/payments.ts
import 'server-only'
// ...
```

Ahora, si un archivo cliente lo importa, **falla el build**:

```text
'server-only' cannot be imported from a Client Component module.
```

> "Todo lo que toque la base, secretos o APIs internas lleva `import 'server-only'` en la primera línea. Es una línea que convierte un incidente de seguridad en un error de compilación."

Y el inverso existe: `import 'client-only'` para código que usa `window` y no debería correr en el servidor.

### 6.3 — Variables de entorno

| Variable | Dónde está disponible |
|---|---|
| `DATABASE_URL` | Solo servidor |
| `NEXT_PUBLIC_SITE_URL` | Servidor **y** cliente (se pega en el bundle en build) |

> "`NEXT_PUBLIC_` no es 'pública' de decorado: el valor queda escrito literalmente en el JS que descarga cualquiera. Nunca una clave ahí."

---

## Bloque 7 — Promesas al cliente con `use()` (15 min)

Un Server Component puede empezar una consulta **sin esperarla** y pasarle la promesa a un Client Component. El cliente la resuelve con `use()` y se suspende mientras tanto.

```tsx
// server
import { Suspense } from 'react'

export default async function ProductPage(props: PageProps<'/products/[slug]'>) {
  const { slug } = await props.params
  const product = getProductBySlug(slug)
  if (!product) notFound()

  const relatedPromise = getRelatedProducts(product.category) // sin await

  return (
    <>
      {/* ... */}
      <Suspense fallback={<p>Cargando relacionados…</p>}>
        <RelatedCarousel productsPromise={relatedPromise} />
      </Suspense>
    </>
  )
}
```

```tsx
// client
'use client'
import { use } from 'react'
import type { Product } from '@/data/products'

export function RelatedCarousel({ productsPromise }: { productsPromise: Promise<Product[]> }) {
  const products = use(productsPromise)
  return (
    <div className="flex gap-3 overflow-x-auto">
      {products.map((p) => (
        <div key={p.id} className="min-w-40 rounded border p-3">{p.name}</div>
      ))}
    </div>
  )
}
```

> "Cuándo conviene: cuando el componente que muestra los datos necesita interactividad (carrusel con flechas, gráfico). Si no necesita interactividad, que sea un Server Component async y listo."

---

## Checklist para decidir (pegala en el aula)

```text
¿Usa useState, useEffect, useRef, onClick, onChange, context, window?
  └─ sí → Client Component. Lo más chico posible.
  └─ no → Server Component.

¿Necesita datos del servidor Y interactividad?
  └─ Server Component que busca datos → pasa props mínimas → Client Component.

¿Un Client Component necesita mostrar contenido de servidor?
  └─ Pasalo como children / prop JSX. No lo importes.

¿Toca DB, secretos o APIs internas?
  └─ import 'server-only'.
```

---

## Errores comunes

| Error | Causa | Solución |
|---|---|---|
| `Event handlers cannot be passed to Client Component props` | `onClick` en Server Component | Extraer isla cliente |
| `You're importing a component that needs useState` | Lib sin `'use client'` | Re-exportar desde archivo cliente |
| `Functions cannot be passed directly to Client Components` | Pasar callback desde server | Mover lógica al cliente o usar Server Action |
| `window is not defined` | Client Component renderizado en SSR | `useEffect` o `dynamic(..., { ssr: false })` |
| `async/await is not yet supported in Client Components` | `async function` + `'use client'` | Server Component, o `use()` con promesa |

---

## Tarea

1. Página de producto completa con: galería (server, imágenes estáticas), info (server), `AddToCartButton` (client), `Tabs` (client) con tres paneles server.
2. `ThemeToggle` (claro/oscuro) con `next-themes`, sin convertir el layout en cliente.
3. Creá `src/lib/secrets.ts` con `import 'server-only'` y probá importarlo desde un Client Component: pegá el error del build en el PR.
4. Medí con Network el JS de `/products/[slug]` antes y después de mover el markdown al servidor. Anotá los KB.

**Terminado cuando:** la página de producto tiene exactamente dos archivos con `'use client'` (botón y tabs) más el toggle de tema.
