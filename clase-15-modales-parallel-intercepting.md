# Clase 15 — Modales: parallel routes e intercepting routes

**Duración:** 2:30 hs
**Branch de salida:** `clase-15-modales`
**Lo que se llevan:** el "quick view" de producto que se abre en un modal desde la grilla **con URL propia** (compartible, recargable, cerrable con "atrás"), dashboards con secciones independientes usando slots paralelos, y el criterio para decidir cuándo un modal va por URL y cuándo por estado.

**Requisito:** tag `clase-14`.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:10 | El dolor: el modal que se pierde al recargar |
| 0:10 – 0:35 | Parallel routes: slots, `default.tsx` y navegación suave vs dura |
| 0:35 – 0:55 | Intercepting routes: `(.)`, `(..)`, `(...)` |
| 0:55 – 1:40 | **En vivo:** quick view de producto en modal |
| 1:40 – 2:05 | Slots paralelos en el panel: secciones con su propio loading y error |
| 2:05 – 2:20 | Modal por URL vs modal por estado |
| 2:20 – 2:30 | Tarea |

---

## Bloque 1 — El dolor (10 min)

La versión ingenua: un `Dialog` de shadcn con estado en la grilla, que hace fetch del producto al abrirse.

Problemas (pedí que los encuentren ellos):

1. La URL no cambia → no se puede compartir "mirá este queso".
2. F5 → el modal desaparece.
3. Botón "atrás" del celular → **sale de la tienda** en vez de cerrar el modal.
4. El contenido del modal se pide desde el cliente → spinner, y no aprovecha nada de la cache del servidor.

> "Instagram, Airbnb, Mercado Libre: tocás una foto, se abre encima, la URL cambia, recargás y ves la página completa. Eso lo hacen dos features de routing que Next trae de fábrica."

---

## Bloque 2 — Parallel routes (25 min)

Un **slot** es una carpeta que empieza con `@`. No es un segmento de URL: es una "ranura" que el layout recibe como prop, además de `children`.

```text
(shop)/
├── layout.tsx         ← recibe { children, modal }
├── page.tsx           ← children en "/"
├── @modal/
│   └── default.tsx    ← qué mostrar en el slot cuando no hay nada que matchee
└── products/...
```

```tsx
// src/app/(frontend)/[locale]/(shop)/layout.tsx
export default function ShopLayout({ children, modal }: { children: React.ReactNode; modal: React.ReactNode }) {
  return (
    <>
      <SiteHeader />
      <main className="mx-auto max-w-6xl px-4 py-8">{children}</main>
      <SiteFooter />
      {modal}
    </>
  )
}
```

> `LayoutProps<'/ruta'>` incluye los slots tipados automáticamente cuando corren `next typegen`. Acá tipamos a mano porque hay **dos** layouts que resuelven `/[locale]` (el root de `[locale]` y el de `(shop)`), y el tipo generado es por ruta, no por archivo. Cuando un layout vive dentro de un route group, el tipo explícito es más claro.

### 2.1 — `default.tsx` es obligatorio (Next 16)

```tsx
// src/app/(frontend)/[locale]/(shop)/@modal/default.tsx
export default function ModalDefault() {
  return null
}
```

> "En Next 16, **todo slot paralelo necesita `default.tsx`** o el build falla. Antes era opcional y causaba 404 misteriosos al recargar."

### 2.2 — Navegación suave vs dura

| | Navegación suave (`<Link>`, `router.push`) | Navegación dura (F5, URL directa) |
|---|---|---|
| Slot sin match para la URL nueva | **Conserva** lo que mostraba | Renderiza `default.tsx` |

Esto es lo que permite que un modal quede abierto mientras se navega "por debajo", y que al recargar desaparezca prolijamente.

---

## Bloque 3 — Intercepting routes (20 min)

Interceptar = "cuando se navega **desde adentro de la app** a la ruta X, en vez de mostrar X, mostrá esta otra cosa (sin cambiar la URL de destino)". En carga directa, no se intercepta: se muestra X normal.

