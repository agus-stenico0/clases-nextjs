# Clase 13 — Server Actions

**Duración:** 2:30 hs
**Branch de salida:** `clase-13-actions`
**Lo que se llevan:** mutar datos sin escribir APIs, con formularios que funcionan aun sin JavaScript, validación con Zod compartida, estados de error y de carga con `useActionState` y `useFormStatus`, el carrito en cookie y un checkout transaccional que no vende stock que no existe. Y la regla de oro: **cada Server Action es un endpoint público**.

**Requisito:** tag `clase-12`.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:10 | El dolor: un endpoint, un fetch y tres estados para un botón |
| 0:10 – 0:35 | Qué es una Server Action por dentro |
| 0:35 – 1:05 | **En vivo:** el carrito (agregar, cambiar cantidad, quitar) |
| 1:05 – 1:35 | Formularios con validación: `useActionState`, `useFormStatus`, Zod |
| 1:35 – 2:05 | **En vivo:** checkout transaccional |
| 2:05 – 2:25 | Actions = endpoints públicos: lo que hay que verificar siempre |
| 2:25 – 2:30 | Tarea |

---

## Bloque 1 — El dolor (10 min)

Así se hacía "Agregar al carrito" en una SPA:

```tsx
// 1. route handler POST /api/cart
// 2. en el cliente:
const [loading, setLoading] = useState(false)
const [error, setError] = useState<string | null>(null)

async function add() {
  setLoading(true)
  try {
    const res = await fetch('/api/cart', { method: 'POST', body: JSON.stringify({ productId, qty }) })
    if (!res.ok) throw new Error(await res.text())
    router.refresh() // para que el badge se actualice
  } catch (e) {
    setError(String(e))
  } finally {
    setLoading(false)
  }
}
```

> "Un archivo de API, tipos duplicados en los dos lados, tres estados manuales, un `router.refresh()` que hace un segundo viaje. Y si el usuario no tiene JS cargado todavía, el botón no hace nada."

---

## Bloque 2 — Qué es una Server Action (25 min)

```ts
'use server'

export async function addToCart(productId: string, quantity: number) {
  // corre en el servidor
}
```

Lo que pasa cuando un Client Component la importa:

1. En build, Next reemplaza la función por una **referencia** con un ID cifrado y no determinístico (cambia en cada build).
2. Al llamarla desde el cliente, React hace un **POST a la ruta actual** con el ID y los argumentos serializados.
3. El servidor ejecuta la función.
4. La respuesta trae el valor de retorno **y**, si la action tocó cookies o invalidó cache (`updateTag`, `revalidatePath`, `refresh`), **el árbol actualizado de la página**. Todo en **un solo viaje**.

> "Por eso el badge del carrito se actualiza solo: la action setea la cookie, Next re-renderiza la página en el servidor en la misma respuesta. No hay `router.refresh()`."

Dos formas de declararlas:

```ts
// A. Archivo de actions (lo que usamos): todas las exportadas son actions
// src/features/cart/actions.ts
'use server'
export async function addToCart(...) {}
```

```tsx
// B. Inline en un Server Component (útil para cosas muy locales)
export default function Page() {
  async function subscribe(formData: FormData) {
    'use server'
    // ...
  }
  return <form action={subscribe}>...</form>
}
```

Características que importan:

- **Se despachan en serie** desde un mismo cliente: si el usuario aprieta tres botones, las actions corren una atrás de la otra. No sirven para **leer** datos en paralelo; para leer están los Server Components.
- **Límite de body:** 1 MB por defecto (`serverActions.bodySizeLimit` en `next.config` para subir archivos más grandes).
- **Solo POST.** No se pueden llamar con GET: eso mitiga CSRF junto con la verificación de `Origin` vs `Host` que hace Next.

---

## Bloque 3 — En vivo: el carrito (30 min)

### 3.1 — Diseño

- El carrito vive en una cookie `cart` (`httpOnly`), con **IDs y cantidades**. Nunca precios: el precio siempre se calcula del lado del servidor con la base.
- Máximo 4 KB por cookie → limitamos a 30 ítems.

### 3.2 — Schema compartido

```ts
// src/features/cart/schemas.ts
import { z } from 'zod'

export const cartLineSchema = z.object({
  p: z.string().min(1).max(40),          // productId
  q: z.number().int().min(1).max(20),    // cantidad
})

export const cartCookieSchema = z.array(cartLineSchema).max(30)
export type CartCookie = z.infer<typeof cartCookieSchema>
```

