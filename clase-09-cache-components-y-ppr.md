# Clase 9 — Cache Components y Partial Prerendering

**Duración:** 2:30 hs
**Branch de salida:** `clase-09-ppr`
**Lo que se llevan:** el modelo mental completo de rendering de Next 16 (estático / cacheado / dinámico **por componente**), saber leer y resolver cada error de prerender, maximizar el static shell empujando los accesos dinámicos hacia abajo, y entender qué es el App Shell y el ISR del modelo nuevo.

**Requisito:** tag `clase-08`.

> Esta es la clase más importante del curso junto con la 10. Si el grupo viene cansado, cortá la tarea, no la teoría.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:15 | El dolor: "¿por qué esta página es dinámica?" — el modelo viejo |
| 0:15 – 0:45 | El modelo nuevo: tres tipos de contenido y el static shell |
| 0:45 – 1:10 | Runtime APIs, valores no deterministas y `connection()` |
| 1:10 – 1:40 | **En vivo:** el badge del carrito en el header (el caso clásico) |
| 1:40 – 2:00 | Empujar lo dinámico hacia abajo: `params.then`, promesas, boundaries |
| 2:00 – 2:20 | Rutas con params: `generateStaticParams`, App Shell, ISR nuevo |
| 2:20 – 2:30 | Guardas (`instant`, `ensureStatic`) + tarea |

---

## Bloque 1 — El dolor y el modelo viejo (15 min)

Mostrá este código de un proyecto Next 14/15 (sin Cache Components):

```tsx
// app/products/page.tsx (modelo viejo)
export const revalidate = 60

export default async function Page() {
  const products = await fetch('https://api/products').then((r) => r.json()) // ¿cacheado? depende de la versión
  const session = cookies().get('session')  // ← esto vuelve TODA la ruta dinámica
  return <Grid products={products} user={session} />
}
```

Preguntas que nadie del equipo sabía contestar con seguridad:

- ¿El `fetch` se cachea? (En 13–14 sí por defecto; en 15 no por defecto. Cambió entre versiones.)
- ¿La página es estática? (Lo era hasta que alguien agregó `cookies()` en un componente del header. Toda la ruta pasó a dinámica **sin avisar**.)
- ¿Qué hace `revalidate = 60` si la ruta es dinámica? (Nada útil.)

> "El modelo viejo decidía **por ruta** y **en silencio**. Una línea en un componente compartido cambiaba el costo de todo el sitio y te enterabas mirando la factura. Next 16 dio vuelta las dos cosas: decide **por componente** y **te obliga a decidir a vos**."

Con `cacheComponents: true`, las configuraciones de segmento del modelo viejo (`export const dynamic`, `revalidate`, `fetchCache`) no conviven: Next te indica migrarlas. Lo que las reemplaza es lo que vemos hoy y en la clase 10.

---

## Bloque 2 — El modelo nuevo (30 min)

### 2.1 — Tres tipos de contenido

Dibujá tres columnas en el pizarrón y dejalas todo el bloque:

| **Estático** | **Cacheado** | **Dinámico** |
|---|---|---|
| Se puede calcular en build y no cambia | Cambia, pero aceptás una copia por un tiempo | Tiene que ser de **este** request |
| JSX puro, imports, `readFileSync`, cálculos | `'use cache'` + `cacheLife` (clase 10) | `cookies()`, `headers()`, `searchParams`, params desconocidos, queries sin cachear, `connection()` |
| Va al shell automáticamente | Va al shell si su vida no es demasiado corta | **Tiene que estar dentro de un `<Suspense>`**; su fallback va al shell |

### 2.2 — El static shell

En build, Next renderiza el árbol de cada ruta:

```text
<RootLayout>                    ← estático  ┐
  <Header>                      ← estático  │
    <CartBadge/>   ⟵ cookies()  ← dinámico  │  Fallback del Suspense → SHELL
  </Header>                                 │
  <Hero/>                       ← estático  │  SHELL
  <FeaturedProducts/> 'use cache' ← cacheado │  SHELL (con datos)
  <Suspense fallback={Skel}>                │  Skel → SHELL
    <StockLive/>   ⟵ query sin cache        │  se streamea por request
  </Suspense>                               ┘
```

El resultado: **HTML + RSC payload** con todo lo que se pudo resolver, y "huecos" donde van los fallbacks. Eso es el **static shell**, y se sirve desde disco o CDN sin pasar por el render. Cuando llega un request, Next manda el shell **inmediatamente** y en la misma respuesta streamea el contenido de los huecos.

> "Esto se llama Partial Prerendering y es el comportamiento por defecto con Cache Components. No es una feature que se activa por página: es cómo funciona todo."

### 2.3 — La salida del build, de nuevo

