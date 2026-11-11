# Clase 14 — UI optimista, transiciones y formularios ricos

**Duración:** 2:30 hs
**Branch de salida:** `clase-14-optimistic`
**Lo que se llevan:** interfaces que responden en el acto aunque el servidor tarde (`useOptimistic`), control fino de estados pendientes con `useTransition`, validación en el cliente con react-hook-form **sin duplicar** la del servidor, feedback con toasts, y animaciones de navegación con View Transitions de React 19.3.

**Requisito:** tag `clase-13`.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:10 | El dolor: el corazón de favoritos tarda 400 ms en llenarse |
| 0:10 – 0:45 | `useOptimistic` a fondo: favoritos y cantidades del carrito |
| 0:45 – 1:05 | `useTransition`: qué es "pendiente" y quién lo ve |
| 1:05 – 1:45 | react-hook-form + Zod + Server Action: un schema, dos validaciones |
| 1:45 – 2:00 | Toasts y patrones de feedback |
| 2:00 – 2:25 | View Transitions y `<Activity>` |
| 2:25 – 2:30 | Tarea |

---

## Bloque 1 — El dolor (10 min)

Botón de favorito "honesto": espera a la action y después pinta.

```tsx
'use client'
export function FavoriteButton({ productId, isFavorite }: Props) {
  const [isPending, startTransition] = useTransition()
  return (
    <button onClick={() => startTransition(() => toggleFavorite(productId))} disabled={isPending}>
      <Heart className={isFavorite ? 'fill-red-500' : ''} />
    </button>
  )
}
```

Con "Fast 4G": click → nada → 400 ms → corazón lleno.

> "En una app nativa, el corazón se llena **en el momento del toque**. Si el servidor falla, vuelve atrás y te avisa. Eso es UI optimista: asumimos que va a salir bien, porque el 99,9 % de las veces sale bien."

---

## Bloque 2 — `useOptimistic` (35 min)

### 2.1 — La API

```ts
const [optimisticValue, setOptimistic] = useOptimistic(realValue, reducer?)
```

- `realValue`: lo que viene del servidor (props).
- Mientras haya una **transición en curso** (una action ejecutándose), `optimisticValue` muestra lo que vos le pusiste con `setOptimistic`.
- Cuando la transición termina, vuelve a mostrar `realValue`… que para entonces ya es el nuevo, porque la action actualizó el árbol.
- Si la action falla, vuelve al `realValue` viejo. **Rollback automático.**

### 2.2 — Favoritos

DAL y action (la DAL usa `requireUser`, que en la clase 17 pasa a ser la sesión real):

```ts
// src/server/dal/favorites.ts
import 'server-only'
import { db } from '@/server/db'
import { requireUser } from './session'

export async function toggleFavoriteForCurrentUser(productId: string) {
  const user = await requireUser()
  const key = { userId_productId: { userId: user.id, productId } }
  const existing = await db.favorite.findUnique({ where: key })
  if (existing) await db.favorite.delete({ where: key })
  else await db.favorite.create({ data: { userId: user.id, productId } })
  return { userId: user.id, isFavorite: !existing }
}

export async function listFavoriteIdsForCurrentUser() {
  const user = await requireUser()
  const rows = await db.favorite.findMany({ where: { userId: user.id }, select: { productId: true } })
  return new Set(rows.map((r) => r.productId))
}
```

```ts
// src/features/favorites/actions.ts
'use server'

import { z } from 'zod'
import { updateTag } from 'next/cache'
import { toggleFavoriteForCurrentUser } from '@/server/dal/favorites'
import { tags } from '@/server/cache-tags'

export async function toggleFavorite(productId: string) {
  const id = z.string().min(1).max(40).parse(productId)
  const { userId } = await toggleFavoriteForCurrentUser(id)
  updateTag(tags.favorites(userId)) // agregá favorites: (id) => `favorites:${id}` a cache-tags.ts
}
```

Botón:

```tsx
// src/features/favorites/components/favorite-button.tsx
'use client'

import { useOptimistic, useTransition } from 'react'
import { Heart } from 'lucide-react'
import { toast } from 'sonner'
import { cn } from '@/lib/utils'
import { toggleFavorite } from '../actions'

export function FavoriteButton({ productId, isFavorite }: { productId: string; isFavorite: boolean }) {
  const [optimistic, setOptimistic] = useOptimistic(isFavorite)
  const [, startTransition] = useTransition()

  function onClick() {
    startTransition(async () => {
      setOptimistic(!optimistic) // ← se pinta YA
      try {
        await toggleFavorite(productId)
      } catch {
        toast.error('No pudimos guardar tu favorito')
        // no hace falta "deshacer": al terminar la transición vuelve al valor real
      }
    })
  }

  return (
    <button onClick={onClick} aria-pressed={optimistic} aria-label="Favorito" className="rounded-full p-2 hover:bg-muted">
      <Heart className={cn('size-5 transition', optimistic && 'fill-red-500 text-red-500')} />
    </button>
  )
}
```

Reglas que siempre se olvidan:

1. **`setOptimistic` tiene que llamarse dentro de una transición** (`startTransition` o una action de form). Fuera de una, React avisa y no hace nada útil.
2. **El valor real tiene que actualizarse al terminar.** Si la action no invalida nada (`updateTag`/`refresh`), cuando termina la transición el corazón **vuelve** al valor viejo, porque la prop no cambió. Es el bug más común: "se llena y se vacía solo".

**Validación:**

- Con throttling, el corazón se llena al instante.
- Para probar el rollback, hacé que la action tire un error con 50 % de probabilidad: el corazón se llena, y a los 400 ms vuelve y aparece el toast.

### 2.3 — Con reducer: el carrito

Cuando el estado optimista es una lista, se usa el reducer:

```tsx
'use client'

import { useOptimistic, useTransition } from 'react'
import { setQuantity, removeFromCart } from '../actions'
import type { CartLineDTO } from '@/server/dal/cart'

type Op = { type: 'qty'; productId: string; quantity: number } | { type: 'remove'; productId: string }

export function CartLines({ lines }: { lines: CartLineDTO[] }) {
  const [optimisticLines, apply] = useOptimistic(lines, (state: CartLineDTO[], op: Op) => {
    if (op.type === 'remove') return state.filter((l) => l.productId !== op.productId)
    return state.map((l) =>
      l.productId === op.productId
        ? { ...l, quantity: op.quantity, lineTotalCents: l.unitPriceCents * op.quantity }
        : l,
    )
  })
  const [isPending, startTransition] = useTransition()

  const total = optimisticLines.reduce((s, l) => s + l.lineTotalCents, 0)

  return (
    <div aria-busy={isPending}>
      {optimisticLines.map((l) => (
        <div key={l.productId} className="flex items-center gap-3 py-3">
          <span className="flex-1">{l.name}</span>
          <select
            value={l.quantity}
            onChange={(e) => {
              const quantity = Number(e.target.value)
              startTransition(async () => {
                apply({ type: 'qty', productId: l.productId, quantity })
                await setQuantity(l.productId, quantity)
              })
            }}
          >
            {Array.from({ length: Math.min(l.stock, 20) }, (_, i) => i + 1).map((n) => <option key={n}>{n}</option>)}
          </select>
          <button
            onClick={() =>
              startTransition(async () => {
                apply({ type: 'remove', productId: l.productId })
                await removeFromCart(l.productId)
              })
            }
          >
            Quitar
          </button>
        </div>
      ))}
      <p className={isPending ? 'opacity-60' : ''}>Total: {/* <Price cents={total} /> */}</p>
    </div>
  )
}
```

> "El total optimista se calcula en el cliente para que la pantalla sea coherente **mientras** viaja la action. El total de verdad lo vuelve a calcular el servidor con los precios de la base. Si difieren (cambió un precio), gana el servidor al terminar."