### 3.3 — Leer el carrito (DAL)

```ts
// src/server/dal/cart.ts
import 'server-only'
import { cache } from 'react'
import { cookies } from 'next/headers'
import { cartCookieSchema, type CartCookie } from '@/features/cart/schemas'
import { db } from '@/server/db'

export const CART_COOKIE = 'cart'

export const readCart = cache(async (): Promise<CartCookie> => {
  const raw = (await cookies()).get(CART_COOKIE)?.value
  if (!raw) return []
  try {
    const parsed = cartCookieSchema.safeParse(JSON.parse(raw))
    return parsed.success ? parsed.data : []
  } catch {
    return [] // cookie manipulada o corrupta: carrito vacío, no un 500
  }
})

export async function writeCart(cart: CartCookie) {
  ;(await cookies()).set(CART_COOKIE, JSON.stringify(cart), {
    httpOnly: true,
    sameSite: 'lax',
    secure: process.env.NODE_ENV === 'production',
    path: '/',
    maxAge: 60 * 60 * 24 * 14,
  })
}

export type CartLineDTO = {
  productId: string
  slug: string
  name: string
  imageUrl: string
  unitPriceCents: number
  quantity: number
  stock: number
  lineTotalCents: number
}

/** Une la cookie con la base: precios y stock REALES. */
export const getCartView = cache(async () => {
  const cart = await readCart()
  if (cart.length === 0) return { lines: [] as CartLineDTO[], totalCents: 0, count: 0 }

  const products = await db.product.findMany({
    where: { id: { in: cart.map((l) => l.p) }, isActive: true },
    select: { id: true, slug: true, name: true, imageUrl: true, priceCents: true, stock: true },
  })
  const byId = new Map(products.map((p) => [p.id, p]))

  const lines: CartLineDTO[] = cart.flatMap((l) => {
    const p = byId.get(l.p)
    if (!p) return [] // producto borrado o desactivado: desaparece del carrito
    const quantity = Math.min(l.q, p.stock)
    return quantity > 0
      ? [{ productId: p.id, slug: p.slug, name: p.name, imageUrl: p.imageUrl,
           unitPriceCents: p.priceCents, quantity, stock: p.stock,
           lineTotalCents: p.priceCents * quantity }]
      : []
  })

  return {
    lines,
    totalCents: lines.reduce((s, l) => s + l.lineTotalCents, 0),
    count: lines.reduce((s, l) => s + l.quantity, 0),
  }
})
```

> "Si el usuario edita la cookie a mano y pone `q: 999` o un `p` que no existe, el schema la rechaza o el `findMany` la ignora. **La cookie es input del usuario**, igual que un formulario."

### 3.4 — Las actions

```ts
// src/features/cart/actions.ts
'use server'

import { z } from 'zod'
import { db } from '@/server/db' // ← excepción documentada: o bien movés esto a la DAL (tarea)
import { readCart, writeCart } from '@/server/dal/cart'

const addSchema = z.object({
  productId: z.string().min(1).max(40),
  quantity: z.coerce.number().int().min(1).max(20),
})

export type CartActionResult = { ok: true } | { ok: false; error: 'invalid' | 'out_of_stock' | 'cart_full' }

export async function addToCart(input: { productId: string; quantity: number }): Promise<CartActionResult> {
  const parsed = addSchema.safeParse(input)
  if (!parsed.success) return { ok: false, error: 'invalid' }
  const { productId, quantity } = parsed.data

  const product = await db.product.findUnique({
    where: { id: productId, isActive: true },
    select: { stock: true },
  })
  if (!product || product.stock === 0) return { ok: false, error: 'out_of_stock' }

  const cart = await readCart()
  const existing = cart.find((l) => l.p === productId)
  if (!existing && cart.length >= 30) return { ok: false, error: 'cart_full' }

  const next = existing
    ? cart.map((l) => (l.p === productId ? { ...l, q: Math.min(l.q + quantity, product.stock, 20) } : l))
    : [...cart, { p: productId, q: Math.min(quantity, product.stock) }]

  await writeCart(next)
  return { ok: true }
}

export async function setQuantity(productId: string, quantity: number) {
  const q = z.number().int().min(0).max(20).parse(quantity)
  const cart = await readCart()
  await writeCart(q === 0 ? cart.filter((l) => l.p !== productId) : cart.map((l) => (l.p === productId ? { ...l, q } : l)))
}

export async function removeFromCart(productId: string) {
  const cart = await readCart()
  await writeCart(cart.filter((l) => l.p !== productId))
}
```

