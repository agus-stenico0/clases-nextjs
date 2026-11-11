# Clase 16 — Route Handlers

**Duración:** 2:30 hs
**Branch de salida:** `clase-16-route-handlers`
**Lo que se llevan:** cuándo un Route Handler y cuándo una Server Action; una API pública versionada para la app móvil con validación, paginación, CORS y cache; el webhook de pagos firmado e idempotente; exportación CSV en streaming; Server-Sent Events para el estado del pedido; y un health check honesto.

**Requisito:** tag `clase-15`.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:10 | El dolor: la app móvil no puede llamar a una Server Action |
| 0:10 – 0:30 | `route.ts`: anatomía, tipos, y cómo se comporta con Cache Components |
| 0:30 – 1:05 | **En vivo:** API pública `/api/v1/products` |
| 1:05 – 1:40 | **En vivo:** webhook de pagos firmado e idempotente |
| 1:40 – 2:00 | Streaming: CSV de pedidos y Server-Sent Events |
| 2:00 – 2:15 | Health check y la tabla handler vs action |
| 2:15 – 2:30 | Tarea |

---

## Bloque 1 — El dolor (10 min)

> "La agencia también hace la app de Comarca en React Native. La app necesita el catálogo. ¿Le pasamos el ID de una Server Action? No: las actions son un protocolo interno de React, con IDs que cambian en cada build. Para hablar con el mundo de afuera —apps, MercadoPago, un ERP, un cron— necesitamos HTTP de verdad. Eso son los Route Handlers."

---

## Bloque 2 — Anatomía (20 min)

```ts
// src/app/api/health/route.ts
export async function GET(request: Request) {
  return Response.json({ ok: true })
}
```

- Archivo `route.ts` en cualquier carpeta de `app/`. **No puede convivir** con un `page.tsx` en el mismo segmento.
- Se exporta una función por método: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`. Un método no exportado → 405 automático.
- Usa la API estándar web: `Request`, `Response`, `Headers`, `URL`. `NextRequest` agrega `nextUrl` y `cookies`.
- Params dinámicos: segundo argumento, **promesa**, tipado con `RouteContext`:

```ts
// src/app/api/v1/products/[slug]/route.ts
export async function GET(_req: Request, ctx: RouteContext<'/api/v1/products/[slug]'>) {
  const { slug } = await ctx.params
  // ...
}
```

### 2.1 — Con Cache Components

Un `GET` sigue el **mismo modelo que una página**:

- Si no lee nada dinámico (ni `request`, ni cookies, ni datos sin cache) → se **prerenderiza en build** y se sirve estático.
- Si lee el request o datos sin cache → corre por request.
- Para cachear un pedazo: `'use cache'` en la función que obtiene los datos (no en el handler).

```ts
// src/app/api/v1/categories/route.ts
import { listCategories } from '@/server/dal/catalog' // tiene 'use cache'

export async function GET() {
  const categories = await listCategories()
  return Response.json({ data: categories })
}
```

`npm run build` → `○ /api/v1/categories`. Es un JSON estático que se regenera cuando se invalida el tag `catalog`.

> "¿Se acuerdan del matcher del proxy? Excluimos `/api`. Los handlers se protegen solos: auth, rate limit y validación van adentro de cada uno (o en un helper)."

---

## Bloque 3 — En vivo: API pública para la app (35 min)

### 3.1 — Contrato

```text
GET /api/v1/products?category=quesos&locale=es&cursor=<id>&limit=20
200 { data: ProductCardDTO[], nextCursor: string | null }
400 { error: { code: 'invalid_query', issues: [...] } }
```

> "Versionamos desde el día uno (`/v1`). Una app móvil vieja va a seguir llamando a este endpoint durante años: no lo pueden cambiar sin romper usuarios que no actualizaron."

### 3.2 — Helpers de respuesta

```ts
// src/server/http.ts
import 'server-only'
import type { ZodError } from 'zod'

export const ALLOWED_ORIGINS = (process.env.API_ALLOWED_ORIGINS ?? '').split(',').filter(Boolean)

export function corsHeaders(origin: string | null): HeadersInit {
  if (!origin || !ALLOWED_ORIGINS.includes(origin)) return {}
  return {
    'Access-Control-Allow-Origin': origin,
    'Access-Control-Allow-Methods': 'GET, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    Vary: 'Origin',
  }
}

