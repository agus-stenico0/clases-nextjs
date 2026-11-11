# Clase 23 — Integradora y proyecto final

**Duración:** 2:30 hs
**Branch de salida:** `clase-23-resenas`
**Lo que se llevan:** una feature completa construida de punta a punta tocando **todas** las capas del curso, en el orden en que se piensa en un proyecto real; la consigna del proyecto final y la rúbrica con la que se evalúa.

**Requisito:** tag `clase-22` desplegado.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:15 | La feature: reseñas de productos. Requisitos y decisiones |
| 0:15 – 0:35 | Datos y DAL (con autorización de negocio) |
| 0:35 – 1:00 | Lectura: cache, tags, shell, JSON-LD |
| 1:00 – 1:30 | Escritura: action, validación, rate limit, optimistic, invalidación |
| 1:30 – 1:45 | Moderación desde el panel y desde el CMS (decisión) |
| 1:45 – 2:05 | Tests y deploy |
| 2:05 – 2:30 | Proyecto final: consigna, rúbrica, preguntas de defensa |

> **Modalidad:** los chicos manejan el teclado por turnos (15 min cada uno). Vos hacés las preguntas de cada bloque **antes** de que escriban código. La clase evalúa si pueden tomar las decisiones solos.

---

## Bloque 1 — Requisitos y decisiones (15 min)

**Feature:** reseñas de productos.

- Solo puede reseñar quien **compró y recibió** el producto (pedido `delivered`).
- Una reseña por usuario por producto (se puede editar).
- Puntaje 1–5 + texto opcional (máx. 1.000 caracteres).
- En la página de producto: promedio, cantidad y las 10 más recientes.
- El productor puede **ocultar** reseñas ofensivas (no borrarlas).
- Estrellas en Google (JSON-LD `aggregateRating`).
- En los tres idiomas (la interfaz; el texto de la reseña queda en el idioma en que se escribió).

Preguntas para el grupo (respuestas esperadas entre paréntesis):

1. ¿Las reseñas van en Prisma o en Payload? (**Prisma**: las escriben usuarios de la tienda, son transaccionales y dependen de pedidos.)
2. ¿El promedio y la lista son estáticos, cacheados o dinámicos? (**Cacheados** con tag por producto: cambian poco y no dependen del usuario.)
3. ¿"¿Ya reseñaste?" / "Escribí tu reseña" es cacheado o dinámico? (**Dinámico**: depende de la sesión → hueco con Suspense.)
4. ¿Qué invalida la action? (`updateTag(tags.reviews(productId))` y el producto, por el `aggregateRating`.)
5. ¿Dónde se verifica que el usuario compró el producto? (**En la DAL**, en la escritura, no solo en la UI.)
6. ¿Qué rate limit? (Por usuario: 5 reseñas por hora.)

---

## Bloque 2 — Datos y DAL (20 min)

```prisma
model Review {
  id        String   @id @default(cuid())
  productId String
  userId    String
  rating    Int
  body      String?
  locale    String
  hidden    Boolean  @default(false)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  product Product @relation(fields: [productId], references: [id], onDelete: Cascade)
  user    User    @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([productId, userId])
  @@index([productId, hidden, createdAt])
  @@map("reviews")
}
```

(+ relaciones `reviews Review[]` en `Product` y `User`; + `CHECK (rating BETWEEN 1 AND 5)` a mano en el SQL de la migración.)

```ts
// src/server/cache-tags.ts
reviews: (productId: string) => `reviews:${productId}`,
```

