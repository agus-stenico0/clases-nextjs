# Clase 5 — Prisma 7 + PostgreSQL dentro de Next

**Duración:** 2:30 hs
**Branch de salida:** `clase-05-prisma`
**Lo que se llevan:** Postgres en Docker con dos bases (app y CMS), el schema de Comarca en Prisma 7, el cliente con driver adapter y singleton para que el hot reload no abra cien conexiones, migraciones, seed, y el catálogo leyendo de la base.

**Requisito:** tag `clase-04`. Docker Desktop con integración WSL andando (`docker ps` sin error).

> En el módulo de NestJS vieron Prisma a fondo. Hoy no repetimos fundamentos: nos concentramos en **lo que cambia cuando Prisma vive adentro de Next** y en lo que cambió de Prisma 6 a 7.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:10 | El dolor: el JSON no escala y no se puede escribir |
| 0:10 – 0:30 | Docker Compose: Postgres con dos bases + Redis |
| 0:30 – 0:50 | Prisma 7: qué cambió (generador, adapter, `prisma.config.ts`) |
| 0:50 – 1:20 | Schema de Comarca completo |
| 1:20 – 1:40 | El cliente en Next: singleton y por qué |
| 1:40 – 2:00 | Migraciones + seed |
| 2:00 – 2:25 | **En vivo:** catálogo desde la base, y el primer choque con Cache Components |
| 2:25 – 2:30 | Tarea |

---

## Bloque 1 — El dolor (10 min)

> "El panel del productor tiene que crear productos. El checkout tiene que descontar stock **sin vender dos veces el último queso**. Un JSON no tiene transacciones, ni índices, ni concurrencia. Pasamos a Postgres."

---

## Bloque 2 — Infraestructura local (20 min)

```yaml
# docker/compose.dev.yml
name: comarca

services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: comarca
      POSTGRES_PASSWORD: comarca
      POSTGRES_DB: comarca
    ports:
      - '5432:5432'
    volumes:
      - pg-data:/var/lib/postgresql/data
      - ./postgres/init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U comarca -d comarca']
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7-alpine
    ports:
      - '6379:6379'
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 5s
      retries: 10

volumes:
  pg-data:
```

```sql
-- docker/postgres/init/01-create-cms-db.sql
-- Payload (clase 19) usa su propia base. Misma instancia, base distinta:
-- cada ORM es dueño de su schema y sus migraciones no se pisan.
CREATE DATABASE comarca_cms OWNER comarca;
```

> "¿Por qué dos bases y no una? Payload y Prisma generan migraciones cada uno por su cuenta. Si comparten base, el día que Prisma ve tablas que no conoce te propone borrarlas. Separadas, ninguno sabe que el otro existe."

Scripts:

```json
{
  "scripts": {
    "infra:up": "docker compose -f docker/compose.dev.yml up -d --wait",
    "infra:down": "docker compose -f docker/compose.dev.yml down",
    "infra:reset": "docker compose -f docker/compose.dev.yml down -v && npm run infra:up"
  }
}
```

```bash
npm run infra:up
docker compose -f docker/compose.dev.yml ps    # postgres y redis: healthy
docker exec -it comarca-postgres-1 psql -U comarca -c '\l'   # aparecen comarca y comarca_cms
```

> El script de `init/` solo corre la **primera vez** que se crea el volumen. Si alguien ya tenía el volumen, `npm run infra:reset`.

---

## Bloque 3 — Prisma 7: qué cambió (20 min)

| Prisma 6 | Prisma 7 |
|---|---|
| `provider = "prisma-client-js"` | `provider = "prisma-client"` |
| El cliente se generaba en `node_modules/.prisma` | **`output` obligatorio**: el cliente es código de tu proyecto |
| Motor de consultas en Rust (binario) | Sin binario Rust: el cliente habla con la base mediante un **driver adapter** de JS |
| `url = env("DATABASE_URL")` en el schema | La conexión se configura en **`prisma.config.ts`** (CLI) y en el adapter (runtime) |
| `import { PrismaClient } from '@prisma/client'` | `import { PrismaClient } from '@/generated/prisma/client'` |

