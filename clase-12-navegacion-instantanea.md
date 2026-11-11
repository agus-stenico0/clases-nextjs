# Clase 12 — Navegación instantánea y prefetching

**Duración:** 2:30 hs
**Branch de salida:** `clase-12-instant`
**Lo que se llevan:** entender qué se precarga en cada link y cuánto cuesta, distinguir static shell / App Shell / prefetch por link / navegación, usar `<Link prefetch>`, `navigation()` y `prefetch()` para decidir qué se carga en cada etapa, inspeccionar estados de carga con el Navigation Inspector, y blindar la velocidad con tests de Playwright.

**Requisito:** tag `clase-11`. Cierra el bloque de cache: al final de esta clase va el **checkpoint técnico**.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:10 | El dolor: click y pantalla congelada |
| 0:10 – 0:35 | Las cuatro etapas de una navegación |
| 0:35 – 0:55 | Por qué un Suspense que anda en la carga no anda en la navegación |
| 0:55 – 1:20 | Partial Prefetching: el default y `<Link prefetch>` |
| 1:20 – 1:40 | `navigation()` y `prefetch()`: correr trabajo a otra etapa |
| 1:40 – 2:00 | Navigation Inspector + validación de navegación instantánea |
| 2:00 – 2:20 | Tests con `@next/playwright` |
| 2:20 – 2:30 | Tarea + consignas del checkpoint |

---

## Bloque 1 — El dolor (10 min)

Agregá una página de búsqueda: `/[locale]/search?q=queso`. Ya saben que esperar `searchParams` sin Suspense no compila, así que hacela "bien a medias", como la haría cualquiera que aprendió la clase 9 de memoria:

```tsx
export default function SearchPage(props: PageProps<'/[locale]/search'>) {
  return (
    <Suspense fallback={<FullPageSpinner />}>  {/* un solo boundary gigante */}
      <SearchResults searchParams={props.searchParams} />
    </Suspense>
  )
}
```

Desde la home, click en un link a `/es/search?q=queso`. En `npm run start` con throttling "Fast 4G": spinner de página entera, header incluido... o peor, la URL cambia y la pantalla vieja se queda congelada hasta que llega todo.

> "Pasa la validación de la clase 9, pero la experiencia es mala. Hoy aprendemos a que **algo útil** aparezca en el instante del click, siempre."

---

## Bloque 2 — Las cuatro etapas (25 min)

| Etapa | Cuándo ocurre | Qué contiene | Quién lo genera |
|---|---|---|---|
| **Static shell** | Carga directa (escribís la URL, F5) | HTML con todo lo estático/cacheado + fallbacks | Build (o ISR) — servido desde disco/CDN |
| **App Shell** | Prefetch por defecto de un `<Link>` visible | Lo mismo pero reutilizable para cualquier valor de params; incluye datos de sesión (`cookies()`/`headers()`) si están cacheados con lifetime | Servidor, se cachea por ruta y por sesión |
| **Prefetch por link** | `<Link prefetch>` visible | App Shell + contenido cacheado que depende de **esa** URL (`params`, `searchParams`) | Servidor, **una invocación por link** |
| **Navegación** | El click | Todo lo que faltaba (huecos dinámicos) | Servidor, streameado |

> "La pregunta de diseño de cada link es: **¿cuánto quiero tener listo antes del click y cuánto me cuesta?** Un link a un producto en una grilla de 40 productos: App Shell (barato, compartido). El botón 'Ir a pagar': prefetch completo (vale la pena)."

Configuración que ya tenemos desde la clase 1:

```ts
cacheComponents: true,
partialPrefetching: true,
```

> "`partialPrefetching` hay que ponerlo explícito: si falta, Next avisa. En Next 17 los dos van a ser el comportamiento por defecto y las opciones desaparecen."

Detalles de 16.x que conviene contar:

- Links a la **misma ruta** comparten **un** App Shell (40 productos = 1 request de App Shell, no 40).
- Prefetches chicos se **agrupan** en un solo request (16.3, *prefetch inlining*).
- Layouts compartidos se descargan **una vez** (16.0, *layout deduplication*).
- En `next dev` **no hay prefetch**. Siempre midan en `npm run build && npm run start`.

