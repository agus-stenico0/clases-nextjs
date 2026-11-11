# Clase 21 — Errores, observabilidad y testing

**Duración:** 2:30 hs
**Branch de salida:** `clase-21-calidad`
**Lo que se llevan:** una estrategia de errores por capas (boundaries de ruta y de componente con reintento del servidor), `instrumentation.ts` para capturar cada error del servidor con su contexto, logs estructurados con request-id, y una pirámide de tests realista para Next: unitarios, DAL contra Postgres de verdad, Server Actions, y e2e con Playwright.

**Requisito:** tag `clase-20`.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:10 | El dolor: "en producción algo falla" y solo tenemos un digest |
| 0:10 – 0:40 | Errores por capas: `error.tsx`, `global-error`, `catchError` + `retry()` |
| 0:40 – 1:05 | `instrumentation.ts`: `register` y `onRequestError` |
| 1:05 – 1:20 | Logs estructurados con request-id (y qué no loguear) |
| 1:20 – 2:15 | Testing: Vitest (unit, DAL, actions) + Playwright (flujos) |
| 2:15 – 2:30 | Tarea |

---

## Bloque 1 — El dolor (10 min)

Un usuario manda captura: "Algo salió mal. Código: 2817364519". En los logs del servidor: nada útil, porque nadie loguea con contexto.

> "En producción, Next **oculta** el mensaje de los errores del servidor (podría tener SQL, rutas o datos) y le muestra al usuario un `digest`. Ese número es el puente entre lo que ve el usuario y lo que pasó en el servidor. Hoy construimos el otro lado del puente."

---

## Bloque 2 — Errores por capas (30 min)

### 2.1 — El mapa

```text
global-error.tsx      ← falla el root layout (reemplaza todo, trae su <html>)
  [locale]/error.tsx  ← falla cualquier página del sitio sin boundary más cercano
    (shop)/products/error.tsx  ← falla el catálogo, header y footer siguen
      <CatalogBoundary> (catchError)  ← falla UNA sección, el resto de la página sigue
```

### 2.2 — `error.tsx` por segmento

Lo vieron en la clase 2. Repaso de reglas:

- Cliente (`'use client'`), recibe `error` (con `digest`) y `reset`.
- Atrapa errores de los hijos, **no** del layout del mismo segmento.
- `reset()` re-renderiza en el cliente. Si el error vino de un Server Component, re-renderizar en el cliente no vuelve a pedir datos al servidor: muchas veces "Reintentar" no hace nada útil.

### 2.3 — `catchError` (16.3): boundaries de componente que reintentan en el servidor

```tsx
// src/components/section-error-boundary.tsx
'use client'

import { catchError, type ErrorInfo } from 'next/error'
import { Button } from '@/components/ui/button'

function SectionError(props: { title: string }, { error, retry }: ErrorInfo) {
  return (
    <div role="alert" className="rounded-lg border border-dashed p-6 text-sm">
      <p className="font-medium">{props.title}</p>
      {process.env.NODE_ENV === 'development' && <p className="text-muted-foreground">{error.message}</p>}
      <Button variant="outline" size="sm" className="mt-3" onClick={() => retry()}>
        Reintentar
      </Button>
    </div>
  )
}

export const SectionErrorBoundary = catchError(SectionError)
```

```tsx
// uso en la página de producto
<SectionErrorBoundary title="No pudimos cargar los relacionados">
  <Suspense fallback={<ProductGridSkeleton count={4} />}>
    <RelatedProducts categoryId={product.categoryId} excludeId={product.id} />
  </Suspense>
</SectionErrorBoundary>
```

Diferencias con un error boundary casero de React:

- `retry()` vuelve a **pedir al servidor** los hijos (incluidos Server Components).
- **No interfiere** con `notFound()` ni `redirect()`: esos siguen funcionando como navegación (un boundary de React común los atrapaba como si fueran errores).

**Validación:** hacé que `listRelated` falle con 50 % de probabilidad. La página carga; la sección muestra el error; "Reintentar" la trae (o vuelve a fallar). El resto de la página nunca se entera.

### 2.4 — Qué error mostrar

| Tipo | Ejemplo | Dónde se maneja |
|---|---|---|
| Esperado de negocio | Sin stock, email inválido | **Valor de retorno** de la action / DTO |
| No encontrado | Slug inexistente | `notFound()` |
| Sin permiso | Producto ajeno | `redirect` / `forbidden()` |
| Inesperado | Base caída, bug | `throw` → boundary + `onRequestError` |

---

## Bloque 3 — `instrumentation.ts` (25 min)

Archivo en `src/instrumentation.ts`. Next lo carga **una vez** al iniciar el servidor.