> "La consecuencia práctica más linda para Next: no hay binario nativo que pelee con Turbopack ni con Docker Alpine. Es todo TypeScript."

### 3.1 — Instalación

```bash
npm i -D prisma@7 tsx @types/pg
npm i @prisma/client@7 @prisma/adapter-pg pg dotenv
npx prisma init --datasource-provider postgresql --output ../src/generated/prisma
```

`init` crea `prisma/schema.prisma`, `prisma.config.ts` y un `.env`. **Borrá ese `.env`**: usamos `.env.local` (clase 4).

```ts
// prisma.config.ts
import { config } from 'dotenv'
import { defineConfig, env } from 'prisma/config'

// La CLI de Prisma no lee .env.local por su cuenta.
config({ path: ['.env.local', '.env'] })

export default defineConfig({
  schema: 'prisma/schema.prisma',
  migrations: {
    path: 'prisma/migrations',
    seed: 'tsx prisma/seed.ts',
  },
  datasource: {
    url: env('DATABASE_URL'),
  },
})
```

`.gitignore`:

```text
/src/generated/
```

> "El cliente generado no se versiona: se regenera con `prisma generate`. Lo agregamos a `postinstall` para que nadie se olvide."

```json
{ "scripts": { "postinstall": "prisma generate" } }
```

---

## Bloque 4 — El schema de Comarca (30 min)

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
}

enum Role {
  customer
  producer
  admin
}

enum OrderStatus {
  pending
  paid
  preparing
  shipped
  delivered
  cancelled
}

// ───────────── Usuarios ─────────────
// Forma compatible con Better Auth (clase 17): id String sin default
// (lo genera la lib), name, email, emailVerified, image.
model User {
  id            String   @id
  name          String
  email         String   @unique
  emailVerified Boolean  @default(false)
  image         String?
  role          Role     @default(customer)
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  producer  Producer?
  orders    Order[]
  favorites Favorite[]

  @@map("user")
}

// ───────────── Catálogo ─────────────
model Producer {
  id          String   @id @default(cuid())
  slug        String   @unique
  name        String
  description String?
  province    String
  ownerId     String   @unique
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  owner    User      @relation(fields: [ownerId], references: [id])
  products Product[]

  @@map("producers")
}

model Category {
  id       String    @id @default(cuid())
  slug     String    @unique
  name     String
  products Product[]

  @@map("categories")
}

model Product {
  id          String   @id @default(cuid())
  slug        String   @unique
  name        String
  description String
  priceCents  Int
  stock       Int      @default(0)
  imageUrl    String
  isActive    Boolean  @default(true)
  producerId  String
  categoryId  String
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  producer   Producer    @relation(fields: [producerId], references: [id])
  category   Category    @relation(fields: [categoryId], references: [id])
  orderItems OrderItem[]
  favorites  Favorite[]

  @@index([categoryId, isActive])
  @@index([producerId])
  @@map("products")
}

// ───────────── Pedidos ─────────────
model Order {
  id         String      @id @default(cuid())
  userId     String
  status     OrderStatus @default(pending)
  totalCents Int
  paymentId  String?     @unique
  createdAt  DateTime    @default(now())
  updatedAt  DateTime    @updatedAt

  user  User        @relation(fields: [userId], references: [id])
  items OrderItem[]

  @@index([userId, createdAt])
  @@map("orders")
}

model OrderItem {
  id             String @id @default(cuid())
  orderId        String
  productId      String
  quantity       Int
  unitPriceCents Int

  order   Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  product Product @relation(fields: [productId], references: [id])

  @@map("order_items")
}