```ts
// src/server/dal/reviews.ts
import 'server-only'
import { cacheLife, cacheTag } from 'next/cache'
import { db } from '@/server/db'
import { tags } from '@/server/cache-tags'
import { requireUser, requireProducer, getCurrentUser } from './session'

export type ReviewDTO = { id: string; rating: number; body: string | null; authorName: string; createdAt: Date }
export type ReviewSummaryDTO = { average: number; count: number; latest: ReviewDTO[] }

// ───── Lectura pública (cacheada) ─────
export async function getReviewSummary(productId: string): Promise<ReviewSummaryDTO> {
  'use cache'
  cacheLife('catalog')
  cacheTag(tags.reviews(productId))

  const [agg, latest] = await Promise.all([
    db.review.aggregate({ where: { productId, hidden: false }, _avg: { rating: true }, _count: true }),
    db.review.findMany({
      where: { productId, hidden: false },
      orderBy: { createdAt: 'desc' },
      take: 10,
      select: { id: true, rating: true, body: true, createdAt: true, user: { select: { name: true } } },
    }),
  ])
  return {
    average: Math.round((agg._avg.rating ?? 0) * 10) / 10,
    count: agg._count,
    latest: latest.map((r) => ({
      id: r.id, rating: r.rating, body: r.body, createdAt: r.createdAt,
      authorName: r.user.name.split(' ')[0], // solo el nombre de pila: privacidad
    })),
  }
}

// ───── Estado del usuario actual (dinámico) ─────
export async function getMyReviewState(productId: string) {
  const user = await getCurrentUser()
  if (!user) return { kind: 'anonymous' as const }
  const [purchased, mine] = await Promise.all([
    hasReceivedProduct(user.id, productId),
    db.review.findUnique({ where: { productId_userId: { productId, userId: user.id } }, select: { rating: true, body: true } }),
  ])
  if (!purchased) return { kind: 'not_eligible' as const }
  return { kind: 'eligible' as const, mine }
}

// ───── Escritura (con regla de negocio) ─────
export async function upsertMyReview(input: { productId: string; rating: number; body?: string; locale: string }) {
  const user = await requireUser()
  if (!(await hasReceivedProduct(user.id, input.productId))) throw new Error('NOT_ELIGIBLE')

  await db.review.upsert({
    where: { productId_userId: { productId: input.productId, userId: user.id } },
    create: { ...input, userId: user.id },
    update: { rating: input.rating, body: input.body, locale: input.locale, hidden: false },
  })
  const product = await db.product.findUnique({ where: { id: input.productId }, select: { slug: true } })
  return { productSlug: product!.slug }
}

export async function setReviewHidden(reviewId: string, hidden: boolean) {
  const producer = await requireProducer()
  // Autorización: la reseña tiene que ser de un producto de ESTE productor
  const res = await db.review.updateMany({
    where: { id: reviewId, product: { producerId: producer.id } },
    data: { hidden },
  })
  if (res.count === 0) throw new Error('NOT_FOUND')
  const review = await db.review.findUnique({ where: { id: reviewId }, select: { productId: true, product: { select: { slug: true } } } })
  return { productId: review!.productId, productSlug: review!.product.slug }
}

// No exportada: regla de negocio interna
async function hasReceivedProduct(userId: string, productId: string) {
  const n = await db.orderItem.count({ where: { productId, order: { userId, status: 'delivered' } } })
  return n > 0
}
```

Preguntá: "¿por qué `authorName` es solo el nombre de pila?" (porque se publica; minimización de datos). "¿Por qué `setReviewHidden` usa `updateMany` con `product.producerId`?" (anti-IDOR por construcción, clase 17).

---

## Bloque 3 — Lectura (25 min)

```tsx
// src/features/reviews/components/product-reviews.tsx
import { getTranslations, getFormatter } from 'next-intl/server'
import { getReviewSummary } from '@/server/dal/reviews'
import { Stars } from './stars'

export async function ProductReviews({ productId }: { productId: string }) {
  const t = await getTranslations('Reviews')
  const format = await getFormatter()
  const { average, count, latest } = await getReviewSummary(productId)

  return (
    <section className="space-y-4">
      <h2 className="text-xl font-semibold">{t('title')}</h2>
      {count === 0 ? (
        <p className="text-muted-foreground">{t('empty')}</p>
      ) : (
        <>
          <p className="flex items-center gap-2"><Stars value={average} /> {t('summary', { average, count })}</p>
          <ul className="divide-y">
            {latest.map((r) => (
              <li key={r.id} className="py-3">
                <Stars value={r.rating} size="sm" />
                {r.body && <p className="mt-1 whitespace-pre-line">{r.body}</p>}
                <p className="text-xs text-muted-foreground">
                  {r.authorName} · {format.dateTime(r.createdAt, { dateStyle: 'medium' })}
                </p>
              </li>
            ))}
          </ul>
        </>
      )}
    </section>
  )
}
```