```ts
// src/instrumentation.ts
import type { Instrumentation } from 'next'

export async function register() {
  // Inicializaciones de servidor: OpenTelemetry, Sentry, validar conexiones...
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    await import('./server/observability/otel')
  }
}

export const onRequestError: Instrumentation.onRequestError = async (err, request, context) => {
  const { logger } = await import('./server/observability/logger')
  const error = err as Error & { digest?: string }

  logger.error({
    msg: 'request_error',
    digest: error.digest,              // el mismo número que ve el usuario
    error: { name: error.name, message: error.message, stack: error.stack },
    request: {
      path: request.path,
      method: request.method,
      requestId: request.headers['x-request-id'], // lo pone el proxy (clase 7)
    },
    context, // routerKind, routePath, routeType ('render' | 'route' | 'action' | 'proxy'), renderSource...
  })
}
```

> "`onRequestError` se llama para **cualquier** error del servidor: render de páginas, Server Actions, Route Handlers, proxy. Con el `digest` del usuario buscan en los logs y tienen el stack, la ruta, el método y el request-id. Se acabó el 'no sé qué pasó'."

### 3.1 — OpenTelemetry (opcional, recomendado)

Next emite spans propios (render de rutas, fetch, etc.). Con `@vercel/otel` se exportan a cualquier backend OTLP (Grafana Tempo, Honeycomb, Datadog):

```ts
// src/server/observability/otel.ts
import { registerOTel } from '@vercel/otel'
registerOTel({ serviceName: 'comarca-web' })
```

(Variables `OTEL_EXPORTER_OTLP_ENDPOINT`, etc. Si la agencia usa Sentry, `npx @sentry/wizard@latest -i nextjs` configura `instrumentation.ts` e `instrumentation-client.ts` por ustedes.)

### 3.2 — Errores del navegador

`src/instrumentation-client.ts` corre en el navegador **antes** de que la app hidrate:

```ts
// src/instrumentation-client.ts
window.addEventListener('error', (e) => {
  navigator.sendBeacon?.('/api/client-errors', JSON.stringify({ message: e.message, path: location.pathname }))
})
window.addEventListener('unhandledrejection', (e) => {
  navigator.sendBeacon?.('/api/client-errors', JSON.stringify({ message: String(e.reason), path: location.pathname }))
})
```

(Con un handler `POST /api/client-errors` con rate limit, clase 18.)

---

## Bloque 4 — Logs estructurados (15 min)

```bash
npm i pino
```

```ts
// src/server/observability/logger.ts
import 'server-only'
import pino from 'pino'

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  base: { service: 'comarca-web', env: process.env.APP_ENV },
  redact: {
    paths: ['*.password', '*.token', '*.authorization', '*.cookie', 'request.headers.cookie', '*.email'],
    censor: '[redacted]',
  },
})
```

Uso en actions y handlers:

```ts
import { headers } from 'next/headers'
import { logger } from '@/server/observability/logger'

export async function placeOrderAction(...) {
  const requestId = (await headers()).get('x-request-id')
  const log = logger.child({ requestId, action: 'placeOrder' })
  // ...
  log.info({ orderId, items: cart.length, totalCents }, 'order_placed')
}
```

Reglas:

- **JSON**, una línea por evento. Coolify / Docker / Loki los indexan.
- **Nunca** contraseñas, tokens, cookies, números de tarjeta. Emails, idealmente tampoco (`redact`).
- Dentro de `'use cache'` un log solo sale en el **miss**: no lo usen para métricas de tráfico.
- Leer `headers()` para el request-id es dato de request: en Server Components, solo donde ya hay contenido dinámico. En actions y handlers, siempre se puede.

---

## Bloque 5 — Testing (55 min)

### 5.1 — La pirámide para una app Next

```text
          ▲  e2e (Playwright): 5–10 flujos críticos + tests de instant()
         ▲▲▲  integración: DAL contra Postgres real, Server Actions, Route Handlers
        ▲▲▲▲▲  unitarios: schemas, utils, reglas de negocio puras
```

> "Los Server Components async son difíciles de testear aislados (las herramientas todavía no los renderizan bien fuera de Next). No peleen con eso: **testeen la DAL** (donde está la lógica y la seguridad) y **los flujos** con Playwright. El componente en el medio es mayormente presentación."

### 5.2 — Configuración de Vitest