```text
○  (Static)            todo el árbol fue estático o cacheado
◐  (Partial Prerender) shell estático + huecos dinámicos
ƒ  (Dynamic)           no se pudo generar shell útil
```

> "Objetivo del curso: **cero `ƒ`** en páginas públicas. Las privadas (cuenta, panel) van a ser `◐` con un shell mínimo (layout + skeletons) y todo lo del usuario en huecos."

### 2.4 — Por qué Next frena el build

Si un acceso dinámico **no** está dentro de un `<Suspense>`, Next no puede saber dónde cortar el shell: el dato está bloqueando la página entera. En vez de volver toda la ruta dinámica en silencio (modelo viejo), **te muestra un insight en dev y falla el build**.

Los insights que van a ver (y que vienen con fixes sugeridos en el overlay):

| Insight | Qué lo causa | Fixes que ofrece |
|---|---|---|
| **blocking-route** | Dato no cacheado o runtime API fuera de Suspense | *Stream* (envolver en Suspense), *Cache* (`'use cache'`), *Block* (`instant = false`) |
| **blocking-prerender-current-time** | `Date.now()`, `new Date()` | `connection()` + Suspense, o cachear |
| **blocking-prerender-random** | `Math.random()` | idem |
| **blocking-prerender-crypto** | `crypto.randomUUID()` | idem |
| **metadata runtime / dynamic** | `generateMetadata` dinámico en página por lo demás estática | `'use cache'` o marcador dinámico explícito (clase 20) |

> "Lean el insight entero. Dice qué componente, qué línea, y las opciones. El botón **Copy prompt** genera las instrucciones para un agente. Úsenlo, pero la decisión entre *Stream* y *Cache* es de producto, no técnica: ¿el usuario tiene que ver el dato de este segundo o le alcanza con el de hace 5 minutos?"

---

## Bloque 3 — Runtime APIs y valores no deterministas (25 min)

### 3.1 — La lista

Son dinámicos (necesitan un request):

- `cookies()`, `headers()`, `draftMode()`
- `searchParams` de la page
- `params` cuyos valores **no** devolvió `generateStaticParams`
- E/S async no cacheada: queries a la base, `fetch`, `readFile` async dentro del render
- `connection()`: "esperá a que exista un request", sin leer nada

Requieren decisión explícita:

- `Date.now()`, `new Date()`, `Math.random()`, `crypto.randomUUID()`

> "`performance.now()` no entra: Next lo considera telemetría. Úsenlo para medir, no para renderizar."

Son estáticos (predecibles):

- Imports de módulos (incluidos JSON)
- `fs.readFileSync` y E/S síncrona local
- Lecturas async hechas **a nivel de módulo** (`const config = await readFile(...)` fuera del componente)

### 3.2 — `connection()`

```tsx
import { connection } from 'next/server'

async function OrderNumberPreview() {
  await connection()                     // "esto es por request"
  const n = crypto.randomUUID().slice(0, 8)
  return <p>Tu número provisorio: {n}</p>
}
```

> "`connection()` es la forma de decir 'esto es dinámico' cuando no estás leyendo cookies ni headers. Reemplaza al viejo `export const dynamic = 'force-dynamic'`, pero a nivel de componente."

También es la forma correcta de leer variables de entorno **en runtime** (clase 4): `await connection()` antes de `process.env.X` en algo que se renderiza.

---

## Bloque 4 — En vivo: el badge del carrito (30 min)

El carrito vive en una cookie `cart` (la escribimos en la clase 13; hoy la seteamos a mano en DevTools: `cart=[{"p":"queso-azul","q":2}]`).

### 4.1 — Versión ingenua

```tsx
// src/components/layout/cart-badge.tsx
import { cookies } from 'next/headers'
import { Link } from '@/i18n/navigation'
import { ShoppingBag } from 'lucide-react'

export async function CartBadge() {
  const raw = (await cookies()).get('cart')?.value
  const count = raw ? (JSON.parse(raw) as { q: number }[]).reduce((n, i) => n + i.q, 0) : 0
  return (
    <Link href="/cart" className="relative inline-flex items-center gap-1">
      <ShoppingBag className="size-5" />
      {count > 0 && <span className="rounded-full bg-primary px-1.5 text-xs text-primary-foreground">{count}</span>}
    </Link>
  )
}
```

Lo ponemos en `SiteHeader` (que está en el layout de `(shop)`). Resultado:

- Overlay: **blocking-route** en **todas** las páginas de la tienda.
- `npm run build` falla.

> "Un componente de 10 líneas en el header bloquea **todo el sitio**. En Next 15 no hubiera fallado: hubiera vuelto todo el sitio dinámico. Mismo bug, pero ahora lo ven antes de producción."

### 4.2 — La corrección