---

## Bloque 3 — `useTransition` (20 min)

Una **transición** es una actualización que React puede hacer "en segundo plano": no bloquea la UI y tiene un estado `isPending`.

Cosas que **ya son transiciones** sin que hagas nada:

- Navegaciones con `<Link>` y `router.push`.
- La action de un `<form action={...}>`.

Cosas que **tenés que envolver**:

- Llamar a una action desde un `onClick`/`onChange`.
- `router.replace` cuando querés saber si está pendiente (el selector de idioma de la clase 8).

```tsx
const [isPending, startTransition] = useTransition()

<Button onClick={() => startTransition(() => archiveOrder(id))} disabled={isPending}>
  {isPending ? 'Archivando…' : 'Archivar'}
</Button>
```

| Hook | Qué te dice |
|---|---|
| `useTransition().isPending` | Si **esta** transición (la que inició este componente) está en curso |
| `useFormStatus().pending` | Si el `<form>` padre está enviando |
| `useActionState()[2]` | Si **esa** action está en curso |
| `useLinkStatus()` (de `next/link`) | Si el `<Link>` en el que está anidado está navegando |

```tsx
'use client'
import { useLinkStatus } from 'next/link'

export function LinkSpinner() {
  const { pending } = useLinkStatus()
  return pending ? <span className="ml-1 inline-block size-3 animate-spin rounded-full border" /> : null
}

// <Link href="/x">Ir <LinkSpinner /></Link>
```

> "Con navegaciones instantáneas (clase 12) `useLinkStatus` casi nunca se ve. Es la red de seguridad para cuando una navegación sí bloquea."

---

## Bloque 4 — react-hook-form + Zod + Server Action (40 min)

¿Por qué sumar validación en el cliente si ya validamos en el servidor?

- Errores **mientras escribe** (o al salir del campo), sin viajar al servidor.
- Formularios complejos: campos dependientes, arrays dinámicos (variantes del producto).

Lo que **no** queremos: dos schemas que se desincronicen. Usamos **el mismo** `productFormSchema` de la clase 13.

```bash
npm i react-hook-form @hookform/resolvers
```

