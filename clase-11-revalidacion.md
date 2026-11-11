# Clase 11 — Revalidación: que lo cacheado no mienta

**Duración:** 2:30 hs
**Branch de salida:** `clase-11-revalidacion`
**Lo que se llevan:** elegir entre `updateTag`, `revalidateTag`, `revalidatePath` y `refresh` sabiendo qué ve cada usuario y cuándo; un webhook que invalida desde afuera; una cache compartida con Redis para cuando hay más de una instancia; y la capacidad de leer código del modelo de cache anterior.

**Requisito:** tag `clase-10`.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:10 | El dolor: el productor cambia el precio y la tienda no se entera |
| 0:10 – 0:40 | Las cuatro herramientas y cuándo va cada una |
| 0:40 – 1:10 | **En vivo:** acción de editar precio con `updateTag` |
| 1:10 – 1:30 | Webhook externo con `revalidateTag(tag, 'max')` |
| 1:30 – 1:45 | Qué ve cada usuario: cache del servidor vs cache del navegador |
| 1:45 – 2:15 | Más de una instancia: cache handler con Redis |
| 2:15 – 2:30 | El modelo viejo (para leer código legacy) + tarea |

---

## Bloque 1 — El dolor (10 min)

En Prisma Studio cambiá el precio de `queso-azul` de 12.500 a 14.000. Recargá `/es/products/queso-azul` en `npm run start`: **sigue en 12.500**. Y va a seguir así hasta una hora (perfil `catalog`).

> "La cache sin invalidación es una mentira con fecha de vencimiento. La clase pasada hicimos que la tienda sea rápida; hoy hacemos que sea **correcta**."

---

## Bloque 2 — Las cuatro herramientas (30 min)

Todas se importan de `next/cache`.

### 2.1 — `updateTag(tag)`: "lo cambié yo y lo quiero ver ya"

- **Solo en Server Actions.**
- Expira las entradas con ese tag **inmediatamente** y, en la misma respuesta de la action, el usuario recibe la UI ya regenerada. Semántica *read-your-own-writes*.

```ts
'use server'
import { updateTag } from 'next/cache'

export async function updatePrice(slug: string, priceCents: number) {
  await db.product.update({ where: { slug }, data: { priceCents } })
  updateTag(tags.product(slug))
}
```

### 2.2 — `revalidateTag(tag, perfil)`: "que se actualice, no hay apuro"

- En Server Actions **y** Route Handlers.
- Marca las entradas como viejas con semántica **stale-while-revalidate**: el próximo visitante recibe lo viejo **una vez** y dispara la regeneración; los siguientes ven lo nuevo.
- El segundo argumento es **obligatorio** en Next 16 (la forma de un argumento está deprecada). Es un perfil de `cacheLife` que dice cuánto tiempo se puede seguir sirviendo lo viejo mientras se regenera. `'max'` es el recomendado.

```ts
revalidateTag(tags.catalog, 'max')
```

> "Si ven `revalidateTag('x')` con un solo argumento en un proyecto, es código de 15 que nadie migró. TypeScript ya lo marca."

### 2.3 — `revalidatePath(path)`: "no sé qué tags, invalidá esta URL"

```ts
revalidatePath('/es/products/queso-azul')
revalidatePath('/[locale]/products/[slug]', 'page') // todas las páginas de esa ruta
revalidatePath('/[locale]/(shop)', 'layout')       // todo lo que cuelga de ese layout
```

Funciona porque Next genera **tags implícitos** (soft tags) por cada segmento de la ruta. Es menos preciso: invalida todo lo que se renderizó en esa ruta, incluso lo que no cambió.

> "La doc lo dice: prefieran tags. `revalidatePath` es el martillo para cuando no hay tags o para invalidar una sección entera."

### 2.4 — `refresh()`: "volvé a pedir lo dinámico, sin tocar la cache"

