# Clase 10 — `'use cache'` a fondo

**Duración:** 2:30 hs
**Branch de salida:** `clase-10-use-cache`
**Lo que se llevan:** cachear datos y UI con `'use cache'`, elegir y definir perfiles con `cacheLife`, etiquetar con `cacheTag` para poder invalidar (clase 11), entender qué entra en la clave de cache y qué no se puede cachear, y cuándo usar las variantes `private` y `remote`. Al final, la página de producto queda con casi todo en el static shell.

**Requisito:** tag `clase-09`. La tabla de inventario de la clase 9 a mano.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:10 | El dolor: 30 queries por visita a la home |
| 0:10 – 0:35 | La directiva: dónde se pone y qué hace |
| 0:35 – 1:00 | `cacheLife`: stale, revalidate, expire y perfiles propios |
| 1:00 – 1:15 | `cacheTag`: convención de tags de Comarca |
| 1:15 – 1:40 | Claves de cache, serialización y lo que NO se puede hacer adentro |
| 1:40 – 2:10 | **En vivo:** la DAL y la página de producto cacheadas |
| 2:10 – 2:25 | `use cache: private` y `use cache: remote` |
| 2:25 – 2:30 | Tarea |

---

## Bloque 1 — El dolor (10 min)

Con el log de Prisma activo, recargá la home 5 veces en `npm run start` (producción local). Contá las queries: categorías, destacados, productores, banner... **cada visita las repite todas**, y todo eso está en huecos dinámicos porque son datos "no cacheados".

> "La home muestra lo mismo a todo el mundo durante horas. Recalcularla en cada visita es pagar 10.000 veces por la misma respuesta. Y peor: como son huecos, el usuario ve skeletons en vez de productos. Hoy pasamos todo eso al shell."

---

## Bloque 2 — La directiva (25 min)

`'use cache'` le dice a Next: **el resultado de esta función async depende solo de sus argumentos; guardalo y reusalo**.

### 2.1 — Tres lugares donde se puede poner

```ts
// 1. Nivel función (data-level): cachea un dato
export async function listCategories() {
  'use cache'
  return db.category.findMany({ orderBy: { name: 'asc' } })
}
```

```tsx
// 2. Nivel componente (UI-level): cachea el resultado renderizado
export async function FeaturedProducts() {
  'use cache'
  const products = await listFeatured()
  return <ProductRow products={products} />
}
```

```ts
// 3. Nivel archivo: todas las funciones exportadas del archivo quedan cacheadas
'use cache'

export async function a() { /* ... */ }
export async function b() { /* ... */ }
```

> "Nivel archivo casi nunca: es fácil que alguien agregue una función que no debía cachearse. Nivel función para datos que se usan en varios lugares. Nivel componente cuando lo caro es el render en sí (un markdown largo, un componente que hace varias queries)."

Siempre son funciones **async**. Un Server Component async es una función async, por eso funciona en los dos.

### 2.2 — Qué pasa en cada momento

| Momento | Qué hace Next con una función `'use cache'` |
|---|---|
| **Build** | La ejecuta y mete el resultado en el static shell (si su vida no es demasiado corta) |
| **Request (hit)** | Devuelve el resultado guardado sin ejecutar |
| **Request (stale)** | Devuelve el viejo y regenera en segundo plano |
| **Request (expirado)** | Ejecuta, espera y guarda |
| **Navegación del cliente** | El resultado viaja en el RSC payload y el router lo reusa mientras esté "fresco" para el cliente (`stale`) |

### 2.3 — La prueba de que cachea

```tsx
export async function FeaturedProducts() {
  'use cache'
  const products = await listFeatured()
  const at = new Date().toLocaleTimeString('es-AR') // ← adentro de 'use cache' está permitido: se cachea el valor
  return (
    <section>
      <ProductRow products={products} />
      <p className="text-xs text-muted-foreground">Generado a las {at}</p>
    </section>
  )
}
```

```bash
npm run build && npm run start
```

Recargá: la hora **no cambia** y el log de Prisma **no muestra** la query. Es la hora del build.