```tsx
// src/components/layout/site-header.tsx
import { Suspense } from 'react'
import { CartBadge, CartBadgeFallback } from './cart-badge'

// ...
<div className="ml-auto flex items-center gap-2">
  <LocaleSwitcher />
  <ThemeToggle />
  <Suspense fallback={<CartBadgeFallback />}>
    <CartBadge />
  </Suspense>
</div>
```

```tsx
export function CartBadgeFallback() {
  return (
    <Link href="/cart" className="relative inline-flex items-center gap-1" aria-label="Carrito">
      <ShoppingBag className="size-5" />
    </Link>
  )
}
```

> "El fallback no es un spinner: es **el mismo ícono sin el número**. El usuario no ve ningún salto; el número aparece cuando llega. Un buen fallback es el componente sin la parte dinámica."

**Validación:**

```bash
npm run build     # vuelve a pasar; las páginas de la tienda quedan en ○ → ◐
npm run start
```

`curl localhost:3000/es | grep -c ShoppingBag` → el ícono está en el HTML inicial. En el navegador con la cookie seteada, el número aparece un instante después, en el mismo response.

### 4.3 — El costo real de un hueco

> "Cada hueco significa que **cada request** a esa página ejecuta código en el servidor (acá: leer una cookie y parsear un JSON). Es barato. Si el hueco hiciera 3 queries a la base, multipliquen por el tráfico. Por eso en la clase 10 vamos a meter dentro del shell todo lo que se pueda cachear, y dejar como huecos solo lo que **de verdad** depende del usuario."

---

## Bloque 5 — Empujar lo dinámico hacia abajo (20 min)

Regla de oro (está en la doc): **cuanto más profundo en el árbol esté el `await`, más grande el shell.**

### 5.1 — Layout con params

```tsx
// ❌ el await en el layout bloquea {children} y el sidebar
export default async function ProducerLayout({ children, params }: LayoutProps<'/[locale]/producers/[slug]'>) {
  const { slug } = await params
  return (
    <div>
      <ProducerSidebar />
      <h1>{slug}</h1>
      {children}
    </div>
  )
}
```

```tsx
// ✅ el layout no es async; el await vive dentro de un boundary
import { Suspense } from 'react'

export default function ProducerLayout({ children, params }: LayoutProps<'/[locale]/producers/[slug]'>) {
  return (
    <div>
      <ProducerSidebar />
      <Suspense fallback={<h1 className="h-8 w-48 animate-pulse rounded bg-muted" />}>
        {params.then(({ slug }) => <ProducerHeading slug={slug} />)}
      </Suspense>
      {children}
    </div>
  )
}
```

> "`params.then(...)` dentro del JSX: React sabe renderizar una promesa de un elemento. El layout entero sale en el shell; solo el título espera. Es el mismo patrón que hicimos en la clase 2 con `searchParams`, ahora con `params`."

### 5.2 — Leer la cookie una vez y pasar el valor

Si varios componentes necesitan el carrito, **no** lean `cookies()` en el padre:

```tsx
// ❌ padre async → todo lo de abajo queda bloqueado
async function CartPage() {
  const cart = await readCart()
  return <><CartItems cart={cart} /><CartSummary cart={cart} /></>
}

// ✅ la promesa baja; cada hijo la espera dentro de su boundary
function CartPage() {
  const cartPromise = readCart()   // sin await (y con React.cache adentro, clase 6)
  return (
    <>
      <h1>Tu carrito</h1>
      <Suspense fallback={<CartItemsSkeleton />}><CartItems cartPromise={cartPromise} /></Suspense>
      <Suspense fallback={<CartSummarySkeleton />}><CartSummary cartPromise={cartPromise} /></Suspense>
    </>
  )
}
```

### 5.3 — `useSearchParams` en Client Components

Un Client Component que usa `useSearchParams()` en la carga inicial **suspende** (en build no hay query). Necesita su `<Suspense>`. En navegaciones del lado del cliente no suspende, porque el router ya tiene la URL. Por eso una página puede verse bien navegando y romper el build: siempre validen con `npm run build`.

---

## Bloque 6 — Rutas con params: shell concreto, App Shell e ISR (20 min)

### 6.1 — Params conocidos

`generateStaticParams` devolvió `queso-azul` → en build se genera el shell **concreto** de `/es/products/queso-azul` (con el nombre del producto si está cacheado, clase 10).

### 6.2 — Params desconocidos: App Shell

Un productor carga `miel-de-cana` después del deploy. Nadie lo listó en build. ¿Qué pasa con `/es/products/miel-de-cana`?

- Next tiene el **App Shell** de la ruta: el mismo shell pero con las partes que dependen del param detrás de sus fallbacks. Es reutilizable para cualquier slug.
- El primer visitante recibe el App Shell **al instante** y el contenido se streamea.
- En segundo plano, Next genera el shell concreto para `miel-de-cana` y lo guarda. Los siguientes visitantes reciben el shell completo.