- Solo en Server Actions.
- No invalida ninguna entrada de cache: le pide al router del cliente que vuelva a renderizar la ruta actual, lo que re-ejecuta los huecos dinámicos.
- Para datos **no cacheados** que cambiaron por una action: el contador de notificaciones, el stock en vivo.

```ts
'use server'
import { refresh } from 'next/cache'

export async function markNotificationAsRead(id: string) {
  await db.notification.update({ where: { id }, data: { readAt: new Date() } })
  refresh()
}
```

### 2.5 — La tabla para pegar en el aula

| Situación | Herramienta |
|---|---|
| El usuario edita algo y tiene que verlo al instante (form del panel) | `updateTag` |
| Un sistema externo avisa que algo cambió (webhook, CMS, ERP) | `revalidateTag(tag, 'max')` |
| Cambió algo y no tengo tags para eso | `revalidatePath` |
| Cambió un dato **no cacheado** y quiero que la pantalla actual lo refleje | `refresh` |
| Cambió **todo** (import masivo) | `revalidateTag(tags.catalog, 'max')` |

---

## Bloque 3 — En vivo: editar precio desde el panel (30 min)

> Hoy hacemos una versión mínima de la action para entender la invalidación. En la clase 13 la convertimos en un formulario completo con validación y en la 17 le agregamos autorización.

```ts
// src/server/dal/producer-products.ts
import 'server-only'
import { db } from '@/server/db'

export async function setProductPrice(productId: string, priceCents: number) {
  // ⚠️ falta verificar que el producto sea del productor logueado → clase 17/18
  return db.product.update({
    where: { id: productId },
    data: { priceCents },
    select: { slug: true, producerId: true, category: { select: { slug: true } } },
  })
}
```

```ts
// src/features/producer-dashboard/actions.ts
'use server'

import { updateTag } from 'next/cache'
import { setProductPrice } from '@/server/dal/producer-products'
import { tags } from '@/server/cache-tags'

export async function updatePriceAction(productId: string, formData: FormData) {
  const priceCents = Math.round(Number(formData.get('price')) * 100)
  const updated = await setProductPrice(productId, priceCents)

  // ¿Qué pantallas muestran este precio?
  updateTag(tags.product(updated.slug))             // detalle
  updateTag(tags.category(updated.category.slug))   // listado de la categoría
  updateTag(tags.producerProducts(updated.producerId)) // página del productor y el panel
  // La home (destacados) usa tags.catalog: la dejamos con revalidación en segundo plano
}
```

```tsx
// src/app/(frontend)/[locale]/dashboard/products/[id]/edit/page.tsx (versión mínima)
import { updatePriceAction } from '@/features/producer-dashboard/actions'

export default async function EditProduct(props: PageProps<'/[locale]/dashboard/products/[id]/edit'>) {
  const { id } = await props.params
  const action = updatePriceAction.bind(null, id)
  return (
    <form action={action} className="flex gap-2">
      <input name="price" type="number" step="0.01" className="rounded border px-2" />
      <button className="rounded bg-primary px-3 py-1 text-primary-foreground">Guardar</button>
    </form>
  )
}
```

(`instant = false` en el layout del panel por ahora: todavía no tiene auth ni boundaries; lo arreglamos en la 17.)

**Validación:**

1. `npm run build && npm run start`.
2. Ventana A: `/es/products/queso-azul`. Ventana B: el form de edición.
3. Cambiá el precio en B. Recargá A → **precio nuevo**.
4. `/es/categories/quesos` → precio nuevo.
5. Home → puede mostrar el viejo un rato (no lo invalidamos a propósito).

> "Elegimos tags **específicos**. Podríamos haber hecho `updateTag(tags.catalog)` y listo: invalida todo el catálogo. Funciona, pero cada edición de un productor regenera la home, todas las categorías, todos los productos. Con 300 productores editando, la cache no sirve para nada. Tags específicos = invalidar lo justo."

### 3.1 — ¿Y si me olvido de un tag?

Va a pasar. Por eso:

- Los perfiles tienen un `expire` razonable (el precio igual se corrige en una hora como máximo).
- Centralizar la lista de tags a invalidar **por tipo de cambio** en la DAL, no en cada action:

```ts
// src/server/dal/producer-products.ts
export function tagsAffectedByProduct(p: { slug: string; producerId: string; categorySlug: string }) {
  return [tags.product(p.slug), tags.category(p.categorySlug), tags.producerProducts(p.producerId)]
}
```

```ts
for (const t of tagsAffectedByProduct(updated)) updateTag(t)
```

---

## Bloque 4 — Webhook externo (20 min)

Caso: una cooperativa sincroniza su stock y precios desde su ERP. Cuando termina, nos llama.

```ts
// src/app/api/webhooks/catalog-sync/route.ts
import { revalidateTag } from 'next/cache'
import { timingSafeEqual } from 'node:crypto'
import { tags } from '@/server/cache-tags'

function validSecret(received: string | null) {
  const expected = process.env.CATALOG_WEBHOOK_SECRET
  if (!received || !expected) return false
  const a = Buffer.from(received)
  const b = Buffer.from(expected)
  return a.length === b.length && timingSafeEqual(a, b)
}

export async function POST(request: Request) {
  if (!validSecret(request.headers.get('x-webhook-secret'))) {
    return Response.json({ error: 'unauthorized' }, { status: 401 })
  }

  const body = (await request.json()) as { productSlugs?: string[]; full?: boolean }

  if (body.full) {
    revalidateTag(tags.catalog, 'max')
  } else {
    for (const slug of body.productSlugs ?? []) revalidateTag(tags.product(slug), 'max')
  }

  return Response.json({ revalidated: true, at: Date.now() })
}
```

```bash
curl -X POST localhost:3000/api/webhooks/catalog-sync \
  -H "x-webhook-secret: $CATALOG_WEBHOOK_SECRET" -H 'content-type: application/json' \
  -d '{"productSlugs":["queso-azul"]}'
```

> "¿Por qué `revalidateTag` y no `updateTag` acá? Primero, porque `updateTag` no existe en Route Handlers. Segundo, porque no hay un usuario esperando ver su cambio: con stale-while-revalidate, nadie espera; el primer visitante dispara la regeneración y recibe la versión anterior."

(`timingSafeEqual` evita que se pueda adivinar el secreto midiendo tiempos de respuesta. Lo profundizamos en la clase 16 con firmas HMAC.)

---

## Bloque 5 — Qué ve cada usuario (15 min)

Hay **dos** caches en juego y hay que tenerlas claras:

```text
            ┌──────────────── servidor ────────────────┐
Usuario A ──▶ cache 'use cache' (revalidate / expire)  │
            └──────────────────────────────────────────┘
   ▲
   │ RSC payload
   │
 router del navegador de A (stale)
```

- `updateTag` / `revalidateTag` actúan sobre la cache **del servidor**.
- El usuario que ejecutó la action recibe la UI fresca en la respuesta: su navegador queda al día.
- **Otro usuario** que ya tenía la página en su router puede seguir viendo la versión vieja durante el `stale` del perfil (5 minutos en la mayoría) si navega sin recargar.

> "Por eso el perfil `price` tiene `stale: 60`. Para el catálogo general, 5 minutos de navegador viejo es aceptable. Para el precio, preferimos un minuto. Y para el checkout no importa: el precio final lo recalcula el servidor en la transacción."

---

## Bloque 6 — Más de una instancia: cache handler con Redis (30 min)

### 6.1 — El problema

En producción (clase 22) vamos a correr 2 contenedores detrás de un balanceador. La cache por defecto es **memoria de cada proceso**:

- El contenedor 1 regenera el producto; el 2 sigue con el viejo.
- `updateTag` en el contenedor 1 **no se entera** el 2.

### 6.2 — `cacheHandlers`

Next permite reemplazar dónde se guarda la cache de `'use cache'` (`default`) y de `'use cache: remote'` (`remote`):