> **Regla del curso:** en la tarea, `db` sale de este archivo y la lógica pasa a `src/server/dal/cart.ts` (clase 6). Hoy lo dejamos acá para que se vea entero en pantalla.

### 3.5 — El botón (cliente)

```tsx
// src/features/cart/components/add-to-cart-button.tsx
'use client'

import { useState, useTransition } from 'react'
import { useTranslations } from 'next-intl'
import { toast } from 'sonner'
import { Button } from '@/components/ui/button'
import { addToCart } from '../actions'

export function AddToCartButton({ productId, stock }: { productId: string; stock: number }) {
  const t = useTranslations('Product')
  const [qty, setQty] = useState(1)
  const [isPending, startTransition] = useTransition()

  function onAdd() {
    startTransition(async () => {
      const result = await addToCart({ productId, quantity: qty })
      if (result.ok) toast.success(t('added'))
      else toast.error(t(`errors.${result.error}`))
    })
  }

  return (
    <div className="flex items-center gap-3">
      <input type="number" min={1} max={Math.min(stock, 20)} value={qty}
             onChange={(e) => setQty(Number(e.target.value))} className="w-16 rounded border px-2 py-1" />
      <Button onClick={onAdd} disabled={isPending || stock === 0}>
        {isPending ? '…' : t('addToCart')}
      </Button>
    </div>
  )
}
```

Agregá `<Toaster />` de `sonner` en el layout (dentro de `Providers`) y las claves `Product.errors.*` en los mensajes.

**Validación:**

1. Agregá un producto → toast + el badge del header cambia **sin recargar**.
2. Network: un solo POST a `/es/products/queso-azul` con header `Next-Action`. La respuesta trae el RSC con el header actualizado.
3. Editá la cookie a `[{"p":"x","q":999}]` → `/cart` muestra carrito vacío, sin error.

### 3.6 — La página del carrito: formularios sin JS

```tsx
// src/features/cart/components/cart-line.tsx (Server Component)
import { removeFromCart, setQuantity } from '../actions'
import { Price } from '@/components/price'

export function CartLine({ line }: { line: CartLineDTO }) {
  return (
    <li className="flex items-center gap-4 py-4">
      <span className="flex-1">{line.name}</span>
      <form action={async (fd: FormData) => { 'use server'; await setQuantity(line.productId, Number(fd.get('q'))) }}>
        <select name="q" defaultValue={line.quantity} className="rounded border px-2 py-1">
          {Array.from({ length: Math.min(line.stock, 20) }, (_, i) => i + 1).map((n) => (
            <option key={n} value={n}>{n}</option>
          ))}
        </select>
        <button className="ml-2 text-sm underline">Actualizar</button>
      </form>
      <Price cents={line.lineTotalCents} />
      <form action={removeFromCart.bind(null, line.productId)}>
        <button className="text-sm text-destructive">Quitar</button>
      </form>
    </li>
  )
}
```

> "Desactiven JavaScript en DevTools y prueben: **funciona**. Un `<form action={serverAction}>` es un form HTML de verdad que hace POST. Cuando hay JS, React lo intercepta y lo hace sin recargar. Eso es *progressive enhancement*, gratis."

Sobre `bind`:

> "`removeFromCart.bind(null, line.productId)` crea una action con el primer argumento fijado. **Ese argumento viaja al navegador y vuelve.** El usuario lo puede cambiar. Para el carrito no importa (solo afecta su propia cookie), pero en el panel, `deleteProduct.bind(null, productId)` sin verificar que el producto sea suyo es un IDOR (clase 18)."

---

## Bloque 4 — Formularios con validación (30 min)

Caso: el productor crea un producto en `/dashboard/products/new`.

### 4.1 — Schema compartido cliente/servidor

```ts
// src/features/producer-dashboard/schemas.ts
import { z } from 'zod'

export const productFormSchema = z.object({
  name: z.string().trim().min(3, 'Mínimo 3 caracteres').max(80),
  description: z.string().trim().min(20, 'Contá un poco más (20+ caracteres)').max(4000),
  price: z.coerce.number().positive('Tiene que ser mayor a 0').max(10_000_000),
  stock: z.coerce.number().int().min(0).max(100_000),
  categorySlug: z.string().min(1, 'Elegí una categoría'),
})

export type ProductFormValues = z.infer<typeof productFormSchema>

export type FormState = {
  status: 'idle' | 'error' | 'success'
  message?: string
  fieldErrors?: Partial<Record<keyof ProductFormValues, string[]>>
  values?: Partial<Record<keyof ProductFormValues, string>>
}
```