---

## Bloque 3 — Por qué el Suspense de la carga no siempre sirve en la navegación (20 min)

Dibujá:

```text
Carga directa de /es/products/hats         Navegación /es/products/shoes → /es/products/hats

<RootLayout>  ← se renderiza               <RootLayout>  ← ya montado, NO se re-renderiza
  <ShopLayout> ← se renderiza                <ShopLayout> ← ya montado, NO se re-renderiza
    <Suspense A>                               <Suspense A>  ← está ARRIBA del corte:
      <Page/>  ← se renderiza                     <Page/>     ← solo esto se re-renderiza
```

> "En una navegación, React solo re-renderiza **debajo del layout que comparten** la página de origen y la de destino. Un `<Suspense>` que está en el layout (arriba del corte) no se activa: no hay fallback que mostrar, entonces el router **espera** a tener la página nueva. Eso es la pantalla congelada."

Regla práctica: **cada página tiene que tener sus propios boundaries**, aunque el layout también tenga. Es lo que la validación de navegación instantánea chequea (bloque 6).

Mismo motivo por el que `useSearchParams()` se comporta distinto: en la carga inicial suspende (en el servidor no hay query); en una navegación del cliente resuelve en el acto.

---

## Bloque 4 — Partial Prefetching y `<Link prefetch>` (25 min)

### 4.1 — El default: App Shell

```tsx
<Link href={`/products/${p.slug}`}>{p.name}</Link>
```

Con el link visible en pantalla, el router pide el App Shell de `/[locale]/products/[slug]`: layout, estructura de la página, skeletons. Al hacer click aparece **eso** al instante y el resto se streamea.

Mirá en Network (filtrá por `_rsc`), en producción local: al scrollear la grilla salen pocos requests aunque haya muchos links.

### 4.2 — Prefetch por link

```tsx
<Link href={`/products/${p.slug}`} prefetch>{p.name}</Link>
```

Ahora el router resuelve `params` de **ese** link y trae también el contenido **cacheado** que depende del slug: nombre, foto, precio, descripción. Al hacer click, la página aparece **completa** salvo el stock.

Costo: **una invocación del servidor por link visible**.

| Dónde | Prefetch | Por qué |
|---|---|---|
| Grilla de 40 productos | default (App Shell) | 40 invocaciones por pantalla es caro |
| 4 destacados de la home | `prefetch` | Son los más clickeados |
| "Ir a pagar" en el carrito | `prefetch` | Es el embudo de conversión |
| Paginación / filtros de búsqueda | `prefetch` | La búsqueda está cacheada por `q`: el resultado llega antes del click |
| Links del footer | `prefetch={false}` | Nadie los clickea; ni el App Shell vale la pena |

### 4.3 — La búsqueda bien hecha

```tsx
// src/app/(frontend)/[locale]/(shop)/search/page.tsx
import { Suspense } from 'react'
import { setRequestLocale } from 'next-intl/server'
import { SearchForm } from '@/features/catalog/components/search-form'
import { SearchResults, SearchResultsSkeleton } from '@/features/catalog/components/search-results'

export default async function SearchPage(props: PageProps<'/[locale]/search'>) {
  const { locale } = await props.params
  setRequestLocale(locale)
  return (
    <div className="space-y-6">
      <SearchForm />                            {/* cliente, en el shell */}
      <Suspense fallback={<SearchResultsSkeleton />}>
        <SearchResults searchParams={props.searchParams} />
      </Suspense>
    </div>
  )
}
```

```tsx
// src/features/catalog/components/search-results.tsx
import { searchProducts } from '@/server/dal/catalog'

export async function SearchResults({ searchParams }: { searchParams: Promise<{ q?: string | string[] }> }) {
  const { q } = await searchParams
  const query = typeof q === 'string' ? q.trim().slice(0, 80) : ''
  const results = query ? await searchProducts(query) : []
  // ... grilla
}
```

```ts
// DAL
export async function searchProducts(query: string) {
  'use cache'
  cacheLife('catalog')
  cacheTag(tags.catalog)
  const lang = await rootLocale()
  // ... findMany con contains / insensitive ...
}
```