> "Esto es el ISR del modelo nuevo (16.3). Antes tenías que elegir: o el primer visitante esperaba el render completo (bloqueante), o veía un loading que nunca se convertía en página estática. Ahora tiene las dos cosas."

Requisito para que funcione: que la parte que depende del `slug` esté dentro de un `<Suspense>` (exactamente lo que hicimos en la clase 6 con `ProductSection`).

### 6.3 — Route Handlers GET

Con Cache Components, un `GET` de `route.ts` sigue el mismo modelo que una página: si no lee nada dinámico se prerenderiza en build. Lo vemos en la clase 16.

---

## Bloque 7 — Guardas (10 min)

### 7.1 — `instant = false`: "acepto que esto bloquee"

```tsx
// src/app/(frontend)/[locale]/dashboard/layout.tsx
export const instant = false
```

Apaga las validaciones de navegación instantánea **para entrar a ese segmento**. Útil para migrar de a poco o para pantallas donde no vale la pena (un reporte pesado de admin). Las navegaciones **entre** páginas de adentro siguen validándose. Lo vemos con más detalle en la clase 12.

### 7.2 — `ensureStatic` (16.4): "esto NUNCA puede volverse dinámico"

```tsx
// src/app/(frontend)/[locale]/(shop)/about/page.tsx
export const ensureStatic = 'navigation'
```

Si mañana alguien mete un componente que lee cookies en esa página, **falla el build**. Valores (cada uno incluye al anterior):

| Valor | Qué tiene que ser estático |
|---|---|
| `'shell'` | El App Shell (lo que precarga un `<Link>` por defecto). Los huecos con Suspense se permiten. |
| `'prefetch'` | Shell + el prefetch por link (`<Link prefetch>`, clase 12) |
| `'navigation'` | **La ruta completa**. Nada de cookies, headers, queries sin cache ni `connection()`, **ni siquiera dentro de Suspense** |

Probalo en vivo: ponele `'navigation'` a `/about` y corré `npm run build`. **Falla**, aunque `/about` no lee nada. ¿Por qué?

> "Porque el nivel cubre **todos los segmentos de la ruta, incluidos los layouts de arriba**. El layout de `(shop)` tiene el `CartBadge`, que lee una cookie. Para `/about` hay dos caminos: o le alcanza con `'shell'` (el header con su hueco está permitido), o la sacamos a un route group `(legal)` con un layout sin partes dinámicas. Esta es exactamente la clase de acople que `ensureStatic` hace visible."

Elegimos mover `/about` y `/terms` a `(legal)` con un header simple y `ensureStatic = 'navigation'` en su layout. Es un test de arquitectura gratis.

---

## Ejercicio de cierre: inventario de la página de producto

Completen en grupo esta tabla para `/[locale]/products/[slug]` (es la base de la clase 10):

| Componente | Lee | Tipo hoy | Tipo objetivo | Por qué |
|---|---|---|---|---|
| Header + nav | nada | estático | estático | — |
| CartBadge | cookie | dinámico | dinámico | es del usuario |
| Nombre, foto, descripción | DB | dinámico | **cacheado** (horas) | cambia poco |
| Precio | DB | dinámico | **cacheado** (minutos) + invalidación al editar | cambia a veces |
| Stock | DB | dinámico | dinámico | tiene que ser real |
| Relacionados | DB | dinámico | cacheado | no crítico |
| Favorito (corazón lleno/vacío) | sesión | — | dinámico / `use cache: private` | es del usuario |

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| Todo el sitio falla el build después de un cambio en el header | Runtime API en un componente del layout sin Suspense | Suspense alrededor de ese componente |
| La página navega bien pero el build falla | `useSearchParams` sin Suspense (solo suspende en carga inicial) | Suspense |
| `Route segment config "revalidate" is not compatible...` | Código del modelo viejo | Reemplazar por `'use cache'` + `cacheLife` |
| Pusieron Suspense alrededor de **toda** la página | Pasa la validación pero el shell está vacío | Bajar los boundaries a lo mínimo |

---

## Tarea

1. Llevá **todas** las páginas públicas a `◐` u `○` (ninguna `ƒ`), sin usar `instant = false`.
2. Implementá `CartBadge` con su fallback sin salto de layout.
3. Agregá en la página de producto un componente `StockLive` (query directa sin cache) dentro de su propio boundary, separado del resto del detalle.
4. Route group `(legal)` con `/about` y `/terms`, layout propio y `ensureStatic = 'navigation'`; probá romperlo agregando `CartBadge` al layout y pegá el error en el PR.
5. Entregá la tabla de inventario completa para `/products` y `/cart`.

**Terminado cuando:** `npm run build` no muestra ninguna ruta pública con `ƒ` y la tabla está en el PR.