| Convención | Intercepta |
|---|---|
| `(.)carpeta` | un segmento del **mismo** nivel |
| `(..)carpeta` | un nivel **arriba** |
| `(..)(..)carpeta` | dos niveles arriba |
| `(...)carpeta` | desde la **raíz** de `app` |

> "Los niveles se cuentan por **segmentos de ruta**, no por carpetas. Los route groups `(shop)` y los slots `@modal` **no cuentan**. Es el error número uno: cuentan carpetas y no anda."

Para nosotros: el slot `@modal` está en `(shop)`, que corresponde a `/[locale]`. `products` está al mismo nivel → `(.)products`.

---

## Bloque 4 — En vivo: quick view de producto (45 min)

### 4.1 — Estructura

```text
(shop)/
├── layout.tsx
├── @modal/
│   ├── default.tsx                       ← null
│   ├── [...catchAll]/page.tsx            ← null (cierra el modal al navegar a otra cosa)
│   └── (.)products/
│       └── [slug]/
│           └── page.tsx                  ← el modal
└── products/
    ├── page.tsx                          ← grilla
    └── [slug]/
        └── page.tsx                      ← página completa (carga directa / F5)
```

### 4.2 — El contenedor del modal (cliente)

```tsx
// src/components/route-modal.tsx
'use client'

import { useRouter } from '@/i18n/navigation'
import { Dialog, DialogContent, DialogDescription, DialogTitle } from '@/components/ui/dialog'

export function RouteModal({
  title,
  description,
  children,
}: {
  title: string
  description?: string
  children: React.ReactNode
}) {
  const router = useRouter()
  return (
    <Dialog defaultOpen onOpenChange={(open) => { if (!open) router.back() }}>
      <DialogContent className="max-h-[90dvh] max-w-3xl overflow-y-auto">
        <DialogTitle>{title}</DialogTitle>
        {description && <DialogDescription>{description}</DialogDescription>}
        {children}
      </DialogContent>
    </Dialog>
  )
}
```

> "Cerrar = `router.back()`. El modal se abrió con una navegación (push), así que cerrar es volver. Eso también hace que el botón atrás del celular cierre el modal: es el comportamiento que los usuarios esperan."

### 4.3 — La página interceptada (servidor)

```tsx
// src/app/(frontend)/[locale]/(shop)/@modal/(.)products/[slug]/page.tsx
import { Suspense } from 'react'
import { notFound } from 'next/navigation'
import { setRequestLocale } from 'next-intl/server'
import { getProductBySlug } from '@/server/dal/catalog'
import { RouteModal } from '@/components/route-modal'
import { ProductQuickView } from '@/features/catalog/components/product-quick-view'
import { StockLive, StockSkeleton } from '@/features/catalog/components/stock-live'

export default async function ProductModal(props: PageProps<'/[locale]/products/[slug]'>) {
  const { locale, slug } = await props.params
  setRequestLocale(locale)

  const product = await getProductBySlug(slug) // cacheado: misma entrada que la página completa
  if (!product) notFound()

  return (
    <RouteModal title={product.name} description={product.producerName}>
      <ProductQuickView
        product={product}
        stockSlot={
          <Suspense fallback={<StockSkeleton />}>
            <StockLive slug={slug} />
          </Suspense>
        }
      />
    </RouteModal>
  )
}
```

Puntos para remarcar:

- `RouteModal` es cliente, pero `ProductQuickView` (servidor) entra como `children`: patrón de composición de la clase 3.
- `getProductBySlug` está cacheado (clase 10): el modal **comparte la entrada de cache** con la página completa. Abrir el modal no hace query de producto.
- El stock sigue siendo un hueco dinámico, también dentro del modal.
- `ProductQuickView` es una versión compacta de `ProductDetail` con botón "Ver página completa":

```tsx
<a href={`/${locale}/products/${product.slug}`} className="text-sm underline">Ver página completa</a>
```