### 4.2 — La action con firma `(prevState, formData)`

```ts
// src/features/producer-dashboard/actions.ts
'use server'

import { z } from 'zod'
import { updateTag } from 'next/cache'
import { redirect } from '@/i18n/navigation'
import { getLocale } from 'next-intl/server'
import { createProductForCurrentProducer } from '@/server/dal/producer-products'
import { tags } from '@/server/cache-tags'
import { productFormSchema, type FormState } from './schemas'

export async function createProductAction(_prev: FormState, formData: FormData): Promise<FormState> {
  const raw = Object.fromEntries(formData) as Record<string, string>
  const parsed = productFormSchema.safeParse(raw)

  if (!parsed.success) {
    return {
      status: 'error',
      message: 'Revisá los campos marcados.',
      fieldErrors: z.flattenError(parsed.error).fieldErrors,
      values: raw, // para no perder lo que escribió
    }
  }

  let slug: string
  try {
    const created = await createProductForCurrentProducer(parsed.data) // la DAL verifica sesión y rol (clase 17)
    slug = created.slug
    updateTag(tags.catalog)
  } catch (err) {
    if (err instanceof Error && err.message === 'DUPLICATE_NAME') {
      return { status: 'error', fieldErrors: { name: ['Ya tenés un producto con ese nombre'] }, values: raw }
    }
    throw err // error inesperado → error boundary
  }

  // redirect FUERA del try (lanza una excepción especial, clase 2)
  redirect({ href: `/dashboard/products?created=${slug}`, locale: await getLocale() })
}
```

> "Errores **esperados** (validación, duplicados) se **devuelven** como datos. Errores **inesperados** (la base se cayó) se **lanzan** y los atrapa el `error.tsx`. Nunca devuelvan `err.message` de un error inesperado al cliente: puede tener SQL, rutas internas o datos."

### 4.3 — El formulario

```tsx
// src/features/producer-dashboard/components/product-form.tsx
'use client'

import { useActionState } from 'react'
import { useFormStatus } from 'react-dom'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Textarea } from '@/components/ui/textarea'
import { Field, FieldError, FieldLabel } from '@/components/ui/field'
import { createProductAction } from '../actions'
import type { FormState } from '../schemas'

const initialState: FormState = { status: 'idle' }

export function ProductForm({ categories }: { categories: { slug: string; name: string }[] }) {
  const [state, formAction] = useActionState(createProductAction, initialState)
  const err = (k: keyof NonNullable<FormState['fieldErrors']>) => state.fieldErrors?.[k]?.[0]

  return (
    <form action={formAction} className="grid max-w-xl gap-5" noValidate>
      {state.status === 'error' && state.message && (
        <p role="alert" className="rounded bg-destructive/10 p-3 text-sm text-destructive">{state.message}</p>
      )}

      <Field data-invalid={!!err('name')}>
        <FieldLabel htmlFor="name">Nombre</FieldLabel>
        <Input id="name" name="name" defaultValue={state.values?.name} aria-invalid={!!err('name')} />
        <FieldError>{err('name')}</FieldError>
      </Field>

      <Field data-invalid={!!err('description')}>
        <FieldLabel htmlFor="description">Descripción</FieldLabel>
        <Textarea id="description" name="description" rows={5} defaultValue={state.values?.description} />
        <FieldError>{err('description')}</FieldError>
      </Field>

      <div className="grid grid-cols-2 gap-4">
        <Field data-invalid={!!err('price')}>
          <FieldLabel htmlFor="price">Precio (ARS)</FieldLabel>
          <Input id="price" name="price" type="number" step="0.01" defaultValue={state.values?.price} />
          <FieldError>{err('price')}</FieldError>
        </Field>
        <Field data-invalid={!!err('stock')}>
          <FieldLabel htmlFor="stock">Stock</FieldLabel>
          <Input id="stock" name="stock" type="number" defaultValue={state.values?.stock ?? '0'} />
          <FieldError>{err('stock')}</FieldError>
        </Field>
      </div>

      <Field data-invalid={!!err('categorySlug')}>
        <FieldLabel htmlFor="categorySlug">Categoría</FieldLabel>
        <select id="categorySlug" name="categorySlug" defaultValue={state.values?.categorySlug ?? ''}
                className="h-9 rounded-md border bg-transparent px-3">
          <option value="" disabled>Elegí…</option>
          {categories.map((c) => <option key={c.slug} value={c.slug}>{c.name}</option>)}
        </select>
        <FieldError>{err('categorySlug')}</FieldError>
      </Field>

      <SubmitButton />
    </form>
  )
}

function SubmitButton() {
  const { pending } = useFormStatus() // estado del <form> padre
  return <Button type="submit" disabled={pending}>{pending ? 'Guardando…' : 'Crear producto'}</Button>
}
```