> "¿Se acuerdan de la clase 1? `Date` en la página tiraba error. Adentro de `'use cache'` no, porque le estamos diciendo a Next exactamente qué queremos: el valor del momento en que se generó esta entrada de cache."

---

## Bloque 3 — `cacheLife` (25 min)

**Siempre** acompañen un `'use cache'` con un `cacheLife`. Sin él, se usa el perfil `default`.

### 3.1 — Las tres duraciones

| Propiedad | Qué controla | Dónde |
|---|---|---|
| `stale` | Cuánto tiempo el **navegador** usa su copia sin preguntar al servidor | Cliente (router) |
| `revalidate` | Después de cuánto el servidor sirve la copia vieja **y regenera en segundo plano** | Servidor |
| `expire` | Después de cuánto la copia ya no se puede servir: el próximo request **espera** la nueva | Servidor |

Dibujalo como una línea de tiempo:

```text
0 ──── revalidate ──────────── expire ────▶
│ fresco │ viejo pero se sirve │ ya no: bloquea │
│        │ (y se regenera)     │                │
```

### 3.2 — Perfiles incluidos

| Perfil | `stale` | `revalidate` | `expire` |
|---|---|---|---|
| `default` | 5m | 15m | nunca |
| `seconds` | 30s | 1s | 60s |
| `minutes` | 5m | 1m | 1h |
| `hours` | 5m | 1h | 1d |
| `days` | 5m | 1d | 1w |
| `weeks` | 5m | 1w | 30d |
| `max` | 5m | 30d | 1y |

### 3.3 — "Vida corta" = hueco dinámico

Un cache con perfil `seconds`, `revalidate: 0` o `expire` menor a 5 minutos se considera **de vida corta**: Next **no** lo mete en el shell (quedaría viejo enseguida) y lo trata como hueco dinámico. Por eso tiene que estar dentro de un `<Suspense>`.

> "Esto es lo que más confunde. `cacheLife('seconds')` no hace que el dato sea estático con refresco cada segundo; hace que sea **dinámico con una cache de un segundo** para no reventar la base si llegan 500 requests juntos."

### 3.4 — Perfiles propios de Comarca

```ts
// next.config.ts
const nextConfig: NextConfig = {
  cacheComponents: true,
  partialPrefetching: true,
  cacheLife: {
    // Catálogo: cambia cuando un productor edita (y ahí invalidamos por tag)
    catalog: { stale: 300, revalidate: 3600, expire: 86400 },
    // Precio: si falla la invalidación, que no quede viejo más de 10 min
    price: { stale: 60, revalidate: 600, expire: 3600 },
    // Contenido del CMS: solo cambia por webhook
    cms: { stale: 300, revalidate: 60 * 60 * 24 * 30, expire: 60 * 60 * 24 * 365 },
  },
}
```

```ts
import { cacheLife } from 'next/cache'

async function getX() {
  'use cache'
  cacheLife('catalog') // tipado: TS conoce los perfiles definidos
}
```

> "Nombrar perfiles por **dominio** (catalog, price, cms) y no por tiempo (hours) hace que el día que el negocio pida 'que los precios se actualicen más rápido', se cambia un número en un lugar."

---

## Bloque 4 — `cacheTag` (15 min)

Un tag es una etiqueta para poder **invalidar** después (clase 11): "todo lo que tenga `product:queso-azul`, tiralo".

```ts
// src/server/cache-tags.ts
// Un solo lugar para los nombres de tags. Nunca strings sueltos.
export const tags = {
  catalog: 'catalog',                                     // todo el catálogo
  category: (slug: string) => `category:${slug}`,
  product: (slug: string) => `product:${slug}`,
  producer: (id: string) => `producer:${id}`,
  producerProducts: (id: string) => `producer-products:${id}`,
  cms: (collection: string) => `cms:${collection}`,       // clase 19
} as const
```

```ts
import { cacheLife, cacheTag } from 'next/cache'
import { tags } from '@/server/cache-tags'

async function getProductBySlug(slug: string) {
  'use cache'
  cacheLife('catalog')
  cacheTag(tags.catalog, tags.product(slug))
  // ...
}
```

Reglas:

- Un mismo resultado puede tener **varios** tags (general + específico).
- Los tags se pueden calcular **después** de obtener los datos (por ejemplo, `cacheTag(tags.producer(row.producerId))` cuando ya leíste la fila).
- **Los tags y las claves se guardan en texto plano.** Nada de emails, tokens ni datos personales en un tag. IDs sí.

---

## Bloque 5 — Claves, serialización y prohibiciones (25 min)

### 5.1 — Qué forma la clave de cache

```text
build id  +  id de la función  +  argumentos serializados  +  variables capturadas del closure
```

Consecuencias:

1. **Cada deploy arranca con cache vacía** (cambia el build id). Incluso la `remote`.
2. `getProductBySlug('queso-azul')` y `getProductBySlug('miel')` son entradas distintas.
3. Si la función **captura** una variable de afuera, esa variable también es parte de la clave:

```tsx
async function ProductPage({ slug }: { slug: string }) {
  async function getData() {
    'use cache'
    return db.product.findUnique({ where: { slug } }) // slug capturado → parte de la clave
  }
  // ...
}
```

### 5.2 — Argumentos y retorno serializables

Tienen que poder serializarse como props de un Client Component (clase 3): primitivos, objetos planos, arrays, `Date`, `Map`, `Set`. **No**: instancias de clases, funciones, el cliente de Prisma, `Decimal` de Prisma (convertilo a número/string en la DAL).

Excepción útil: **pass-through**. Un componente cacheado puede recibir `children` o JSX y renderizarlo **sin mirarlo**:

```tsx
async function ProductShell({ slug, children }: { slug: string; children: React.ReactNode }) {
  'use cache'
  cacheLife('catalog')
  const product = await getProductBySlug(slug)
  return (
    <article>
      <h1>{product?.name}</h1>
      {children /* no entra en la clave; se "enchufa" después */}
    </article>
  )
}

// uso: el hueco dinámico va adentro del componente cacheado
<ProductShell slug={slug}>
  <Suspense fallback={<StockSkeleton />}>
    <StockLive slug={slug} />
  </Suspense>
</ProductShell>
```

> "Esto es potentísimo: la cáscara cacheada envuelve un pedazo dinámico. Si `ProductShell` intentara leer o transformar `children`, se rompe; si solo lo pone en el JSX, funciona."

### 5.3 — Lo que NO se puede hacer adentro de `'use cache'`

```ts
async function bad() {
  'use cache'
  const c = await cookies()   // ❌ error: runtime API dentro de cache compartida
  const h = await headers()   // ❌
}
```

> "Piensen por qué: si la cache es **compartida entre usuarios** y adentro leés la cookie de Juan, el resultado se le sirve a María. Next lo prohíbe directamente."

Patrón correcto (de la doc): **leer afuera, pasar adentro**.

```tsx
async function Prices() {
  const currency = (await cookies()).get('currency')?.value ?? 'ARS' // afuera: dinámico
  return <CachedPrices currency={currency} />                         // adentro: cacheado por moneda
}

async function CachedPrices({ currency }: { currency: string }) {
  'use cache'
  cacheLife('price')
  // ...
}
```

Excepción: **root params** (`locale()` de `next/root-params`) **sí** se pueden leer dentro de `'use cache'`: Next los suma a la clave automáticamente.

### 5.4 — `React.cache` y `'use cache'`

Las funciones que ahora tienen `'use cache'` ya no necesitan el `cache()` de React de la clase 6: dentro del mismo request, la segunda llamada es un hit. Mantengan `React.cache` para lo que **no** se puede cachear entre requests (por ejemplo, la sesión del usuario, clase 17).

---

## Bloque 6 — En vivo: la DAL y la página de producto (30 min)

### 6.1 — La DAL