```tsx
// src/features/producer-dashboard/components/product-form-rhf.tsx
'use client'

import { useTransition } from 'react'
import { useForm, Controller } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { toast } from 'sonner'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Textarea } from '@/components/ui/textarea'
import { Field, FieldError, FieldLabel } from '@/components/ui/field'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { productFormSchema, type ProductFormValues, type FormState } from '../schemas'
import { createProductAction } from '../actions'

type Props = { categories: { slug: string; name: string }[]; defaultValues?: Partial<ProductFormValues> }

export function ProductFormRHF({ categories, defaultValues }: Props) {
  const [isPending, startTransition] = useTransition()
  const form = useForm<ProductFormValues>({
    resolver: zodResolver(productFormSchema),
    mode: 'onBlur',
    defaultValues: { name: '', description: '', price: 0, stock: 0, categorySlug: '', ...defaultValues },
  })

  const onSubmit = form.handleSubmit((values) => {
    const fd = new FormData()
    Object.entries(values).forEach(([k, v]) => fd.append(k, String(v)))

    startTransition(async () => {
      const result: FormState = await createProductAction({ status: 'idle' }, fd)
      // Si la action redirige, no llegamos acá. Si devuelve errores, los mapeamos al form.
      if (result.status === 'error') {
        for (const [field, messages] of Object.entries(result.fieldErrors ?? {})) {
          form.setError(field as keyof ProductFormValues, { message: messages?.[0] })
        }
        if (result.message) toast.error(result.message)
      }
    })
  })

  const { errors } = form.formState

  return (
    <form onSubmit={onSubmit} className="grid max-w-xl gap-5" noValidate>
      <Field data-invalid={!!errors.name}>
        <FieldLabel htmlFor="name">Nombre</FieldLabel>
        <Input id="name" {...form.register('name')} aria-invalid={!!errors.name} />
        <FieldError>{errors.name?.message}</FieldError>
      </Field>

      <Field data-invalid={!!errors.description}>
        <FieldLabel htmlFor="description">Descripción</FieldLabel>
        <Textarea id="description" rows={5} {...form.register('description')} />
        <FieldError>{errors.description?.message}</FieldError>
      </Field>

      <div className="grid grid-cols-2 gap-4">
        <Field data-invalid={!!errors.price}>
          <FieldLabel htmlFor="price">Precio</FieldLabel>
          <Input id="price" type="number" step="0.01" {...form.register('price')} />
          <FieldError>{errors.price?.message}</FieldError>
        </Field>
        <Field data-invalid={!!errors.stock}>
          <FieldLabel htmlFor="stock">Stock</FieldLabel>
          <Input id="stock" type="number" {...form.register('stock')} />
          <FieldError>{errors.stock?.message}</FieldError>
        </Field>
      </div>

      <Field data-invalid={!!errors.categorySlug}>
        <FieldLabel>Categoría</FieldLabel>
        <Controller
          control={form.control}
          name="categorySlug"
          render={({ field }) => (
            <Select value={field.value} onValueChange={field.onChange}>
              <SelectTrigger><SelectValue placeholder="Elegí…" /></SelectTrigger>
              <SelectContent>
                {categories.map((c) => <SelectItem key={c.slug} value={c.slug}>{c.name}</SelectItem>)}
              </SelectContent>
            </Select>
          )}
        />
        <FieldError>{errors.categorySlug?.message}</FieldError>
      </Field>

      <Button type="submit" disabled={isPending}>{isPending ? 'Guardando…' : 'Guardar'}</Button>
    </form>
  )
}
```

Comparalo con la versión de la clase 13:

| `useActionState` + form nativo | react-hook-form |
|---|---|
| Funciona sin JS | Necesita JS |
| Validación al enviar (servidor) | Validación al salir del campo (cliente) + servidor |
| Ideal para forms cortos | Ideal para forms largos o dinámicos |

> "Los dos llaman a **la misma action** y validan con **el mismo schema**. La decisión es de UX, no de seguridad. El servidor valida siempre."

`z.coerce.number()` en el schema hace que funcione igual con strings del `FormData` y con números del cliente.

---

## Bloque 5 — Feedback (15 min)

Patrones con `sonner`:

```ts
// Éxito simple
toast.success('Producto creado')

// Promesa: loading → success/error
toast.promise(archiveProduct(id), {
  loading: 'Archivando…',
  success: 'Archivado',
  error: 'No se pudo archivar',
})

// Con deshacer (combina con optimistic)
toast('Producto quitado del carrito', {
  action: { label: 'Deshacer', onClick: () => startTransition(() => addToCart({ productId, quantity })) },
})
```

Toast después de un `redirect` (la action ya no vuelve al cliente para mostrarlo): usamos un query param y lo leemos en la página destino.

```tsx
// /dashboard/products?created=queso-azul
'use client'
import { useEffect } from 'react'
import { useSearchParams } from 'next/navigation'
import { useRouter, usePathname } from '@/i18n/navigation'
import { toast } from 'sonner'

export function CreatedToast() {
  const params = useSearchParams()
  const router = useRouter()
  const pathname = usePathname()
  useEffect(() => {
    const slug = params.get('created')
    if (!slug) return
    toast.success(`Producto "${slug}" creado`)
    router.replace(pathname) // limpia la URL
  }, [params, router, pathname])
  return null
}
```

(Con su `<Suspense>`, por el `useSearchParams`.)

---

## Bloque 6 — View Transitions y `<Activity>` (25 min)

### 6.1 — View Transitions (estables en React 19.3)

El navegador sabe animar entre dos estados de la página (API `document.startViewTransition`). React las integra con el componente `<ViewTransition>`: cuando lo que está adentro cambia **dentro de una transición** (navegación, action, `startTransition`), React anima el cambio.