> "¿Por qué `<a>` y no `<Link>`? Un `<Link>` a `/products/x` desde adentro de la app… se intercepta otra vez y abre el modal. Para forzar la página completa usamos navegación dura. Es de los poquísimos `<a>` internos que van a escribir."

### 4.4 — Cerrar el modal al navegar a otro lado

Si dentro del modal hay un link a la categoría (`/categories/quesos`), la navegación es suave → el slot `@modal` **conserva** el modal abierto encima de la categoría. Para evitarlo:

```tsx
// src/app/(frontend)/[locale]/(shop)/@modal/[...catchAll]/page.tsx
export default function CatchAll() {
  return null
}
```

Cualquier otra URL matchea este catch-all del slot → el slot se vacía → el modal se cierra.

### 4.5 — Validación (hacela con ellos)

1. Grilla `/es/products` → click en un producto → **se abre el modal**, la URL es `/es/products/queso-azul`.
2. Botón atrás → se cierra, la grilla está donde estaba (scroll incluido).
3. Con el modal abierto, F5 → **página completa** del producto.
4. Copiá la URL en otra pestaña → página completa.
5. Click en "Ver página completa" → página completa.
6. Desde el modal, link a la categoría → navega y el modal se cierra.
7. `npm run build` → no hay errores de `default.tsx` faltante.

### 4.6 — Con prefetch

Los links de la grilla usan el prefetch por defecto (App Shell). Para el modal, eso significa que el **contenedor** del modal aparece al instante y el contenido cacheado llega enseguida. Si quieren que el modal abra completo en el acto para los destacados de la home, `<Link prefetch>` (clase 12).

---

## Bloque 5 — Slots paralelos en el panel (25 min)

El resumen del panel del productor tiene tres secciones independientes: ventas del mes (query pesada), pedidos pendientes (rápida), productos con poco stock (rápida). Queremos que cada una cargue y falle **por su cuenta**.

```text
dashboard/
├── layout.tsx            ← recibe { children, sales, pending, lowstock }
├── page.tsx              ← título y accesos rápidos
├── @sales/
│   ├── page.tsx
│   ├── loading.tsx
│   ├── error.tsx
│   └── default.tsx
├── @pending/
│   ├── page.tsx
│   ├── loading.tsx
│   └── default.tsx
└── @lowstock/
    ├── page.tsx
    └── default.tsx
```

```tsx
// src/app/(frontend)/[locale]/dashboard/layout.tsx
type OverviewSlots = { children: React.ReactNode; sales: React.ReactNode; pending: React.ReactNode; lowstock: React.ReactNode }

export default function DashboardLayout({ children, sales, pending, lowstock }: OverviewSlots) {
  return (
    <div className="grid gap-6 lg:grid-cols-[220px_1fr]">
      <DashboardSidebar />
      <div className="space-y-6">
        {children}
        <div className="grid gap-6 md:grid-cols-3">
          <section className="md:col-span-2">{sales}</section>
          <section>{pending}</section>
        </div>
        {lowstock}
      </div>
    </div>
  )
}
```

```tsx
// src/app/(frontend)/[locale]/dashboard/@sales/error.tsx
'use client'
export default function SalesError({ reset }: { error: Error; reset: () => void }) {
  return (
    <div className="rounded border p-4 text-sm">
      No pudimos cargar las ventas. <button onClick={reset} className="underline">Reintentar</button>
    </div>
  )
}
```

> "Si la query de ventas explota, el resto del panel sigue andando. Cada slot tiene su propio `loading` y su propio `error`. Es la misma idea que varios `<Suspense>` hermanos de la clase 6, pero declarada en el routing."

**Ojo:** los slots solo existen en `/dashboard`. En `/dashboard/products`, los slots no matchean → en navegación suave conservan lo último; en carga dura muestran `default.tsx`. Para que **no** aparezcan en las subpáginas, el `default.tsx` de cada slot devuelve `null`… y en navegación suave seguirían visibles. Solución: los slots solo en un layout de un route group que envuelva únicamente la página de resumen:

```text
dashboard/
├── layout.tsx                 ← sidebar
├── (overview)/
│   ├── layout.tsx             ← recibe sales, pending, lowstock
│   ├── page.tsx               → /dashboard
│   ├── @sales/ ...
│   └── ...
└── products/...               → /dashboard/products (sin slots)
```

> "Este es el truco del 'route group para opt-in de layouts' de la doc de estructura de proyecto. Cuando entienden route groups + slots, pueden armar cualquier layout."

(Cuando agreguen `ensureStatic` en layouts del panel en el futuro: los slots paralelos que se renderizan juntos tienen que tener el mismo valor o no tenerlo.)

---

## Bloque 6 — Modal por URL vs modal por estado (15 min)

| Usar **modal por URL** (intercepting) cuando... | Usar **modal por estado** (`Dialog` con `useState`) cuando... |
|---|---|
| El contenido tiene sentido como página propia | Es una confirmación ("¿Borrar producto?") |
| Se quiere compartir o recargar | Es un paso de un flujo que no tiene sentido solo |
| "Atrás" debería cerrarlo | No debería aparecer en el historial |
| Ej: quick view, login desde el header, foto ampliada | Ej: confirmar, selector de fecha, formulario de 2 campos |

Ejemplo de modal por estado con una Server Action:

```tsx
'use client'

import { useTransition } from 'react'
import {
  AlertDialog, AlertDialogAction, AlertDialogCancel, AlertDialogContent,
  AlertDialogDescription, AlertDialogFooter, AlertDialogTitle, AlertDialogTrigger,
} from '@/components/ui/alert-dialog'
import { Button } from '@/components/ui/button'
import { archiveProduct } from '../actions'

export function ArchiveProductButton({ productId, name }: { productId: string; name: string }) {
  const [isPending, startTransition] = useTransition()
  return (
    <AlertDialog>
      <AlertDialogTrigger asChild><Button variant="outline" size="sm">Archivar</Button></AlertDialogTrigger>
      <AlertDialogContent>
        <AlertDialogTitle>¿Archivar “{name}”?</AlertDialogTitle>
        <AlertDialogDescription>Deja de verse en la tienda. Podés reactivarlo después.</AlertDialogDescription>
        <AlertDialogFooter>
          <AlertDialogCancel>Cancelar</AlertDialogCancel>
          <AlertDialogAction disabled={isPending} onClick={() => startTransition(() => archiveProduct(productId))}>
            Archivar
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  )
}
```

(`npx shadcn@latest add alert-dialog`.)

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| Build: falta `default` para el slot | Next 16 lo exige | `default.tsx` que devuelve `null` |
| El modal no intercepta | Contaron carpetas, no segmentos | `(shop)` y `@modal` no cuentan |
| El modal queda abierto sobre otra página | Navegación suave conserva el slot | `@modal/[...catchAll]/page.tsx` con `null` |
| "Ver página completa" vuelve a abrir el modal | `<Link>` se intercepta | `<a href>` (navegación dura) |
| Al recargar `/dashboard/products` aparecen los slots del resumen | Slots en el layout equivocado | Route group `(overview)` |
| `DialogContent requires a DialogTitle` (warning de accesibilidad) | Falta el título | `DialogTitle` siempre (puede ser visualmente oculto) |

---

## Tarea

1. Quick view de producto completo, con `FavoriteButton` y `AddToCartButton` adentro, y las 7 validaciones del bloque 4.5 pasando.
2. Login en modal: el link "Ingresar" del header abre `/sign-in` interceptado en un modal; recargando se ve la página de login completa. (Ojo: `(auth)` es otro route group; pensá dónde va el slot y qué convención de intercepción usar.)
3. Resumen del panel con los tres slots en `(overview)`, cada uno con su loading, y `@sales` con error simulado y reintento.
4. `ArchiveProductButton` en la tabla de productos del panel.

**Terminado cuando:** `npm run build` pasa y el test de Playwright "abrir modal → atrás → la grilla mantiene el scroll" está en verde.