model Favorite {
  userId    String
  productId String
  createdAt DateTime @default(now())

  user    User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  product Product @relation(fields: [productId], references: [id], onDelete: Cascade)

  @@id([userId, productId])
  @@map("favorites")
}
```

Decisiones para comentar:

- **Precio en centavos (`Int`)**. Nunca `Float` para plata. Ya lo vieron en Nest, pero repetilo.
- **`unitPriceCents` en `OrderItem`**: el precio del momento de la compra. Si el productor sube el precio mañana, el pedido de ayer no cambia.
- **`@@index([categoryId, isActive])`**: la query más frecuente del sitio es "productos activos de una categoría".
- **`Producer.ownerId @unique`**: un usuario productor tiene un solo emprendimiento (regla de negocio simplificada).
- **Las tablas de sesión de Better Auth** (`session`, `account`, `verification`) las agrega la CLI de Better Auth en la clase 17.

---

## Bloque 5 — El cliente en Next (20 min)

```ts
// src/server/db.ts
import 'server-only'
import { PrismaPg } from '@prisma/adapter-pg'
import { PrismaClient } from '@/generated/prisma/client'
import { env } from '@/lib/env/server'

function createPrismaClient() {
  const adapter = new PrismaPg({ connectionString: env.DATABASE_URL })
  return new PrismaClient({
    adapter,
    log: env.NODE_ENV === 'development' ? ['query', 'warn', 'error'] : ['error'],
  })
}

const globalForPrisma = globalThis as unknown as {
  prisma?: ReturnType<typeof createPrismaClient>
}

export const db = globalForPrisma.prisma ?? createPrismaClient()

if (env.NODE_ENV !== 'production') globalForPrisma.prisma = db
```

### 5.1 — ¿Por qué el `globalThis`?

Demostralo sin el singleton:

```ts
export const db = createPrismaClient()
```

Abrí otra terminal:

```bash
watch -n1 "docker exec comarca-postgres-1 psql -U comarca -tc \"select count(*) from pg_stat_activity where datname='comarca'\""
```

Guardá un archivo 15 veces con `npm run dev` corriendo. Las conexiones suben y suben.

> "Cada hot reload re-evalúa los módulos. Cada evaluación crea un `PrismaClient` nuevo con su pool. El viejo no se cierra. En diez minutos de desarrollo llegan al `max_connections` de Postgres y todo se cae con `too many clients`. `globalThis` sobrevive al hot reload: guardamos el cliente ahí y lo reusamos."

En producción no hace falta (el módulo se evalúa una vez por proceso), por eso el `if`.

### 5.2 — `server-only` acá no es opcional

> "Este archivo tiene la cadena de conexión. Si un Client Component lo importa por error, sin `server-only` el build podría intentar empaquetar `pg` para el navegador y fallar de forma confusa, o peor, filtrar lógica. Con `server-only`, el error dice exactamente qué pasó."

---

## Bloque 6 — Migraciones y seed (20 min)

```bash
npx prisma migrate dev --name init
npx prisma generate
```

> "En Prisma 7 no asuman que `migrate dev` regenera el cliente ni corre el seed solo: córranlos explícitos. En CI y en Docker usamos `prisma migrate deploy`, que **solo aplica** migraciones existentes y nunca resetea. `migrate dev` en producción es como `rm -rf` con otro nombre: si detecta drift, te ofrece borrar la base."

Mostrá el SQL generado en `prisma/migrations/xxxx_init/migration.sql` y que los índices están.

### 6.1 — Seed

```ts
// prisma/seed.ts
import { config } from 'dotenv'
import { PrismaPg } from '@prisma/adapter-pg'
import { PrismaClient } from '../src/generated/prisma/client'

config({ path: ['.env.local', '.env'] })

const db = new PrismaClient({
  adapter: new PrismaPg({ connectionString: process.env.DATABASE_URL! }),
})

const categories = [
  { slug: 'quesos', name: 'Quesos' },
  { slug: 'miel', name: 'Miel' },
  { slug: 'vinos', name: 'Vinos' },
  { slug: 'conservas', name: 'Conservas' },
]

const producers = [
  { slug: 'la-quesera', name: 'La Quesera', province: 'Buenos Aires', owner: 'quesera@comarca.dev' },
  { slug: 'apicola-sur', name: 'Apícola Sur', province: 'Río Negro', owner: 'apicola@comarca.dev' },
  { slug: 'finca-los-alamos', name: 'Finca Los Álamos', province: 'Mendoza', owner: 'finca@comarca.dev' },
]