```tsx
import { ViewTransition } from 'react'

// En la card del listado
<ViewTransition name={`product-image-${p.slug}`}>
  <Image src={p.imageUrl} alt={p.name} width={400} height={300} />
</ViewTransition>

// En el detalle
<ViewTransition name={`product-image-${product.slug}`}>
  <Image src={product.imageUrl} alt={product.name} width={800} height={800} />
</ViewTransition>
```

Mismo `name` en las dos páginas → al navegar, la imagen "vuela" de la card al detalle.

> **Nota para el docente:** la integración del router de Next con View Transitions se activaba con `experimental.viewTransition` en 15/16.0. Revisá en `node_modules/next/dist/docs` (o en `AGENTS.md`) si en tu versión exacta sigue haciendo falta el flag. React 19.3 la estabilizó del lado de React.

CSS para controlar la animación:

```css
::view-transition-old(root),
::view-transition-new(root) {
  animation-duration: 200ms;
}

@media (prefers-reduced-motion: reduce) {
  ::view-transition-group(*) { animation: none !important; }
}
```

> "Siempre respeten `prefers-reduced-motion`. Hay gente a la que las animaciones le generan mareo de verdad."

### 6.2 — `<Activity>`: ocultar sin desmontar

En el panel, tabs "Datos / Fotos / Stock" del producto. Con renderizado condicional, cambiar de tab **destruye** el estado del form de la tab anterior. Con `<Activity>`:

```tsx
'use client'
import { Activity, useState } from 'react'

export function ProductEditorTabs({ data, photos, stock }: { data: React.ReactNode; photos: React.ReactNode; stock: React.ReactNode }) {
  const [tab, setTab] = useState<'data' | 'photos' | 'stock'>('data')
  return (
    <>
      <nav className="flex gap-4">
        <button onClick={() => setTab('data')}>Datos</button>
        <button onClick={() => setTab('photos')}>Fotos</button>
        <button onClick={() => setTab('stock')}>Stock</button>
      </nav>
      <Activity mode={tab === 'data' ? 'visible' : 'hidden'}>{data}</Activity>
      <Activity mode={tab === 'photos' ? 'visible' : 'hidden'}>{photos}</Activity>
      <Activity mode={tab === 'stock' ? 'visible' : 'hidden'}>{stock}</Activity>
    </>
  )
}
```

- `hidden` = `display: none`, **mantiene el estado** y limpia los efectos (se pausan suscripciones, timers).
- Next usa esto internamente: mantiene montadas (ocultas) algunas páginas visitadas para que volver atrás sea instantáneo. (Payload lo tuvo que tener en cuenta en su admin, clase 19.)

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| "An optimistic state update occurred outside a transition" | `setOptimistic` fuera de `startTransition` | Envolverlo |
| El corazón se llena y se vacía solo | La action no actualiza el valor real | `updateTag`/`refresh` en la action |
| RHF manda `"12"` y el schema espera número | Falta `coerce` | `z.coerce.number()` |
| Errores del servidor no aparecen en RHF | No se mapean | `form.setError` con `fieldErrors` |
| La View Transition no anima | Cambio fuera de una transición o flag de Next faltante | Ver nota del docente |

---

## Tarea

1. `FavoriteButton` optimista en la card y en el detalle (cuando no hay sesión, por ahora, que muestre un toast "Ingresá para guardar favoritos").
2. Carrito con `CartLines` optimista, incluido "Deshacer" al quitar.
3. Editor de producto con `ProductFormRHF` + tabs con `<Activity>` (los cambios de una tab no se pierden al cambiar).
4. View Transition de la imagen card → detalle, respetando `prefers-reduced-motion`.

**Terminado cuando:** con "Fast 4G", ninguna interacción del carrito ni de favoritos tiene demora visible, y forzando errores el estado vuelve atrás con un toast.
