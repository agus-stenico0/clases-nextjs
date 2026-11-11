# Clase 6 — Data fetching, streaming y Data Access Layer

**Duración:** 2:30 hs
**Formato:** clase práctica sobre el código terminado de `comarca-web` (branch `feat/clases-06-08`). No se escribe la DAL desde cero: se abre, se corre, se mide y **se rompe a propósito**.
**Lo que se llevan:** ver un waterfall con números, entender por qué el streaming gana, deduplicar con `React.cache`, y saber leer la DAL de Comarca (DTOs, `server-only`, regla de ESLint).

> Las rutas viven en `src/app/(frontend)/[locale]/...`. La carpeta `[locale]` es de la clase 8: hoy pensala como "la carpeta donde están las páginas". Todas las URLs de hoy empiezan con `/es`.

---

## Antes de entrar al aula (10 min, una sola vez)

```powershell
cd proyectos/comarca-web
npm install
Copy-Item .env.example .env   # solo la primera vez
npm run infra:up          # Postgres en :5440 y Redis en :6389 (Docker)
npm run db:reset          # migraciones + seed: 9 productos, 4 productores, favoritos
npm run dev:slow          # = next dev con FAKE_LATENCY=1
```

Abrí `http://localhost:3000/es/lab`: es el índice de demos. Dejá **la terminal de `npm run dev` visible al lado del navegador** toda la clase: Prisma loguea cada query (`prisma:query SELECT ...`).

| Script | Qué hace |
|---|---|
| `npm run dev` | dev normal |
| `npm run dev:slow` | cada query de la DAL tarda 150–400 ms (`src/server/dal/_latency.ts`) |

> `FAKE_LATENCY=1 npm run dev` no anda en PowerShell. Por eso existe `dev:slow` (usa `cross-env`).

---

## Guion

| Tiempo | Demo |
|---|---|
| 0:00 – 0:20 | Demo 1 — El waterfall, medido |
| 0:20 – 0:45 | Demo 2 — `Promise.all` y por qué no alcanza |
| 0:45 – 1:15 | Demo 3 — Streaming por componentes (la página real) |
| 1:15 – 1:35 | Demo 4 — `React.cache`: sacarlo y contar queries |
| 1:35 – 2:05 | Demo 5 — La DAL: DTOs, `server-only`, ESLint |
| 2:05 – 2:20 | Demo 6 — N+1 en la terminal |
| 2:20 – 2:30 | Checklist + tarea |

---

## Demo 1 — El waterfall, medido (20 min)

**Abrí** `src/app/(frontend)/[locale]/(shop)/lab/waterfall/[slug]/page.tsx`, función `loadNaive`:

```ts
const product = await getProductBySlug(slug)                      // 300 ms
const related = await listRelated(product.categoryId, product.id)    // 400 ms
const fromProducer = await listByProducer(product.producerId, product.id) // 400 ms
const favCount = await countFavorites(product.id)                  // 200 ms
```

**Corré** (con `npm run dev:slow`): `http://localhost:3000/es/lab/waterfall/queso-azul?modo=secuencial`

**Mirá:**
- En pantalla: "Cargando TODO…" y después aparece todo junto.
- La caja amarilla y la terminal: `[lab/waterfall] modo=secuencial → ~1370 ms`.

**Pregunta al aula:** "¿Por qué no lo ven en la pestaña Network como una escalera?"

> "Cuatro consultas, una atrás de la otra, y el contenido en blanco hasta la última. En una SPA el waterfall se ve en Network como escalera de requests; acá pasa **adentro del servidor** y lo único que lo delata es el tiempo total. Por eso es más traicionero."

Dibujá el grafo en el pizarrón:

```text
getProductBySlug ──┬──▶ listRelated(categoryId)
                   ├──▶ listByProducer(producerId)
                   └──▶ countFavorites(productId)
```

Las tres de abajo dependen del producto, pero **no entre sí**.