En la página de producto (dentro de `SectionErrorBoundary`, clase 21):

```tsx
<SectionErrorBoundary title={t('reviewsError')}>
  <ProductReviews productId={product.id} />                {/* cacheado → en el shell */}
  <Suspense fallback={<ReviewFormSkeleton />}>
    <MyReviewSlot productId={product.id} />                {/* sesión → hueco */}
  </Suspense>
</SectionErrorBoundary>
```

```tsx
// src/features/reviews/components/my-review-slot.tsx
import { getTranslations } from 'next-intl/server'
import { Link } from '@/i18n/navigation'
import { getMyReviewState } from '@/server/dal/reviews'
import { ReviewForm } from './review-form'

export async function MyReviewSlot({ productId }: { productId: string }) {
  const t = await getTranslations('Reviews')
  const state = await getMyReviewState(productId)
  if (state.kind === 'anonymous') return <Link href="/sign-in" className="text-sm underline">{t('signInToReview')}</Link>
  if (state.kind === 'not_eligible') return <p className="text-sm text-muted-foreground">{t('onlyBuyers')}</p>
  return <ReviewForm productId={productId} initial={state.mine} />
}
```

- `r.body` se renderiza como **texto** (`whitespace-pre-line`), no como HTML: no hace falta sanitizar porque React escapa. Pregunta para el grupo: "¿qué pasaría si quisieran permitir markdown?" (`renderUserMarkdown` de la clase 18).
- `format.dateTime` dentro de algo cacheado: la fecha es la de la reseña, no "ahora"; no hay problema de determinismo.

JSON-LD (amplía el de la clase 20):

```ts
...(summary.count > 0 && {
  aggregateRating: { '@type': 'AggregateRating', ratingValue: summary.average, reviewCount: summary.count },
}),
```

**Validación:** `npm run build` → la página de producto sigue `◐`; `curl` de la página trae las reseñas en el HTML; el formulario es el único hueco nuevo.

---

## Bloque 4 — Escritura (30 min)

```ts
// src/features/reviews/schemas.ts
import { z } from 'zod'
export const reviewSchema = z.object({
  productId: z.cuid(),
  rating: z.coerce.number().int().min(1).max(5),
  body: z.string().trim().max(1000).optional().transform((s) => (s ? s : undefined)),
})
```

```ts
// src/features/reviews/actions.ts
'use server'

import { z } from 'zod'
import { updateTag } from 'next/cache'
import { getLocale } from 'next-intl/server'
import { upsertMyReview } from '@/server/dal/reviews'
import { requireUser } from '@/server/dal/session'
import { rateLimit } from '@/server/rate-limit'
import { tags } from '@/server/cache-tags'
import { reviewSchema } from './schemas'

export type ReviewFormState = { status: 'idle' | 'ok' | 'error'; error?: 'invalid' | 'not_eligible' | 'rate_limited' }

export async function submitReviewAction(_prev: ReviewFormState, formData: FormData): Promise<ReviewFormState> {
  const parsed = reviewSchema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return { status: 'error', error: 'invalid' }

  const user = await requireUser()
  if (!(await rateLimit(`review:${user.id}`, 5, 3600)).ok) return { status: 'error', error: 'rate_limited' }

  try {
    const { productSlug } = await upsertMyReview({ ...parsed.data, locale: await getLocale() })
    updateTag(tags.reviews(parsed.data.productId))
    updateTag(tags.product(productSlug)) // por el aggregateRating del JSON-LD y la card
  } catch (err) {
    if (err instanceof Error && err.message === 'NOT_ELIGIBLE') return { status: 'error', error: 'not_eligible' }
    throw err
  }
  return { status: 'ok' }
}
```

Formulario con `useActionState` y estrellas como radios (funciona sin JS):