```bash
npm i -D vitest @vitejs/plugin-react vite-tsconfig-paths @testing-library/react @testing-library/user-event @testing-library/jest-dom jsdom
```

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [tsconfigPaths(), react()],
  test: {
    environment: 'node',
    setupFiles: ['./test/setup.ts'],
    include: ['src/**/*.test.{ts,tsx}', 'test/**/*.test.{ts,tsx}'],
    pool: 'forks',
    // Los tests de componentes declaran su entorno con el comentario
    // `// @vitest-environment jsdom` en la primera línea del archivo.
  },
})
```

```ts
// test/setup.ts
import { vi } from 'vitest'

// 'server-only' tira error fuera de un Server Component: en tests lo neutralizamos
vi.mock('server-only', () => ({}))

// APIs de cache de Next: no-op en tests ('use cache' es un string, Vitest lo ignora)
vi.mock('next/cache', () => ({
  cacheLife: vi.fn(),
  cacheTag: vi.fn(),
  updateTag: vi.fn(),
  revalidateTag: vi.fn(),
  revalidatePath: vi.fn(),
  refresh: vi.fn(),
}))

// Root params: fijamos el idioma
vi.mock('next/root-params', () => ({ locale: async () => 'es' }))
```

### 5.3 — Unitarios

```ts
// src/lib/safe-redirect.test.ts
import { describe, expect, it } from 'vitest'
import { safeNextPath } from './safe-redirect'

describe('safeNextPath', () => {
  it.each([
    ['/account', '/account'],
    ['https://evil.com', '/'],
    ['//evil.com', '/'],
    ['/\\evil.com', '/'],
    [undefined, '/'],
  ])('%s → %s', (input, expected) => {
    expect(safeNextPath(input as string | undefined)).toBe(expected)
  })
})
```

### 5.4 — DAL contra Postgres de verdad

Base de test aparte en el mismo contenedor:

```sql
-- docker/postgres/init/02-create-test-db.sql
CREATE DATABASE comarca_test OWNER comarca;
```

```bash
# .env.test
DATABASE_URL="postgresql://comarca:comarca@localhost:5432/comarca_test"
```

```json
{
  "scripts": {
    "test": "vitest run",
    "test:db:prepare": "dotenv -e .env.test -- prisma migrate reset --force --skip-seed"
  }
}
```

(`npm i -D dotenv-cli`.)

```ts
// test/factories.ts
import { db } from '@/server/db'

export async function createProducerWithProduct(opts: { email: string; slug: string }) {
  const owner = await db.user.create({ data: { id: crypto.randomUUID(), name: opts.email, email: opts.email, role: 'producer' } })
  const category = await db.category.upsert({ where: { slug: 'quesos' }, update: {}, create: { slug: 'quesos', name: 'Quesos' } })
  const producer = await db.producer.create({ data: { slug: `p-${opts.slug}`, name: 'P', province: 'BA', ownerId: owner.id } })
  const product = await db.product.create({
    data: { slug: opts.slug, name: opts.slug, description: 'x'.repeat(30), priceCents: 1000, stock: 5, imageUrl: '/x.jpg',
            producerId: producer.id, categoryId: category.id },
  })
  return { owner, producer, product }
}
```

```ts
// src/server/dal/producer-products.test.ts
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { db } from '@/server/db'
import { createProducerWithProduct } from '../../../test/factories'

const session = vi.hoisted(() => ({ current: null as null | { id: string; role: string } }))

vi.mock('./session', () => ({
  requireProducer: async () => {
    const p = await db.producer.findUnique({ where: { ownerId: session.current!.id } })
    return { user: session.current, id: p!.id, slug: p!.slug, name: p!.name }
  },
}))

import { setProductPrice } from './producer-products'

describe('setProductPrice', () => {
  beforeEach(async () => {
    await db.orderItem.deleteMany(); await db.product.deleteMany(); await db.producer.deleteMany(); await db.user.deleteMany()
  })

  it('el dueño puede cambiar el precio', async () => {
    const a = await createProducerWithProduct({ email: 'a@test.dev', slug: 'queso-a' })
    session.current = { id: a.owner.id, role: 'producer' }
    await setProductPrice(a.product.id, 2500)
    expect((await db.product.findUnique({ where: { id: a.product.id } }))!.priceCents).toBe(2500)
  })

  it('un productor NO puede cambiar el precio de otro (IDOR)', async () => {
    const a = await createProducerWithProduct({ email: 'a@test.dev', slug: 'queso-a' })
    const b = await createProducerWithProduct({ email: 'b@test.dev', slug: 'queso-b' })
    session.current = { id: b.owner.id, role: 'producer' }

    await expect(setProductPrice(a.product.id, 1)).rejects.toThrow('NOT_FOUND')
    expect((await db.product.findUnique({ where: { id: a.product.id } }))!.priceCents).toBe(1000)
  })
})
```

> "Este test es más valioso que diez tests de componentes. Es la garantía de que el bug más caro de la clase 18 no vuelve."

### 5.5 — Server Actions

Una action es una función async: se llama directo. Hay que simular lo que Next le da (cookies, headers):

```ts
// src/features/cart/actions.test.ts
import { beforeEach, describe, expect, it, vi } from 'vitest'