---

## Demo 2 — `Promise.all` (25 min)

**Corré:** `.../lab/waterfall/queso-azul?modo=paralelo` (mismo archivo, rama `if (modo === 'paralelo')`).

**Mirá:** terminal → `modo=paralelo → ~750 ms`. Es 300 + max(400, 400, 200).

**Medido en la PC del curso** (script que anota cuándo llega cada pedazo del HTML):

| Versión | Llega el header del sitio | Llega el producto | Fin del documento |
|---|---|---|---|
| secuencial | 277 ms | **1741 ms** | 1744 ms |
| `Promise.all` | 163 ms | **984 ms** | 988 ms |

> "Mejor: la página ya no tarda la suma. Pero sigue tardando **lo más lento**, y el nombre del producto (que estaba listo a los 300 ms) espera a los favoritos."

**Rompelo:** en `Promise.all`, metele a `countFavorites` un `fakeLatency(3000)` en `src/server/dal/catalog.ts`. Recargá: **todo** espera 3 s por un contador de corazones. Volvelo a 200.

---

## Demo 3 — Streaming por componentes (30 min)

**Abrí** la página real: `src/app/(frontend)/[locale]/(shop)/products/[slug]/page.tsx`.

```tsx
<Suspense fallback={<ProductDetail.Skeleton />}>
  <ProductSection params={params} />          {/* 300 ms: el producto */}
</Suspense>

// adentro de ProductSection, tres hermanos con su propio boundary:
<Suspense fallback={<FavoritesCount.Skeleton />}><FavoritesCount productId={product.id} /></Suspense>
<Suspense fallback={<ProductGridSkeleton count={4} />}><RelatedProducts ... /></Suspense>
<Suspense fallback={<ProductGridSkeleton count={4} />}><MoreFromProducer ... /></Suspense>
```

Cada componente pide **sus** datos: abrí `src/features/catalog/components/related-products.tsx` (5 líneas de lógica).

**Corré:** `http://localhost:3000/es/products/queso-azul` y recargá varias veces. DevTools → Network → el documento queda "pending" mientras llegan los pedazos.

**Medido:**

```text
status 200 headers en 157 ms
  159 ms  header del sitio + skeletons      ← el static shell
  560 ms  "Queso azul de Tandil" (detalle)
  764 ms  "A 3 personas les encanta"
  998 ms  "También te puede gustar"
 1038 ms  "Más de La Quesera"
 1041 ms  fin del documento   ← UN solo response HTTP
```

> "El documento termina más o menos cuando el `Promise.all`, pero el usuario ve el producto a los 560 ms en vez de a los 984. React renderiza los hermanos en paralelo y cada `<Suspense>` es una ventana independiente: **lo importante aparece primero**."

**Rompelo (waterfall escondido):** en `ProductSection`, antes del `return`, agregá:

```ts
await countFavorites(product.id)   // "lo necesito para algo"
```

Recargá: el detalle y las grillas ahora esperan también a los favoritos. **Un `await` en un padre bloquea a todos sus hijos.** Sacalo.

**Las tres reglas para ubicar boundaries** (pizarrón):

1. **Envolvé la unidad que tiene sentido mostrar junta.** Precio sin nombre no sirve: van en el mismo boundary (`ProductDetail`).
2. **El fallback ocupa el mismo lugar que el contenido.** Abrí `product-grid.tsx`: `ProductGrid` y `ProductGridSkeleton` usan la misma `gridClass`. Si no, la página salta (CLS).
3. **No pongas un boundary por componente "por las dudas".** Diez spinners en cascada es peor que uno.

**`loading.tsx` vs `<Suspense>`:** `loading.tsx` envuelve **toda** la page en un boundary. Es lo que hace el lab (un solo "Cargando TODO…"). Sirve como red de seguridad; lo granular es lo que se usa en serio. Comarca no tiene `loading.tsx` en `products/[slug]`.