export function badRequest(error: ZodError, origin: string | null) {
  return Response.json(
    { error: { code: 'invalid_query', issues: error.issues.map((i) => ({ path: i.path, message: i.message })) } },
    { status: 400, headers: corsHeaders(origin) },
  )
}
```

> "La app nativa no necesita CORS (CORS es una regla del navegador). Lo agregamos para el panel web de un partner que consume la API desde otro dominio. Lista blanca explícita: nunca `Access-Control-Allow-Origin: *` en una API con datos de usuarios."

### 3.3 — DAL paginada

```ts
// src/server/dal/catalog.ts
export async function listProductsPage(params: {
  categorySlug?: string
  locale: string
  cursor?: string
  limit: number
}) {
  'use cache'
  cacheLife('catalog')
  cacheTag(tags.catalog, ...(params.categorySlug ? [tags.category(params.categorySlug)] : []))

  const rows = await db.product.findMany({
    where: { isActive: true, ...(params.categorySlug && { category: { slug: params.categorySlug } }) },
    select: { id: true, ...cardSelect, translations: { where: { locale: params.locale }, select: { name: true } } },
    orderBy: { id: 'asc' },
    take: params.limit + 1, // uno de más para saber si hay siguiente página
    ...(params.cursor && { cursor: { id: params.cursor }, skip: 1 }),
  })

  const hasMore = rows.length > params.limit
  const page = rows.slice(0, params.limit)
  return { data: page.map(toCardDTO), nextCursor: hasMore ? page.at(-1)!.id : null }
}
```

> "Acá el idioma entra como **argumento** y no con `next/root-params`: los Route Handlers no tienen root params (la API no vive bajo `[locale]`). El argumento entra en la clave de cache igual."

> Ojo: el objeto `params` se serializa como clave. Con `'use cache'` está bien (la clave se calcula por valor, no por identidad, a diferencia de `React.cache`).

### 3.4 — El handler

```ts
// src/app/api/v1/products/route.ts
import { z } from 'zod'
import type { NextRequest } from 'next/server'
import { listProductsPage } from '@/server/dal/catalog'
import { badRequest, corsHeaders } from '@/server/http'
import { routing } from '@/i18n/routing'

const querySchema = z.object({
  category: z.string().max(40).optional(),
  locale: z.enum(routing.locales).default(routing.defaultLocale),
  cursor: z.string().max(40).optional(),
  limit: z.coerce.number().int().min(1).max(50).default(20),
})

export async function GET(request: NextRequest) {
  const origin = request.headers.get('origin')
  const parsed = querySchema.safeParse(Object.fromEntries(request.nextUrl.searchParams))
  if (!parsed.success) return badRequest(parsed.error, origin)

  const { category, locale, cursor, limit } = parsed.data
  const result = await listProductsPage({ categorySlug: category, locale, cursor, limit })

  return Response.json(result, {
    headers: {
      ...corsHeaders(origin),
      // Cache en CDN/proxy intermedio: 60 s frescos, 5 min sirviendo viejo mientras revalida
      'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=300',
    },
  })
}

export function OPTIONS(request: NextRequest) {
  return new Response(null, { status: 204, headers: corsHeaders(request.headers.get('origin')) })
}
```

**Validación:**

```bash
curl -s 'localhost:3000/api/v1/products?limit=3' | jq
curl -s "localhost:3000/api/v1/products?limit=3&cursor=$(curl -s 'localhost:3000/api/v1/products?limit=3' | jq -r .nextCursor)" | jq '.data[].slug'
curl -s 'localhost:3000/api/v1/products?limit=999' | jq .error.code   # invalid_query
curl -si -X OPTIONS -H 'Origin: https://partner.example.com' localhost:3000/api/v1/products | head -8
```

---

## Bloque 4 — En vivo: webhook de pagos (35 min)

Flujo con MercadoPago (Checkout Pro):

```text
1. placeOrderAction crea la orden (pending) y una "preferencia" en MP con external_reference = orderId
2. El usuario paga en MP
3. MP llama a POST /api/webhooks/mercadopago con { type: 'payment', data: { id } }
4. Verificamos la FIRMA del webhook
5. Pedimos el pago a la API de MP (nunca confiamos en el body)
6. Si está aprobado: orden → paid, idempotente
```

> Si la agencia ya tiene un core de pagos propio (otro servicio), el patrón es idéntico: el que firma el webhook es el core y Next verifica esa firma.

### 4.1 — Verificar la firma

MercadoPago manda `x-signature: ts=<timestamp>,v1=<hmac>` y `x-request-id`. El HMAC SHA-256 se calcula sobre un "manifest" con el id del recurso, el request id y el timestamp, usando la clave secreta del webhook.

```ts
// src/server/payments/mercadopago-signature.ts
import 'server-only'
import { createHmac, timingSafeEqual } from 'node:crypto'