const jar = vi.hoisted(() => new Map<string, string>())
vi.mock('next/headers', () => ({
  cookies: async () => ({
    get: (k: string) => (jar.has(k) ? { name: k, value: jar.get(k)! } : undefined),
    set: (k: string, v: string) => void jar.set(k, v),
  }),
  headers: async () => new Headers(),
}))

import { addToCart } from './actions'

describe('addToCart', () => {
  beforeEach(() => jar.clear())

  it('rechaza cantidades inválidas sin tocar la cookie', async () => {
    const r = await addToCart({ productId: 'x', quantity: 999 })
    expect(r).toEqual({ ok: false, error: 'invalid' })
    expect(jar.has('cart')).toBe(false)
  })
})
```

### 5.6 — Componentes cliente (lo justo)

```tsx
// @vitest-environment jsdom
// src/features/favorites/components/favorite-button.test.tsx
import '@testing-library/jest-dom/vitest'
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { vi } from 'vitest'

vi.mock('../actions', () => ({ toggleFavorite: vi.fn(() => new Promise(() => {})) })) // nunca resuelve

import { FavoriteButton } from './favorite-button'

it('se marca al instante (optimista)', async () => {
  render(<FavoriteButton productId="p1" isFavorite={false} />)
  await userEvent.click(screen.getByRole('button', { name: 'Favorito' }))
  expect(screen.getByRole('button', { name: 'Favorito' })).toHaveAttribute('aria-pressed', 'true')
})
```

### 5.7 — e2e con Playwright

Además de los tests de `instant()` de la clase 12, el flujo de compra:

```ts
// e2e/checkout.spec.ts
import { expect, test } from '@playwright/test'

test('registro → carrito → checkout', async ({ page }) => {
  const email = `e2e+${Date.now()}@comarca.dev`

  await page.goto('/es/sign-up')
  await page.getByLabel('Nombre').fill('E2E')
  await page.getByLabel('Email').fill(email)
  await page.getByLabel('Contraseña').fill('una-clave-larga-123')
  await page.getByRole('button', { name: 'Crear cuenta' }).click()
  await expect(page).toHaveURL(/\/es\/account/)

  await page.goto('/es/products/miel-monte')
  await page.getByRole('button', { name: 'Agregar al carrito' }).click()
  await expect(page.getByText('Agregado')).toBeVisible()

  await page.goto('/es/cart')
  await page.getByRole('link', { name: 'Finalizar compra' }).click()
  await page.getByRole('button', { name: 'Confirmar pedido' }).click()
  await expect(page).toHaveURL(/\/es\/account\/orders\/.+\?new=1/)
})
```

```ts
// playwright.config.ts (CI contra build de producción)
webServer: {
  command: 'npm run build && npm run start',
  url: 'http://localhost:3000/api/health',
  env: { EXPOSE_TESTING_API: '1', APP_ENV: 'test' },
  timeout: 240_000,
},
```

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| `This module cannot be imported from a Client Component module` en Vitest | `server-only` sin mock | Mock en `setup.ts` |
| "Reintentar" no trae datos nuevos | `reset()` de `error.tsx` solo re-renderiza en cliente | `catchError` con `retry()` |
| El boundary atrapa un `redirect` | Error boundary casero de React | `catchError` |
| Logs con contraseñas | Loguear `formData` entero | `redact` + loguear campos elegidos |
| Tests de DAL que se pisan entre sí | Misma base, en paralelo | `pool: 'forks'` + base de test + limpieza por test |

---

## Tarea

1. `SectionErrorBoundary` en: relacionados, reseñas, rail del CMS y slots del panel.
2. `instrumentation.ts` con `onRequestError` + logger con `redact`. Provocá un error y encontralo en los logs **a partir del digest** que ve el usuario.
3. Tests de DAL: IDOR de productos, idempotencia de `markOrderPaid`, `placeOrderFromCart` sin stock (rollback completo), `listMyOrders` no devuelve pedidos ajenos.
4. e2e del flujo de compra y del flujo del productor (crear producto → aparece en la tienda).
5. Script `npm run test:all` = lint + typecheck + vitest + playwright.

**Terminado cuando:** `npm run test:all` pasa desde cero (`infra:reset` + `test:db:prepare`) y cada error del servidor deja un log con digest y request-id.