Puntos:

- `useActionState` devuelve `[state, formAction, isPending]`. El `state` es lo que devolvió la última ejecución.
- `useFormStatus` **tiene que estar en un componente hijo** del `<form>`; en el mismo componente no ve nada.
- `defaultValue={state.values?.x}`: cuando falla la validación, el form no pierde lo escrito. (React 19 resetea los forms no controlados después de una action; por eso devolvemos los valores.)
- El `<select>` nativo y no el `Select` de shadcn: el nativo funciona sin JS y viaja en el `FormData` sin hacer nada. Si usan el de shadcn, necesitan un `<input type="hidden">`.

> "Validación del lado del cliente para UX (clase 14 con react-hook-form, opcional), validación del lado del servidor **siempre**. El cliente es un navegador que controla otra persona."

---

## Bloque 5 — En vivo: checkout transaccional (30 min)

El problema: dos personas compran el último queso al mismo tiempo. Lo vieron en el módulo de NestJS con `race.ts`. Acá la misma solución, en una action.

Todavía no hay login (clase 17). Para que el pedido tenga dueño, un placeholder **con la misma firma** que va a tener la función real:

```ts
// src/server/dal/session.ts  — PROVISORIO hasta la clase 17
import 'server-only'
import { cache } from 'react'
import { db } from '@/server/db'

export const requireUser = cache(async () => {
  // Agregá al seed un cliente demo: cliente@comarca.dev
  const user = await db.user.findUnique({ where: { email: 'cliente@comarca.dev' }, select: { id: true, role: true } })
  if (!user) throw new Error('Falta el usuario demo en el seed')
  return user
})
```

> "Todas las funciones de la DAL que necesitan usuario llaman a `requireUser()`. En la clase 17 cambiamos **solo este archivo** y todo el sistema pasa a usar la sesión real. Esa es la ventaja de que la identidad se resuelva adentro de la DAL y no se pase como argumento."

```ts
// src/server/dal/orders.ts
import 'server-only'
import { db } from '@/server/db'
import { readCart } from './cart'
import { requireUser } from './session' // clase 17; hoy devuelve un usuario demo del seed

export class OutOfStockError extends Error {
  constructor(public productIds: string[]) { super('OUT_OF_STOCK') }
}

export async function placeOrderFromCart() {
  const user = await requireUser()
  const cart = await readCart()
  if (cart.length === 0) throw new Error('EMPTY_CART')

  return db.$transaction(async (tx) => {
    const products = await tx.product.findMany({
      where: { id: { in: cart.map((l) => l.p) }, isActive: true },
      select: { id: true, priceCents: true },
    })
    const price = new Map(products.map((p) => [p.id, p.priceCents]))

    const failed: string[] = []
    for (const line of cart) {
      // Descuento condicional: solo si alcanza. Atómico en Postgres.
      const res = await tx.product.updateMany({
        where: { id: line.p, isActive: true, stock: { gte: line.q } },
        data: { stock: { decrement: line.q } },
      })
      if (res.count === 0) failed.push(line.p)
    }
    if (failed.length > 0) throw new OutOfStockError(failed) // rollback de todo

    const items = cart.map((l) => ({ productId: l.p, quantity: l.q, unitPriceCents: price.get(l.p)! }))
    const totalCents = items.reduce((s, i) => s + i.unitPriceCents * i.quantity, 0)

    return tx.order.create({
      data: { userId: user.id, totalCents, items: { create: items } },
      select: { id: true },
    })
  })
}
```