export function verifyMercadoPagoSignature(params: {
  signatureHeader: string | null
  requestId: string | null
  dataId: string
  secret: string
  toleranceSeconds?: number
}) {
  const { signatureHeader, requestId, dataId, secret, toleranceSeconds = 300 } = params
  if (!signatureHeader || !requestId) return false

  const parts = Object.fromEntries(
    signatureHeader.split(',').map((p) => p.trim().split('=') as [string, string]),
  )
  const ts = parts.ts
  const v1 = parts.v1
  if (!ts || !v1) return false

  // Anti-replay: rechazar firmas viejas. (Verificá en la doc de MP si ts viene en segundos o ms.)
  const tsSeconds = ts.length > 10 ? Math.floor(Number(ts) / 1000) : Number(ts)
  if (Math.abs(Date.now() / 1000 - tsSeconds) > toleranceSeconds) return false

  const manifest = `id:${dataId.toLowerCase()};request-id:${requestId};ts:${ts};`
  const expected = createHmac('sha256', secret).update(manifest).digest('hex')

  const a = Buffer.from(expected, 'hex')
  const b = Buffer.from(v1, 'hex')
  return a.length === b.length && timingSafeEqual(a, b)
}
```

> "Verifiquen el formato exacto del manifest contra la doc vigente de MercadoPago antes de ir a producción: los proveedores cambian detalles. Lo que **no** cambia es el patrón: HMAC con secreto compartido, comparación en tiempo constante, ventana de tiempo contra replays."

### 4.2 — El handler

```ts
// src/app/api/webhooks/mercadopago/route.ts
import { updateTag } from 'next/cache'  // ❌ no: updateTag es solo para Server Actions
```

> "Trampa a propósito. Acá no hay usuario esperando ver su cambio: `revalidateTag(tag, 'max')`."

```ts
// src/app/api/webhooks/mercadopago/route.ts
import { revalidateTag } from 'next/cache'
import { verifyMercadoPagoSignature } from '@/server/payments/mercadopago-signature'
import { fetchMercadoPagoPayment } from '@/server/payments/mercadopago-client'
import { markOrderPaid } from '@/server/dal/orders'
import { tags } from '@/server/cache-tags'