```ts
// src/server/dal/catalog.ts
import 'server-only'
import { cacheLife, cacheTag } from 'next/cache'
import { locale as rootLocale } from 'next/root-params'
import { db } from '@/server/db'
import { tags } from '@/server/cache-tags'

export async function listCategories() {
  'use cache'
  cacheLife('catalog')
  cacheTag(tags.catalog)
  return db.category.findMany({ select: { slug: true, name: true }, orderBy: { name: 'asc' } })
}

export async function listProducts(categorySlug?: string): Promise<ProductCardDTO[]> {
  'use cache'
  cacheLife('catalog')
  cacheTag(tags.catalog, ...(categorySlug ? [tags.category(categorySlug)] : []))

  const lang = await rootLocale()
  const rows = await db.product.findMany({
    where: { isActive: true, ...(categorySlug && { category: { slug: categorySlug } }) },
    select: { ...cardSelect, translations: { where: { locale: lang }, select: { name: true } } },
    orderBy: { createdAt: 'desc' },
  })
  return rows.map(toCardDTO)
}

export async function getProductBySlug(slug: string): Promise<ProductDetailDTO | null> {
  'use cache'
  cacheLife('catalog')
  cacheTag(tags.catalog, tags.product(slug))

  const lang = await rootLocale()
  const row = await db.product.findUnique({ where: { slug, isActive: true }, select: detailSelect(lang) })
  if (!row) return null

  cacheTag(tags.producer(row.producerId)) // tag calculado con el dato
  return toDetailDTO(row)
}

// Stock: dinámico a propósito. Sin 'use cache'.
export async function getStock(slug: string) {
  const row = await db.product.findUnique({ where: { slug }, select: { stock: true } })
  return row?.stock ?? 0
}
```

> "Fíjense que `getStock` **no** tiene cache. Es una decisión, no un olvido: si mostramos 'quedan 3' y en realidad quedan 0, vendemos algo que no existe. Lo dejamos como hueco dinámico. Igual, el checkout vuelve a validar el stock en una transacción (clase 13): la UI nunca es la fuente de verdad."

### 6.2 — La página

```tsx
// src/app/(frontend)/[locale]/(shop)/products/[slug]/page.tsx
import { Suspense } from 'react'
import { notFound } from 'next/navigation'
import { setRequestLocale } from 'next-intl/server'
import { getProductBySlug, listProductSlugs } from '@/server/dal/catalog'
import { ProductDetail } from '@/features/catalog/components/product-detail'
import { StockLive, StockSkeleton } from '@/features/catalog/components/stock-live'
import { RelatedProducts } from '@/features/catalog/components/related-products'

export async function generateStaticParams() {
  return listProductSlugs()
}

export async function generateMetadata(props: PageProps<'/[locale]/products/[slug]'>) {
  const { slug } = await props.params
  const product = await getProductBySlug(slug) // cacheado → la metadata entra en el shell
  return { title: product?.name, description: product?.description.slice(0, 155) }
}

export default async function ProductPage(props: PageProps<'/[locale]/products/[slug]'>) {
  const { locale, slug } = await props.params
  setRequestLocale(locale)

  const product = await getProductBySlug(slug)
  if (!product) notFound()

  return (
    <div className="space-y-12">
      <ProductDetail
        product={product}
        stockSlot={
          <Suspense fallback={<StockSkeleton />}>
            <StockLive slug={slug} />
          </Suspense>
        }
      />
      <RelatedProducts categoryId={product.categoryId} excludeId={product.id} />
    </div>
  )
}
```

> "¿Por qué ahora sí podemos hacer `await params` y `await getProductBySlug` arriba de todo, sin Suspense? Porque los dos se resuelven en build: `params` está en `generateStaticParams` y `getProductBySlug` está cacheado. Ya no bloquean nada. Y de yapa: `notFound()` ocurre **antes** de empezar a streamear, así que un slug inexistente devuelve un 404 de verdad (el problema que dejamos anotado en la clase 6)."

`RelatedProducts` también es cacheado (usa `listRelated` con `'use cache'`), así que entra al shell. El único hueco es `StockLive` (y el `CartBadge` del header).

**Validación:**

```bash
npm run build
```

- `/[locale]/products/[slug]` → `◐`.
- `curl localhost:3000/es/products/queso-azul` → el HTML trae nombre, precio, descripción, relacionados y la metadata en `<head>`. Solo el stock está como fallback.
- Log de Prisma en `npm run start`: por request, **una sola query** (el stock).

### 6.3 — Para slugs desconocidos