async function main() {
  // Orden inverso a las FKs
  await db.favorite.deleteMany()
  await db.orderItem.deleteMany()
  await db.order.deleteMany()
  await db.product.deleteMany()
  await db.producer.deleteMany()
  await db.category.deleteMany()
  await db.user.deleteMany()

  const cats = await Promise.all(categories.map((c) => db.category.create({ data: c })))
  const catBySlug = Object.fromEntries(cats.map((c) => [c.slug, c.id]))

  for (const p of producers) {
    const owner = await db.user.create({
      data: { id: crypto.randomUUID(), name: p.name, email: p.owner, role: 'producer' },
    })
    await db.producer.create({
      data: { slug: p.slug, name: p.name, province: p.province, ownerId: owner.id },
    })
  }

  const prodBySlug = Object.fromEntries(
    (await db.producer.findMany()).map((p) => [p.slug, p.id]),
  )

  const products = [
    ['queso-azul', 'Queso azul de Tandil', 'quesos', 'la-quesera', 1250000, 12],
    ['reggianito', 'Reggianito estacionado', 'quesos', 'la-quesera', 980000, 30],
    ['sardo', 'Sardo artesanal', 'quesos', 'la-quesera', 870000, 0],
    ['miel-monte', 'Miel de monte', 'miel', 'apicola-sur', 680000, 40],
    ['miel-cremosa', 'Miel cremosa', 'miel', 'apicola-sur', 720000, 25],
    ['malbec-joven', 'Malbec joven', 'vinos', 'finca-los-alamos', 940000, 60],
    ['cabernet-reserva', 'Cabernet reserva', 'vinos', 'finca-los-alamos', 1890000, 8],
    ['dulce-de-leche', 'Dulce de leche colonial', 'conservas', 'la-quesera', 540000, 50],
  ] as const

  for (const [slug, name, cat, producer, priceCents, stock] of products) {
    await db.product.create({
      data: {
        slug,
        name,
        description: `**${name}** elaborado por productores de la comarca.\n\nIdeal para compartir.`,
        priceCents,
        stock,
        imageUrl: `/products/${slug}.jpg`,
        categoryId: catBySlug[cat],
        producerId: prodBySlug[producer],
      },
    })
  }

  console.log(`✔ Seed: ${cats.length} categorías, ${producers.length} productores, ${products.length} productos`)
}

main()
  .catch((e) => {
    console.error(e)
    process.exit(1)
  })
  .finally(() => db.$disconnect())
```

```bash
npx prisma db seed
npx prisma studio      # http://localhost:5555 → mirá las tablas
```

Agregá `"db:reset": "prisma migrate reset --force && prisma generate && prisma db seed"` para empezar de cero en clase.

---

## Bloque 7 — En vivo: el catálogo desde la base (25 min)

### 7.1 — Primer módulo de la Data Access Layer

```ts
// src/server/dal/catalog.ts
import 'server-only'
import { db } from '@/server/db'

export async function listProducts(params: { categorySlug?: string } = {}) {
  return db.product.findMany({
    where: {
      isActive: true,
      ...(params.categorySlug && { category: { slug: params.categorySlug } }),
    },
    select: {
      slug: true,
      name: true,
      priceCents: true,
      imageUrl: true,
      stock: true,
      producer: { select: { name: true } },
    },
    orderBy: { createdAt: 'desc' },
  })
}

export async function getProductBySlug(slug: string) {
  return db.product.findUnique({
    where: { slug, isActive: true },
    include: { producer: true, category: true },
  })
}

export async function listProductSlugs() {
  return db.product.findMany({ where: { isActive: true }, select: { slug: true } })
}
```

> "`select` explícito en el listado: traemos exactamente los campos de la card. Es performance (menos datos) y es seguridad (lo que no se trae no se puede filtrar al cliente)."

### 7.2 — La grilla

```tsx
// src/features/catalog/components/product-grid.tsx
import { listProducts } from '@/server/dal/catalog'
import { ProductCard } from './product-card'