**Status code y streaming — demo:**

```powershell
curl.exe -s -o NUL -w "%{http_code}" http://localhost:3000/es/products/no-existe
# 200   ← y en pantalla dice "No encontramos ese producto"
```

> "Cuando el HTML empieza a streamear, el status **ya salió**. `notFound()` dentro de un boundary muestra el not-found y agrega `noindex`, pero el status fue 200. Para un producto inexistente eso importa en SEO: lo resolvemos en la clase 10, cuando el producto pase al shell cacheado."

**Patrón preload:** está en `catalog.ts` (`preloadProduct`). Dispara la query sin esperarla; como `getProductBySlug` usa `React.cache`, el que la pida después recibe **la misma promesa**.

---

## Demo 4 — `React.cache`: sacarlo y contar (20 min)

`generateMetadata` y la página piden **el mismo producto** (`products/[slug]/page.tsx`).

**Corré:** `http://localhost:3000/es/products/malbec-joven` y en la terminal buscá `"slug" = $1`.

**Rompelo:** en `src/server/dal/catalog.ts` cambiá

```ts
export const getProductBySlug = cache(async (slug: string) => { ... })
// por
export const getProductBySlug = (async (slug: string) => { ... })
```

Recargá.

| | Queries `WHERE slug = $1` por render |
|---|---|
| con `cache()` | **1** |
| sin `cache()` | **2** (metadata + página) |

> En `next dev` vas a ver un render extra **después** del `GET ... 200` (Next valida el static shell en desarrollo). Contá las queries de **antes** de la línea `GET`.

Volvé a poner `cache(`.

| | `React.cache` | `'use cache'` (clase 10) |
|---|---|---|
| Alcance | **Un request** | Entre requests y usuarios |
| Dónde vive | Memoria del render actual | Cache de Next |
| Clave | Argumentos por identidad (primitivos ✅, objetos nuevos ❌) | Argumentos serializados |

> "`React.cache` es memoización de un request. Termina el request, se tira. Pasen **primitivos**: `getProducts({ category: 'quesos' })` dos veces son dos objetos distintos y NO deduplica."

---

## Demo 5 — La Data Access Layer (30 min)

**Abrí** `src/server/dal/catalog.ts` de arriba a abajo. Señalá:

1. `import 'server-only'` → si un Client Component la importa, **falla el build**.
2. Los **DTOs** (`ProductCardDTO`, `ProductDetailDTO`): lo único que sale de la capa.
3. `cardSelect`: `select` explícito, nunca `include: { producer: true }`.
4. `src/server/dal/producers.ts`: el `ProducerDTO` **no tiene `ownerId`**.

> "Si devolviéramos la fila de Prisma con `include: { producer: true }`, viaja `producer.ownerId`. Alguien hace `<AddToCartButton product={product} />` y el ID del dueño llega al navegador. Hoy es un ID; mañana un email o un costo interno."

**Mirá cómo cruza al cliente:** `features/catalog/components/product-detail.tsx` le pasa a `AddToCartButton` (`'use client'`) **dos primitivos**: `productId` y `stock`.

**`await connection()` en cada lectura** — está arriba de cada función de la DAL:

> "Con Cache Components todo es dinámico por defecto. Una query sin cachear es dato de request, y `connection()` se lo dice a Next. Sin esto, el build falla: el driver de Prisma usa `new Date()` y Next no lo deja pasar en el prerender (`Next.js encountered the unstable value new Date()`). En la clase 10 cambiamos `connection()` por `'use cache'`."

**Las tres reglas de la DAL** (pizarrón):

1. **Nadie fuera de `src/server/` importa `db`.**
2. **Ninguna función devuelve un tipo de Prisma.** Devuelve un DTO declarado ahí.
3. **Lo que dependa de quién es el usuario se resuelve adentro** (clase 17). Nunca recibe el `userId` del componente.

**Regla 1 forzada con ESLint — rompelo:** creá `src/features/catalog/components/prueba.tsx`:

```ts
import { db } from '@/server/db'
export const x = db
```

```powershell
npx eslint src/features/catalog/components/prueba.tsx
#  1:1  error  '@/server/db' import is restricted from being used. Usá la DAL (src/server/dal), no la base directa
```

Borralo. La regla vive en `eslint.config.mjs` (fijate que está **en el mismo bloque** que la regla de `next/link` de la clase 8: en flat config, dos bloques con la misma regla para el mismo archivo **se pisan**, no se suman).

> "Si un componente necesita un tipo, importa el DTO (`ProductCardDTO`), no `Product` de Prisma. El día que renombren `priceCents`, el error aparece en la DAL, que es el único lugar a tocar."

---

## Demo 6 — N+1 en la terminal (15 min)

**Abrí** `src/server/dal/producers.ts`, las dos funciones de abajo (`listProducersWithProductsNPlusOne` y `listProducersWithProducts`).

**Corré** con la terminal a la vista:

- `http://localhost:3000/es/lab/n-mas-1?modo=n1`
- `http://localhost:3000/es/lab/n-mas-1?modo=include`

La terminal marca cada demo entre `──── [lab/n-mas-1] modo=... ────` y `──── fin ────`. Contá los `prisma:query`:

| Modo | Queries con 4 productores |
|---|---|
| `n1` (un `findMany` por productor en un `for`) | **5** (1 + N) |
| `include` (`select: { products: ... }` anidado) | **2** |

> "Con 4 productores no duele. Con 400, son 401 viajes a la base por visita. El select anidado sigue siendo 2."

---

## Checklist de data fetching (pegala en el aula)

```text
□ ¿Los await independientes están en paralelo (Promise.all o componentes hermanos)?
□ ¿Hay algún await en un padre que bloquea hijos que no lo necesitan?
□ ¿Las funciones que se llaman varias veces por request usan React.cache con argumentos primitivos?
□ ¿Cada <Suspense> envuelve una unidad con sentido y su skeleton mide lo mismo?
□ ¿Ningún componente importa db ni tipos de Prisma? (npm run lint)
□ ¿Lo que cruza a un Client Component son solo los campos que usa?
```

---

## Errores comunes (todos los vimos hoy)

| Síntoma | Causa | Solución |
|---|---|---|
| La query se ejecuta 2 veces por request | Sin `React.cache` (metadata + page) | Envolver en `cache()` |
| `React.cache` no deduplica | Argumento objeto | Pasar primitivos |
| Todo aparece de golpe aunque haya Suspense | Un `await` en el padre de los boundaries | Empujar el `await` hacia abajo |
| Skeleton y contenido de distinto alto | Fallback improvisado | Misma grilla (`gridClass`) |
| Build: `encountered the unstable value new Date()` | Query de Prisma sin `connection()` ni `'use cache'` | `await connection()` en la DAL (clase 10: `'use cache'`) |
| Producto inexistente devuelve 200 | `notFound()` dentro de un boundary que ya streameó | Clase 10 |

---

## Tarea

1. Agregá a la DAL `listTopFavorites(limit: number)` (productos más favoriteados, con DTO) y mostrala en la home con su propio `<Suspense>` y skeleton exacto.
2. En `/es/producers/la-quesera` (ya resuelta en el repo, `producers/[slug]/page.tsx`), grabá un GIF con `npm run dev:slow` mostrando los tres boundaries llegando en orden.
3. Respondé en el PR: ¿por qué `listRelated` no necesita `React.cache` y `getProductBySlug` sí?
4. Rompé a propósito un DTO: agregá `ownerId` a `ProducerDTO`, pasalo a un Client Component y mostrá en el PR dónde aparece en el HTML (buscalo en "Ver código fuente"). Después sacalo.

**Terminado cuando:** `npm run lint` pasa y el log muestra **una** query de producto por render en la página de detalle.