Creá un producto nuevo desde Prisma Studio (`miel-de-cana`) y visitalo sin rebuildear:

- Primera visita: App Shell + contenido streameado (y se genera el shell concreto en segundo plano).
- Segunda visita: shell completo al instante.

---

## Bloque 7 — Las variantes: `private` y `remote` (15 min)

| | `'use cache'` | `'use cache: remote'` | `'use cache: private'` |
|---|---|---|---|
| Dónde se guarda en el server | Memoria (LRU, por instancia) | Cache handler remoto (Redis, etc.) | **Nunca** en el servidor |
| Compartido entre usuarios | Sí | Sí | No (solo ese navegador) |
| Puede leer cookies/headers | No | No | **Sí** |
| Entra en el static shell | Sí | No (es para lo que corre por request) | No |
| Para qué | Datos públicos | Datos públicos caros en rutas dinámicas, multi-instancia | Datos del usuario que querés precargar en navegaciones |

### 7.1 — `use cache: private` (adelanto de la clase 17)

```ts
export async function getCartSummary() {
  'use cache: private'
  cacheLife({ stale: 60 })
  const raw = (await cookies()).get('cart')?.value
  // ... calcular cantidad y total ...
}
```

El resultado se guarda **solo en el navegador** de ese usuario, como parte del prefetch. Sirve para que una navegación a `/cart` sea instantánea sin cachear nada personal en el servidor. Con `stale` de menos de 30 segundos deja de participar del prefetch.

### 7.2 — `use cache: remote` (lo configuramos en la clase 11)

Con varias instancias del servidor (o serverless), la memoria de cada una es su propia cache: el hit rate baja. `remote` guarda en un store compartido. Tiene costo (latencia de red, infraestructura). La doc recomienda usarlo cuando:

- La API de atrás tiene rate limit o cobra por request.
- La base sufre bajo carga.
- La clave tiene **pocos valores distintos** (por categoría, por idioma; no por usuario ni por filtro de precio).

### 7.3 — Reglas de anidamiento

- `remote` dentro de `remote` o de `use cache`: ✅
- `remote` dentro de `private` o `private` dentro de `remote`: ❌

---

## Seguridad: dos avisos que no son opcionales

1. **Next ≥ 16.4.1.** El 14/10/2026 se publicó un RCE sin autenticación en funciones `'use cache'` que devuelven varios typed arrays (GHSA-673q-mjm6-r5xr). No hay workaround: hay que actualizar. (Clase 18.)
2. **Nunca exporten una función cacheada que reciba un `userId` del llamador.** Si `getOrdersByUser(userId)` está cacheada y exportada, cualquiera que la llame con otro ID obtiene datos ajenos. El patrón correcto (clase 17): la función exportada resuelve el usuario de la sesión y llama a una función cacheada **no exportada** con ese ID.

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| `cookies() was called inside "use cache"` | Runtime API en cache compartida | Leer afuera y pasar el valor, o `use cache: private` |
| `Only plain objects can be passed...` | Retorno con `Decimal` o una clase | Convertir en la DAL |
| El dato cacheado nunca se actualiza | Sin `cacheLife` adecuado ni invalidación | Clase 11: tags + `updateTag` |
| Puse `cacheLife('seconds')` y la página sigue con hueco | Vida corta = dinámico | Es lo esperado; envolver en Suspense |
| En dev parece no cachear | Dev invalida al guardar archivos | Probar con `npm run build && npm run start` |

---

## Tarea

1. Pasá **toda** la DAL de catálogo a `'use cache'` con los perfiles `catalog`/`price` y tags de `cache-tags.ts`. `getStock` queda dinámico.
2. Home: hero, categorías y destacados en el shell; solo el `CartBadge` como hueco. Verificá con `curl` que los destacados están en el HTML.
3. Página de productor con `ProducerShell` cacheado que envuelve (pass-through) una sección dinámica "productos con poco stock".
4. Explicá en el PR, para cada función de la DAL, por qué eligieron ese perfil y esos tags.

**Terminado cuando:** con `npm run start` y el log de Prisma, la home hace **0 queries** por visita y la página de producto **1** (stock).