export async function ProductGrid({ searchParams }: { searchParams: Promise<{ category?: string | string[] }> }) {
  const { category } = await searchParams
  const products = await listProducts({
    categorySlug: typeof category === 'string' ? category : undefined,
  })

  if (products.length === 0) return <p className="text-muted-foreground">No hay productos.</p>

  return (
    <div className="grid grid-cols-2 gap-4 md:grid-cols-3 lg:grid-cols-4">
      {products.map((p) => (
        <ProductCard
          key={p.slug}
          product={{ ...p, producerName: p.producer.name }}
        />
      ))}
    </div>
  )
}
```

### 7.3 — El choque con Cache Components

Ahora la home. Teníamos:

```tsx
export default async function HomePage() {
  const products = await listProducts()   // antes era el JSON síncrono
  return <ProductList products={products} />
}
```

`npm run dev` → overlay con insight **blocking-route** apuntando a `listProducts`. Preguntá: "con el JSON andaba, ¿qué cambió?".

> "El JSON era un archivo de nuestro repo: Next sabía que en build y en runtime vale lo mismo. Una base de datos **cambia**: alguien carga un producto y la home de ayer quedó vieja. Next no sabe si a ustedes les importa o no, entonces les pregunta lo mismo que con la hora: ¿lo querés **fresco en cada request** (stream con Suspense) o te alcanza con una **copia cacheada** (`'use cache'`)? Hoy elegimos fresco. En la clase 10 vamos a cachear la mitad de esto."

```tsx
// src/app/(shop)/page.tsx
import { Suspense } from 'react'
import { ProductGridSkeleton } from '@/features/catalog/components/product-grid-skeleton'
import { LatestProducts } from '@/features/catalog/components/latest-products'

export default function HomePage() {
  return (
    <>
      <section className="py-12">
        <h1 className="font-serif text-4xl">Lo mejor de cada comarca</h1>
        <p className="text-muted-foreground">Directo del productor a tu mesa.</p>
      </section>
      <Suspense fallback={<ProductGridSkeleton count={8} />}>
        <LatestProducts />
      </Suspense>
    </>
  )
}
```

**Validación:**

```bash
npm run build
```

- `/` aparece con `◐`: el hero está en el shell; los productos se streamean.
- En la terminal de `npm run dev`, con `log: ['query']`, se ve el SQL de cada request.

### 7.4 — `generateStaticParams` contra la base

```tsx
// src/app/(shop)/products/[slug]/page.tsx
import { listProductSlugs } from '@/server/dal/catalog'

export async function generateStaticParams() {
  return listProductSlugs()
}
```

> "Esto corre **en build**. Consecuencia que les va a doler en la clase 22: para buildear, la base tiene que estar accesible desde donde se buildea. En Docker eso cambia cómo armamos el Dockerfile. Anótenlo."

La página de producto también lee la base: misma regla, el contenido que depende de la query va dentro de un `<Suspense>` (o se cachea en la clase 10).

---

## Errores comunes

| Error | Causa | Solución |
|---|---|---|
| `Cannot find module '@/generated/prisma/client'` | No se corrió `prisma generate` | `npx prisma generate` (y tenerlo en `postinstall`) |
| `too many clients already` | Sin singleton en dev | `globalThis` |
| `Environment variable not found: DATABASE_URL` al migrar | La CLI no ve `.env.local` | `config({ path: ['.env.local', '.env'] })` en `prisma.config.ts` |
| `Can't reach database server` | Docker no levantó / puerto ocupado por un Postgres local | `npm run infra:up`, revisar puerto 5432 |
| Blocking-route en cualquier página con query | Dato no cacheado fuera de Suspense | `<Suspense>` hoy, `'use cache'` en clase 10 |

---

## Tarea

1. `/categories/[slug]` leyendo de la base con `generateStaticParams` desde `Category`, y 404 si la categoría no existe.
2. Página de producto con nombre, productor, precio, stock y descripción en markdown (`marked`, del lado servidor).
3. Agregá al seed 12 productos más (al menos 3 sin stock) y 2 categorías.
4. Medí y anotá: ¿cuántas queries SQL salen al cargar `/products`? ¿Y `/products/queso-azul`? (Las vamos a bajar en la clase 6.)

**Terminado cuando:** `npm run db:reset && npm run build` pasa desde cero y todas las páginas del catálogo leen de Postgres.