Links de "búsquedas populares" en la home con `prefetch`:

```tsx
{['queso', 'miel', 'malbec'].map((q) => (
  <Link key={q} href={{ pathname: '/search', query: { q } }} prefetch>{q}</Link>
))}
```

**Validación:** en producción local, con throttling, click en "miel" → los resultados aparecen **sin skeleton** (ya estaban precargados).

> "Ojo con la clave de cache: `searchProducts(query)` crea una entrada por cada texto distinto. Para búsquedas populares está perfecto. Si cachearan cada tecla del autocompletado, llenan la memoria con entradas que nadie reusa. Por eso recortamos y normalizamos `q`."

---

## Bloque 5 — `navigation()` y `prefetch()` (16.4) (20 min)

A veces el prefetch por link trae **demasiado**. Ejemplo: la página de pedido del panel muestra el resumen (barato) y el historial completo de eventos (caro). Queremos precargar el resumen, no el historial.

```tsx
import { navigation } from 'next/cache'

async function OrderTimeline({ orderId }: { orderId: string }) {
  await navigation()                    // ⬅ "esto no va en ningún prefetch; solo al navegar"
  const events = await getOrderEvents(orderId)
  return <Timeline events={events} />
}
```

```tsx
<OrderSummary orderId={id} />                       {/* entra al prefetch */}
<Suspense fallback={<TimelineSkeleton />}>
  <OrderTimeline orderId={id} />                    {/* solo al hacer click */}
</Suspense>
```

Y el inverso: `await prefetch()` saca algo del **App Shell** y lo deja para el prefetch por link (o la navegación):

```tsx
import { prefetch } from 'next/cache'

async function PersonalizedBanner() {
  await prefetch()   // no en el shell compartido; sí en <Link prefetch> y al navegar
  // ...
}
```

| Llamada | Sale de | Queda para |
|---|---|---|
| `await prefetch()` | App Shell | Prefetch por link + navegación |
| `await navigation()` | App Shell **y** prefetch por link | Solo navegación |

> "Esto es control fino. No lo pongan por las dudas: úsenlo cuando midan que un prefetch es caro (en Network o en la factura)."

---

## Bloque 6 — Inspeccionar y validar (20 min)

### 6.1 — Navigation Inspector

Next DevTools → **Navigation Inspector** → activá **Pause on navigations**.

1. Recargá `/es/products/queso-azul` → queda congelada en el **static shell** (etiqueta *Page load*). ¿Qué se ve? ¿Es útil?
2. **Resume**. Hacé click en un relacionado → queda congelada en el **shell de la navegación** (etiqueta *Client nav*, con URL origen y destino).
3. Compará: ¿lo que aparece en la navegación es lo que esperabas?

Combinalo con la pestaña **Suspense** de React DevTools: lista los boundaries y permite forzar fallback/resuelto en cada uno.

> "La primera vez que lo usen van a encontrar páginas donde el shell es un header y un skeleton gigante. Pasa la validación, pero no es instantáneo **de verdad**. El objetivo es que el shell tenga **contenido real**."

### 6.2 — Validación de navegación instantánea en dev

Con Cache Components, en desarrollo Next valida cada página **y** las navegaciones hacia ella desde distintos puntos de la jerarquía. Si una navegación bloquearía, aparece un insight **blocking-route** con las mismas tres opciones (Stream / Cache / Block).

Configuración:

```ts
// next.config.ts — solo si quieren validar únicamente rutas marcadas
experimental: {
  instantInsights: { validationLevel: 'manual-warning' },
},
```

Por defecto (`'warning'`) valida todo. Lo dejamos así.

### 6.3 — Optar por bloquear

```tsx
export const instant = false
```

En un layout o page: esa entrada puede bloquear y Next no lo reporta. Las navegaciones **entre hijos** se siguen validando. Para pantallas de sesión con lifetime conocido, la doc sugiere en cambio `use cache: private` con `stale` ≥ 5 minutos para que el App Shell las lleve (clase 17).

---

## Bloque 7 — Tests que impiden que se vuelva lento (20 min)

```bash
npm i -D @playwright/test @next/playwright
npx playwright install chromium   # (en WSL; en la imagen de CI ya viene)
```