export async function POST(request: Request) {
  const url = new URL(request.url)
  const body = (await request.json().catch(() => null)) as { type?: string; data?: { id?: string } } | null
  const dataId = body?.data?.id ?? url.searchParams.get('data.id')

  if (body?.type !== 'payment' || !dataId) {
    return new Response(null, { status: 200 }) // otros eventos: ack y listo
  }

  const valid = verifyMercadoPagoSignature({
    signatureHeader: request.headers.get('x-signature'),
    requestId: request.headers.get('x-request-id'),
    dataId,
    secret: process.env.MP_WEBHOOK_SECRET!,
  })
  if (!valid) return new Response('invalid signature', { status: 401 })

  // Fuente de verdad: la API de MP, no el body del webhook
  const payment = await fetchMercadoPagoPayment(dataId)
  if (payment.status !== 'approved') return new Response(null, { status: 200 })

  const result = await markOrderPaid({
    orderId: payment.external_reference,
    paymentId: String(payment.id),
    amountCents: Math.round(payment.transaction_amount * 100),
  })

  if (result.kind === 'updated') revalidateTag(tags.orders(result.userId), 'max') // agregá orders: (id) => `orders:${id}` a cache-tags.ts
  return new Response(null, { status: 200 })
}
```

```ts
// src/server/dal/orders.ts
export async function markOrderPaid(input: { orderId: string; paymentId: string; amountCents: number }) {
  // Idempotente: si MP reintenta el webhook (lo hace), no pasa nada.
  const order = await db.order.findUnique({ where: { id: input.orderId }, select: { status: true, totalCents: true, userId: true } })
  if (!order) return { kind: 'not_found' as const }
  if (order.status !== 'pending') return { kind: 'already_processed' as const, userId: order.userId }
  if (order.totalCents !== input.amountCents) {
    // Pagó otro monto: no marcamos como pagado. Alerta para revisión manual.
    console.error('[payments] amount mismatch', input)
    return { kind: 'amount_mismatch' as const, userId: order.userId }
  }

  const res = await db.order.updateMany({
    where: { id: input.orderId, status: 'pending' },          // condición = candado optimista
    data: { status: 'paid', paymentId: input.paymentId },     // paymentId @unique en el schema
  })
  return { kind: res.count === 1 ? ('updated' as const) : ('already_processed' as const), userId: order.userId }
}
```

Tres capas de idempotencia, en voz alta:

1. `status !== 'pending'` → ya procesado, respondemos 200.
2. `updateMany ... where status: 'pending'` → si dos webhooks llegan juntos, uno solo actualiza.
3. `paymentId @unique` → la base rechaza un segundo pago con el mismo ID.

> "**Siempre** respondan 200 a lo que ya procesaron. Si responden 500, MP reintenta durante horas. Y respondan rápido: si el procesamiento es pesado (mandar mails, facturar), encolen y respondan. En el módulo de NestJS vieron colas: en una arquitectura real, este handler publica un evento y el core lo procesa."

**Validación local:** con `ngrok http 3000` (o el túnel que usen) configurado como URL de webhook en el panel de MP sandbox, hacé un pago de prueba. Para tests sin MP: un test de Vitest que genere una firma válida con el mismo secreto y verifique 200, y una con un byte cambiado → 401.

---

## Bloque 5 — Streaming (20 min)

### 5.1 — Exportar pedidos a CSV sin cargar todo en memoria

```ts
// src/app/api/dashboard/orders.csv/route.ts
import { requireProducer } from '@/server/dal/session' // clase 17
import { iterateProducerOrderLines } from '@/server/dal/producer-orders'

export async function GET() {
  const producer = await requireProducer() // auth ADENTRO del handler

  const encoder = new TextEncoder()
  const stream = new ReadableStream({
    async start(controller) {
      controller.enqueue(encoder.encode('order_id,fecha,producto,cantidad,precio_unitario\n'))
      for await (const batch of iterateProducerOrderLines(producer.id, 500)) {
        const chunk = batch
          .map((l) => [l.orderId, l.createdAt.toISOString(), csv(l.productName), l.quantity, l.unitPriceCents / 100].join(','))
          .join('\n')
        controller.enqueue(encoder.encode(chunk + '\n'))
      }
      controller.close()
    },
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/csv; charset=utf-8',
      'Content-Disposition': `attachment; filename="pedidos-${new Date().toISOString().slice(0, 10)}.csv"`,
      'Cache-Control': 'no-store',
    },
  })
}

const csv = (s: string) => `"${s.replaceAll('"', '""')}"`
```

```ts
// src/server/dal/producer-orders.ts — generador por lotes con cursor
export async function* iterateProducerOrderLines(producerId: string, batchSize: number) {
  let cursor: string | undefined
  while (true) {
    const rows = await db.orderItem.findMany({
      where: { product: { producerId } },
      select: { id: true, orderId: true, quantity: true, unitPriceCents: true,
                order: { select: { createdAt: true } }, product: { select: { name: true } } },
      orderBy: { id: 'asc' },
      take: batchSize,
      ...(cursor && { cursor: { id: cursor }, skip: 1 }),
    })
    if (rows.length === 0) return
    yield rows.map((r) => ({ orderId: r.orderId, createdAt: r.order.createdAt, productName: r.product.name,
                             quantity: r.quantity, unitPriceCents: r.unitPriceCents }))
    cursor = rows.at(-1)!.id
  }
}
```

> "Con 200.000 líneas, armar el CSV entero en memoria tumba el proceso. Con streaming, la memoria es constante y el navegador empieza a descargar en el primer milisegundo."

### 5.2 — Server-Sent Events: estado del pedido en vivo

```ts
// src/app/api/orders/[id]/events/route.ts
import { getOrderStatusForCurrentUser } from '@/server/dal/orders'