```ts
// src/features/checkout/actions.ts
'use server'

import { updateTag } from 'next/cache'
import { getLocale } from 'next-intl/server'
import { redirect } from '@/i18n/navigation'
import { OutOfStockError, placeOrderFromCart } from '@/server/dal/orders'
import { writeCart } from '@/server/dal/cart'
import { tags } from '@/server/cache-tags'

export type CheckoutState = { error?: 'out_of_stock' | 'empty' }

export async function placeOrderAction(_prev: CheckoutState): Promise<CheckoutState> {
  let orderId: string
  try {
    const order = await placeOrderFromCart()
    orderId = order.id
  } catch (err) {
    if (err instanceof OutOfStockError) return { error: 'out_of_stock' }
    if (err instanceof Error && err.message === 'EMPTY_CART') return { error: 'empty' }
    throw err
  }

  await writeCart([])
  updateTag(tags.catalog) // el stock cambió: que no queden "Quedan 3" viejos en lo cacheado
  redirect({ href: `/account/orders/${orderId}?new=1`, locale: await getLocale() })
}
```

**Validación de concurrencia:** adaptá el script `race.ts` del módulo NestJS para disparar 20 checkouts simultáneos sobre un producto con stock 5 (vas a tener que llamar a la action vía `fetch` con el header `Next-Action`; alternativa más simple: llamar a `placeOrderFromCart` desde un script con 20 usuarios distintos). Resultado esperado: **5 pedidos, 15 errores de stock, stock final 0**, nunca negativo.

> "Fíjense en el detalle: el precio del pedido sale de la base **dentro de la transacción**, no del carrito ni del cliente. El total que ve el usuario en `/cart` es informativo; el que se cobra es este."

---

## Bloque 6 — Actions = endpoints públicos (20 min)

La doc de Data Security lo dice sin vueltas: una Server Action exportada es alcanzable con un POST directo, **aunque no la uses en ningún lado** (Next elimina del bundle las no usadas, pero no lo tomen como protección).

Checklist para **cada** action (escribila en el aula):

```text
□ ¿Valida TODOS los argumentos con Zod? (FormData, bind args, objetos)
□ ¿Verifica la sesión ADENTRO? (no alcanza con que la página la verifique)
□ ¿Verifica que el usuario sea dueño del recurso? (IDOR)
□ ¿Devuelve solo lo que la UI necesita? (nada de filas de la base)
□ ¿Los errores inesperados se lanzan sin exponer el mensaje?
□ ¿La lógica de datos vive en la DAL (server-only) y la action es una capa fina?
□ Si es cara (mails, pagos), ¿tiene rate limit? (clase 18)
```

Demo del punto 2, sin auth todavía:

```tsx
// Page "protegida" (supongamos)
export default async function AdminPage() {
  // ...la página chequea que seas admin...
  return (
    <form action={async () => { 'use server'; await db.product.deleteMany() }}>
      <button>Borrar todo</button>
    </form>
  )
}
```

> "Si alguien obtiene el ID de esa action (por ejemplo, porque estuvo logueado como admin y miró el HTML), la puede invocar **después**, sin ser admin, porque la action no chequea nada. La verificación de la página controla qué **se muestra**; la action es otra puerta. En la clase 17 todas las actions de Comarca van a verificar adentro."

Y sobre las closures:

> "Cuando una action inline usa variables del componente (como `line.productId` en el ejemplo del carrito), Next **cifra** esas variables al mandarlas al cliente, con una clave que se regenera en cada build. Es una protección extra, no un reemplazo de validar. Y si corren varias instancias (clase 22), todas tienen que compartir la clave: `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`."

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| El redirect no ocurre y aparece `NEXT_REDIRECT` en un toast | `redirect` dentro de `try/catch` | Hacerlo después del try |
| `useFormStatus` siempre `pending: false` | Está en el mismo componente que el `<form>` | Componente hijo |
| El form se vacía después de un error | Reset automático de React 19 | Devolver `values` y usar `defaultValue` |
| `Body exceeded 1 MB limit` | Subida de imagen | `serverActions.bodySizeLimit` o subir directo a storage |
| La UI no se actualiza después de mutar datos cacheados | Faltó `updateTag` | Invalidar el tag correcto |

---

## Tarea

1. Mové toda la lógica de `features/cart/actions.ts` a `server/dal/cart.ts`; la action queda en 3 líneas por función.
2. Página `/cart` completa: líneas con cantidad y quitar, total, botón "Finalizar compra" con `prefetch` hacia `/checkout`.
3. `/checkout` con resumen y el form de `placeOrderAction` (`useActionState`) mostrando el error de stock con los productos afectados.
4. Formulario de **edición** de producto reutilizando `ProductForm` (modo create/edit) con `updateTag` de los tags afectados (clase 11).
5. Probá todo el flujo con JavaScript desactivado y anotá qué funciona y qué no.

**Terminado cuando:** el script de concurrencia nunca deja stock negativo y el carrito funciona sin JS.