```tsx
'use client'
import { useActionState } from 'react'
import { useTranslations } from 'next-intl'
import { submitReviewAction, type ReviewFormState } from '../actions'

export function ReviewForm({ productId, initial }: { productId: string; initial?: { rating: number; body: string | null } | null }) {
  const t = useTranslations('Reviews')
  const [state, action, pending] = useActionState<ReviewFormState, FormData>(submitReviewAction, { status: 'idle' })

  return (
    <form action={action} className="space-y-3">
      <input type="hidden" name="productId" value={productId} />
      <fieldset className="flex gap-1">
        <legend className="sr-only">{t('rating')}</legend>
        {[1, 2, 3, 4, 5].map((n) => (
          <label key={n} className="cursor-pointer">
            <input type="radio" name="rating" value={n} defaultChecked={initial?.rating === n} className="peer sr-only" required />
            <span className="text-2xl text-muted-foreground peer-checked:text-amber-500">★</span>
          </label>
        ))}
      </fieldset>
      <textarea name="body" maxLength={1000} defaultValue={initial?.body ?? ''} className="w-full rounded border p-2" rows={3} />
      {state.status === 'error' && <p role="alert" className="text-sm text-destructive">{t(`errors.${state.error}`)}</p>}
      {state.status === 'ok' && <p className="text-sm text-green-700">{t('thanks')}</p>}
      <button disabled={pending} className="rounded bg-primary px-4 py-2 text-primary-foreground">
        {initial ? t('update') : t('submit')}
      </button>
    </form>
  )
}
```

Pregunta para el grupo: "¿hace falta `useOptimistic` acá?" (No mucho: el usuario espera confirmación al publicar una reseña; con `updateTag` ve su reseña en la lista en la misma respuesta. Optimistic conviene en interacciones de alta frecuencia, como favoritos.)

**Validación:**

1. Usuario sin pedidos entregados → no ve el form ("Solo quienes compraron pueden reseñar").
2. Usuario con pedido entregado → publica → su reseña aparece en la lista sin recargar.
3. `curl` directo a la action con un `productId` que no compró → `not_eligible` (la regla está en la DAL).
4. 6 reseñas en una hora → `rate_limited`.

---

## Bloque 5 — Moderación: ¿panel o CMS? (15 min)

Opción A: botón "Ocultar" en el panel del productor (`setReviewHidden`, con `updateTag`).
Opción B: moderación en Payload.

Respuesta esperada: **A**. Las reseñas viven en Prisma y la moderación la hace el productor dueño, que usa el panel, no el CMS. El CMS es para el equipo de marketing y contenido editorial. Mezclarlos obligaría a sincronizar dos bases.

(Tarea: implementar A con `ArchiveProductButton`-style `AlertDialog`, clase 15.)

---

## Bloque 6 — Tests y deploy (20 min)

- Vitest (DAL): no elegible no puede reseñar; segunda reseña actualiza; productor ajeno no puede ocultar (IDOR); `getReviewSummary` excluye ocultas.
- Playwright: el flujo completo con un pedido sembrado como `delivered`.
- `instant()`: la página de producto sigue mostrando nombre **y** reseñas en el shell.
- PR → CI verde → merge → Coolify despliega → verificación en producción.

---

## Bloque 7 — Proyecto final (25 min)

### 7.1 — Consigna

En equipos de 2–3, durante 4 semanas, construyen **una de estas** sobre el stack del curso (o proponen otra equivalente con aprobación):

1. **Comarca Suscripciones:** cajas mensuales de productos (planes, suscripción, pausas, pago recurrente con webhook, panel del suscriptor).
2. **Comarca Eventos:** ferias de productores con cupos, reserva de lugar, QR de entrada, check-in desde el celular.
3. **App propia de la agencia:** una versión simplificada de un proyecto real de AppWise (con datos ficticios), con el mismo stack.

Requisitos **obligatorios** (todos):

- Next ≥ 16.4.1 con `cacheComponents` y `partialPrefetching`.
- Prisma 7 + Postgres; DAL con DTOs; ningún `db` fuera de `src/server`.
- Auth con roles y al menos una regla de autorización por dueño, con test.
- i18n en al menos 2 idiomas.
- Al menos: una página `○` o `◐` con datos cacheados e invalidación por tag; un modal con intercepting routes; un route handler público o webhook firmado; un formulario con Server Action, Zod y estados de error.
- Payload con al menos una colección usada en el sitio (o justificar el Plan B).
- Tests: unit + DAL + 2 flujos e2e + 1 test de `instant()`.
- Deploy en Docker (Coolify o equivalente) con CI.
- README con decisiones de arquitectura (formato abajo).