```ts
// next.config.ts
const nextConfig: NextConfig = {
  // ...
  cacheHandlers: {
    default: require.resolve('./cache-handlers/redis-handler.cjs'),
    remote: require.resolve('./cache-handlers/redis-handler.cjs'),
  },
}
```

Solo en producción (en dev dejá el de memoria):

```ts
const useRedisCache = process.env.NODE_ENV === 'production' && !!process.env.REDIS_URL

const nextConfig: NextConfig = {
  // ...
  ...(useRedisCache && {
    cacheHandlers: {
      default: require.resolve('./cache-handlers/redis-handler.cjs'),
      remote: require.resolve('./cache-handlers/redis-handler.cjs'),
    },
  }),
}
```

### 6.3 — El handler

Un handler implementa cinco métodos. Este está armado con los ejemplos de la doc oficial (entradas + coordinación de tags):

```js
// cache-handlers/redis-handler.cjs
const { createClient } = require('redis')

const client = createClient({ url: process.env.REDIS_URL })
const ready = client.connect()

// Copia local de "cuándo se invalidó cada tag", sincronizada en refreshTags()
const tagTimestamps = new Map()

module.exports = {
  async get(cacheKey) {
    try {
      await ready
      const stored = await client.get(`next:entry:${cacheKey}`)
      if (!stored) return undefined
      const data = JSON.parse(stored)

      // ¿Algún tag de esta entrada se invalidó después de crearla?
      for (const tag of data.tags) {
        const invalidatedAt = tagTimestamps.get(tag) ?? 0
        if (invalidatedAt > data.timestamp) return undefined
      }

      return {
        value: new ReadableStream({
          start(controller) {
            controller.enqueue(Buffer.from(data.value, 'base64'))
            controller.close()
          },
        }),
        tags: data.tags,
        stale: data.stale,
        timestamp: data.timestamp,
        expire: data.expire,
        revalidate: data.revalidate,
      }
    } catch {
      return undefined // un error de Redis nunca debe romper el render: es un miss
    }
  },

  async set(cacheKey, pendingEntry) {
    try {
      await ready
      const entry = await pendingEntry
      const bytes = Buffer.from(await new Response(entry.value).arrayBuffer())
      await client.set(
        `next:entry:${cacheKey}`,
        JSON.stringify({
          value: bytes.toString('base64'),
          tags: entry.tags,
          stale: entry.stale,
          timestamp: entry.timestamp,
          expire: entry.expire,
          revalidate: entry.revalidate,
        }),
        Number.isFinite(entry.expire) ? { EX: Math.max(1, Math.ceil(entry.expire)) } : {},
      )
    } catch (err) {
      console.error('[cache] set failed', err)
    }
  },

  async refreshTags() {
    await ready
    const tagNames = await client.sMembers('next:revalidated-tags')
    if (tagNames.length === 0) return
    const values = await client.mGet(tagNames.map((t) => `next:tag:${t}`))
    tagNames.forEach((t, i) => tagTimestamps.set(t, Number(values[i] ?? 0)))
  },

  async getExpiration(tags) {
    return Math.max(0, ...tags.map((t) => tagTimestamps.get(t) ?? 0))
  },

  async updateTags(tags) {
    await ready
    const now = Date.now()
    const multi = client.multi()
    for (const tag of tags) {
      multi.set(`next:tag:${tag}`, String(now))
      multi.sAdd('next:revalidated-tags', tag)
      tagTimestamps.set(tag, now)
    }
    await multi.exec()
  },
}
```

```bash
npm i redis
```

Puntos para explicar mientras lo escribís:

- **`set` recibe una promesa** y un `ReadableStream` que **se puede leer una sola vez**. Lo consumimos y guardamos bytes, nunca el stream.
- **`get` arma un stream nuevo** en cada hit.
- **Nunca tirar errores desde `get`**: Next no lo envuelve en try/catch; un error ahí rompe la página. Ante cualquier problema, `undefined` (miss).
- **TTL de Redis = `expire`**: Next nunca borra; la limpieza es nuestra.
- **`updateTags`** guarda el momento de invalidación en Redis; **`refreshTags`** (se llama una vez por request antes de leer) trae las invalidaciones que hicieron **otras** instancias.
- Esta versión simplifica: trata toda invalidación como inmediata. La doc muestra cómo distinguir `revalidateTag(tag, 'max')` (stale) de `updateTag` (expirado) usando el segundo argumento `durations` de `updateTags`. Para producción seria, partan del ejemplo oficial `examples/cache-handler-redis` del repo de Next.

**Validación (con dos instancias):**

```bash
npm run build
PORT=3001 REDIS_URL=redis://localhost:6379 npm run start &
PORT=3002 REDIS_URL=redis://localhost:6379 npm run start &
```

Editá el precio a través de `:3001` y recargá el producto en `:3002` → precio nuevo. Sin el handler, `:3002` mostraba el viejo.

```bash
docker exec -it comarca-redis-1 redis-cli --scan --pattern 'next:*' | head
```

> "Recuerden: aunque la cache esté en Redis, **cada deploy empieza de cero** (la clave incluye el build id). Esto es intencional: entre builds puede cambiar la forma de los datos."

---

## Bloque 7 — El modelo viejo, para leer código legacy (10 min)

Van a encontrar proyectos 13–15 (o 16 sin `cacheComponents`). Equivalencias:

| Modelo viejo | Qué hacía | En Cache Components |
|---|---|---|
| `fetch(url)` cacheado por defecto (13–14) | Cache implícita del fetch | `'use cache'` en la función que hace el fetch |
| `fetch(url, { next: { revalidate: 60 } })` | Revalidación por tiempo | `cacheLife({ revalidate: 60 })` |
| `fetch(url, { next: { tags: ['x'] } })` | Tag | `cacheTag('x')` |
| `unstable_cache(fn, keys, { tags, revalidate })` | Cachear funciones no-fetch | `'use cache'` + `cacheTag` + `cacheLife` |
| `export const revalidate = 60` | ISR por ruta | `cacheLife` en los componentes |
| `export const dynamic = 'force-dynamic'` | Todo dinámico | `connection()` dentro de Suspense donde haga falta |
| `export const dynamic = 'force-static'` | Forzar estático | `ensureStatic` (16.4) |
| `revalidateTag('x')` | Invalidar | `revalidateTag('x', 'max')` o `updateTag('x')` |

> "La doc mantiene una guía 'Caching (Previous Model)' y otra 'Migrating to Cache Components'. Si les toca migrar un cliente de la agencia, existe `npx next@canary upgrade --agent` y skills oficiales para que un agente haga el trabajo pesado. Ustedes revisan."

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| `updateTag can only be called from a Server Action` | Lo usaron en un Route Handler | `revalidateTag(tag, 'max')` |
| Edito y no cambia | El tag invalidado no es el que tiene la entrada | Revisar `cache-tags.ts`; loguear en la DAL |
| Edito y cambia en una instancia sí y en otra no | Cache en memoria por proceso | Cache handler compartido |
| Una página rompe cuando Redis está caído | `get` tira excepción | `try/catch` → `undefined` |
| Otro usuario sigue viendo lo viejo sin recargar | `stale` del cliente | Bajar `stale` en ese perfil |

---

## Tarea

1. `tagsAffectedByProduct` en la DAL y usarla en la action de precio.
2. Action "activar/desactivar producto" que invalide también `tags.catalog` (desaparece de la home).
3. Webhook `catalog-sync` con test (Vitest) para: secreto inválido → 401, `full: true` llama a `revalidateTag` con `tags.catalog` (mockeá `next/cache`).
4. Levantá dos instancias con el handler de Redis y grabá un video corto mostrando la invalidación cruzada.

**Terminado cuando:** cambiar un precio desde el panel se ve al instante en detalle, categoría y productor, y la home se actualiza sola dentro de su perfil.