export async function GET(request: Request, ctx: RouteContext<'/api/orders/[id]/events'>) {
  const { id } = await ctx.params
  const encoder = new TextEncoder()
  let last = ''

  const stream = new ReadableStream({
    async start(controller) {
      const send = (event: string, data: unknown) =>
        controller.enqueue(encoder.encode(`event: ${event}\ndata: ${JSON.stringify(data)}\n\n`))

      while (!request.signal.aborted) {
        const status = await getOrderStatusForCurrentUser(id) // verifica que el pedido sea del usuario
        if (status !== last) {
          send('status', { status })
          last = status
          if (status === 'delivered' || status === 'cancelled') break
        }
        await new Promise((r) => setTimeout(r, 5000))
      }
      controller.close()
    },
  })

  return new Response(stream, {
    headers: { 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache, no-transform', Connection: 'keep-alive' },
  })
}
```

```tsx
'use client'
import { useEffect, useState } from 'react'

export function OrderStatusLive({ orderId, initial }: { orderId: string; initial: string }) {
  const [status, setStatus] = useState(initial)
  useEffect(() => {
    const es = new EventSource(`/api/orders/${orderId}/events`)
    es.addEventListener('status', (e) => setStatus(JSON.parse((e as MessageEvent).data).status))
    return () => es.close()
  }, [orderId])
  return <span className="rounded bg-muted px-2 py-1 text-sm">{status}</span>
}
```

> "Esto es polling del lado del servidor, honesto y simple. Para escala o tiempo real de verdad, WebSockets en un servicio aparte (lo vieron en NestJS) o Redis pub/sub. Y ojo con hosting serverless: conexiones largas cuestan plata o se cortan. En nuestro deploy con Docker (clase 22) no hay problema."

---

## Bloque 6 — Health check y la tabla de decisión (15 min)

```ts
// src/app/api/health/route.ts
import { connection } from 'next/server'
import { db } from '@/server/db'

export async function GET() {
  await connection() // sin esto, Next lo prerenderiza en build y siempre diría "ok"
  try {
    await db.$queryRaw`SELECT 1`
    return Response.json({ status: 'ok', db: 'up' })
  } catch {
    return Response.json({ status: 'degraded', db: 'down' }, { status: 503 })
  }
}
```

> "Esta es la demostración perfecta del modelo de la clase 9 aplicado a handlers. Un health check estático es una mentira: el build lo generó cuando la base andaba."

| Necesito... | Route Handler | Server Action |
|---|---|---|
| Que lo llame algo que no es mi UI de React (app, webhook, cron, partner) | ✅ | ❌ |
| Mutación disparada desde mi propia UI | posible, pero más código | ✅ |
| GET cacheable por URL / CDN | ✅ | ❌ (siempre POST) |
| Descargar un archivo, streaming, SSE | ✅ | ❌ |
| Progressive enhancement de formularios | ❌ | ✅ |
| Contrato HTTP estable y versionado | ✅ | ❌ (IDs cambian por build) |

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| El health check siempre dice ok | GET prerenderizado en build | `await connection()` |
| `updateTag can only be called from a Server Action` | Usado en un handler | `revalidateTag(tag, 'max')` |
| MP reintenta el webhook sin parar | Respondemos 4xx/5xx a eventos ya procesados | 200 idempotente |
| CSV que tumba el server | Todo en memoria | `ReadableStream` + lotes |
| CORS funciona en Postman y no en el navegador | Falta `OPTIONS` o `Vary: Origin` | Handler `OPTIONS` + headers |
| `Conflicting route and page` | `route.ts` y `page.tsx` en el mismo segmento | Mover uno |

---

## Tarea

1. `/api/v1/products/[slug]` con 404 JSON, y `/api/v1/categories` estático; tests de Vitest llamando a las funciones `GET` directamente con un `Request`.
2. Webhook de MercadoPago con tests de firma (válida, inválida, vencida) e idempotencia (dos llamadas → una actualización).
3. CSV del panel con 10.000 líneas generadas por un seed y medición de memoria (`process.memoryUsage()` antes/después).
4. Componente `OrderStatusLive` en el detalle del pedido, y un script que cambie el estado del pedido cada 10 s para demostrarlo.

**Terminado cuando:** todos los tests pasan y el webhook procesa correctamente un pago de prueba en el sandbox de MercadoPago (o, sin cuenta de MP, el test de integración con firma generada).