### 7.2 — README de decisiones (obligatorio)

```markdown
## Rendering y cache
| Ruta | Símbolo de build | Qué está en el shell | Huecos dinámicos y por qué |

## Tags de cache
| Tag | Qué lo usa | Qué lo invalida (action/webhook) | updateTag o revalidateTag y por qué |

## Seguridad
- Superficies públicas (actions, handlers) y cómo se protege cada una
- Reglas de autorización por dueño y dónde viven
- CSP elegida y su costo

## Decisiones que cambiarían con 10x de tráfico
```

### 7.3 — Rúbrica (50 % de la nota del curso)

| Criterio | Peso | Excelente (100 %) | Aceptable (60 %) | Insuficiente (0 %) |
|---|---|---|---|---|
| **Rendering y cache** | 20 % | Cero `ƒ` en públicas; shells con contenido real; perfiles y tags justificados; invalidación precisa | Algunas páginas dinámicas sin necesidad o invalidación muy amplia | Cache sin invalidación, o todo dinámico |
| **Seguridad** | 20 % | DAL estricta, auth/authz en cada action/handler, tests de IDOR, input validado, CSP documentada | Falta authz en algún punto no crítico | Una action o handler sin auth, o datos privados en el payload |
| **Arquitectura y código** | 15 % | Estructura del curso respetada; actions finas; DTOs; ESLint sin excepciones | Lógica de negocio en componentes en algunos lugares | `db` en componentes, tipos de Prisma en el cliente |
| **UX e interactividad** | 10 % | Navegaciones instantáneas; optimistic donde corresponde; forms sin JS; accesibles | Loading states correctos pero mejorables | Pantallas congeladas, sin feedback |
| **i18n, SEO e imágenes** | 10 % | Metadata completa, hreflang, OG dinámicas, sitemap, `sizes` en imágenes | Metadata básica | Títulos genéricos, imágenes sin optimizar |
| **Testing** | 10 % | Pirámide completa, CI verde, tests que fallarían ante regresiones reales | Tests presentes pero superficiales | Sin tests o CI rojo |
| **Deploy y operación** | 10 % | Docker + CI + migraciones seguras + logs con digest + health check honesto | Deploy manual pero reproducible | No desplegado |
| **README de decisiones** | 5 % | Explica el porqué de cada decisión con trade-offs | Describe qué se hizo sin justificar | Falta |

### 7.4 — Defensa oral (15 min por equipo)

Preguntas posibles (elegí 4–5 por equipo, sobre **su** código):

1. Señalá un componente de tu app y decime si está en el static shell. ¿Cómo lo comprobás?
2. Si mañana alguien agrega `cookies()` en el header, ¿qué pasa en tu build? ¿Y en el modelo de Next 15?
3. Mostrame una action y explicame qué pasa si la llamo con `curl` siendo otro usuario.
4. ¿Por qué usaste `updateTag` acá y `revalidateTag` en el webhook?
5. ¿Qué viaja al navegador cuando renderizás este Client Component? Abrí el payload y mostrámelo.
6. Si escalamos a 3 instancias, ¿qué se rompe primero?
7. ¿Qué hiciste el día que salió el último boletín de seguridad de Next?
8. ¿Por qué tu proxy no valida la sesión contra la base?

> "La defensa vale tanto como el código. Un proyecto que anda pero que el equipo no puede explicar no aprueba: en la agencia, el que no puede explicar una decisión no la puede mantener."

---

## Cierre del curso

Volvé a la tabla de la clase 1 (dolor → herramienta) y recorrela con ellos. Que cada uno diga, para tres filas, dónde la aplicó en Comarca.

> "Next va a seguir cambiando: en la 17, Cache Components y Partial Prefetching van a ser el único modelo, y seguro aparecen APIs nuevas. Lo que no cambia es lo que aprendieron a preguntarse: **¿qué corre dónde, qué se cachea, quién lo invalida, quién está autorizado y qué viaja al navegador?** Con esas cinco preguntas pueden leer cualquier versión."