```ts
// playwright.config.ts
import { defineConfig } from '@playwright/test'

export default defineConfig({
  testDir: 'e2e',
  use: { baseURL: 'http://localhost:3000' },
  webServer: { command: 'npm run dev', url: 'http://localhost:3000/es', reuseExistingServer: true },
})
```

```ts
// e2e/instant-navigation.spec.ts
import { expect, test } from '@playwright/test'
import { instant } from '@next/playwright'

test.describe('Producto', () => {
  test('carga directa: el nombre está en el shell, el stock no', async ({ page, baseURL }) => {
    await instant(
      page,
      async () => {
        await page.goto('/es/products/queso-azul')
        await expect(page.getByRole('heading', { level: 1 })).toContainText('Queso azul')
        await expect(page.getByTestId('stock')).toHaveCount(0)
      },
      { baseURL },
    )
    await expect(page.getByTestId('stock')).toBeVisible()
  })

  test('navegación desde un relacionado es instantánea', async ({ page }) => {
    await page.goto('/es/products/queso-azul')
    await instant(page, async () => {
      await page.getByRole('link', { name: 'Reggianito estacionado' }).first().click()
      await page.waitForURL('**/es/products/reggianito')
      await expect(page.getByRole('heading', { level: 1 })).toContainText('Reggianito')
    })
  })
})
```

- Dentro de `instant(...)` solo se ve lo que está disponible **sin esperar red**: shell en la carga, UI precargada en la navegación.
- Si mañana alguien mete un `cookies()` en `ProductDetail`, el nombre deja de estar en el shell y el test falla.

Para correr contra un build de producción en CI:

```ts
// next.config.ts
experimental: {
  exposeTestingApiInProductionBuild: process.env.EXPOSE_TESTING_API === '1',
},
```

```bash
EXPOSE_TESTING_API=1 npm run build && npm run start
```

> "Un test por flujo crítico: home → producto, producto → carrito, carrito → checkout. No más. Son tests de **arquitectura**, no de funcionalidad."

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| "En mi máquina el prefetch no anda" | Están en `next dev` | `npm run build && npm run start` |
| La navegación se congela aunque hay `loading.tsx` en el layout | El boundary está arriba del layout compartido | Suspense dentro de la página |
| Factura de servidor alta después de agregar `prefetch` | `prefetch` en grillas grandes | Default (App Shell) en listados |
| `use cache: private` no entra al prefetch | `stale` < 30 s | Subir `stale` |
| El test de `instant` pasa en dev y falla en CI | Build sin testing API | `exposeTestingApiInProductionBuild` |

---

## Tarea

1. Página de búsqueda completa con `SearchForm` (cliente, actualiza `?q=` con `router.replace` y debounce) y resultados cacheados.
2. Decidí el `prefetch` de **cada** `<Link>` de la tienda y documentalo en una tabla en el PR (default / `prefetch` / `false` y por qué).
3. Detalle de pedido en `/account/orders/[id]` (con datos fake por ahora) con `navigation()` en el historial de eventos.
4. Tres tests de `instant()`: home → producto, producto → relacionado, búsqueda popular → resultados.

**Terminado cuando:** los tres tests pasan y el Navigation Inspector muestra contenido real (no solo skeletons) en las tres navegaciones.

---

## Checkpoint técnico (20 % de la nota) — consigna para entregar en una semana

Se les da el branch `checkpoint-lenta` con la página `/es/producers/[slug]` hecha a propósito mal: `await` en el layout, queries sin cache, `cookies()` en un componente del medio, `loading.tsx` como único boundary, y `prefetch` en una grilla de 60 links.

Tienen que:

1. Dejarla en `◐` con el máximo de contenido en el static shell.
2. Ninguna query a la base en una visita repetida, salvo las que justifiquen como dinámicas.
3. Navegación instantánea desde la home, con test de `instant()`.
4. Un README de una página con: inventario (estático / cacheado / dinámico) de cada componente, perfiles de `cacheLife` elegidos, tags y qué los invalida, y decisión de prefetch de cada link.

Se evalúa el razonamiento tanto como el resultado.
