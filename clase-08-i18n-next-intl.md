# Clase 8 — Internacionalización con next-intl

**Duración:** 2:30 hs
**Formato:** clase práctica sobre `comarca-web` (branch `feat/clases-06-08`). Comarca ya está en `es`, `en` y `pt`. Se recorre cada pieza, se prueba en el navegador y con `curl.exe`, y se rompe a propósito.
**Lo que se llevan:** URLs por idioma, detección en el proxy, traducciones en Server y Client Components, moneda y fechas por locale, `hreflang`, contenido de la base traducido, y **todo sin perder el prerender** (`npm run build` sin `ƒ`).

---

## Antes de entrar al aula

```powershell
Copy-Item .env.example .env   # solo la primera vez (si no tenés .env)
npm run infra:up
npm run db:reset     # incluye traducciones en/pt de 5 productos
npm run dev
```

---

## Guion

| Tiempo | Demo |
|---|---|
| 0:00 – 0:10 | El dolor y las decisiones |
| 0:10 – 0:35 | Demo 1 — Los 5 archivos de next-intl y la carpeta `[locale]` |
| 0:35 – 0:55 | Demo 2 — El proxy detecta el idioma |
| 0:55 – 1:25 | Demo 3 — Traducir: server, client, plurales, rich text, formatos |
| 1:25 – 1:50 | Demo 4 — No perder el prerender (y el bug que lo rompe) |
| 1:50 – 2:20 | Demo 5 — Metadata, selector y contenido de la base |
| 2:20 – 2:30 | Tarea |

---

## El dolor y las decisiones (10 min)

> "Cada texto en duro es una promesa de que el día que pidan inglés, alguien pasa una semana con grep. Y `es-AR` metido en `formatPrice`: un brasileño ve el formato argentino, o peor, ve ARS sin saber que son pesos."

| Decisión | Elegimos | Por qué |
|---|---|---|
| Dónde vive el idioma | **URL** (`/en/...`) | Indexable, compartible, cacheable por URL |
| Prefijo | **`always`** | Cada idioma es una ruta distinta con su propio static shell. `as-needed` agrega rewrites invisibles que complican A/B y caches |
| Locales / default | `es`, `en`, `pt` / `es` | Mercado regional |
| Moneda | Siempre **ARS**; cambia el **formato** | El idioma no cambia la moneda de cobro |
| Slugs | Iguales en todos los idiomas | `pathnames` de next-intl queda como tarea opcional |

---

## Demo 1 — Los 5 archivos y la carpeta `[locale]` (25 min)

**Abrí en este orden** (son cortos):

| Archivo | Qué hace |
|---|---|
| `src/i18n/routing.ts` | `defineRouting({ locales: ['es','en','pt'], defaultLocale: 'es', localePrefix: 'always' })` |
| `src/i18n/navigation.ts` | `Link`, `redirect`, `useRouter`, `usePathname` **con el idioma incluido** |
| `src/i18n/request.ts` | Por request: qué locale, qué `messages/{locale}.json`, `timeZone` |
| `next.config.ts` | `createNextIntlPlugin()` envuelve la config |
| `src/i18n/global.d.ts` | Tipa locales y mensajes |
| `messages/es.json`, `en.json`, `pt.json` | Los textos |

**Rompelo (tipado):** en `src/features/catalog/components/product-card.tsx` cambiá `t('outOfStock')` por `t('outOfStok')` y corré `npx tsc --noEmit` → error de TypeScript. Volvelo.

**La estructura** (mostrala en el explorador):

```text
src/app/
├── (frontend)/[locale]/
│   ├── layout.tsx          ← ES EL ROOT LAYOUT (<html lang={locale}>)
│   ├── not-found.tsx
│   ├── [...rest]/page.tsx  ← /es/cualquier-cosa → notFound() con header traducido
│   ├── (shop)/ page, home-b, products, categories, producers, cart, about, lab
│   ├── (auth)/sign-in
│   └── dashboard/
├── global-error.tsx
└── globals.css
```

Ya no existe `src/app/layout.tsx`: es el caso de "múltiples root layouts" de la clase 2.

> "Los route groups `(frontend)` y `(shop)` no aparecen en la URL; `[locale]` sí. Por eso los tipos ahora son `PageProps<'/[locale]/products/[slug]'>`. Si mueven una página, `npx next typegen && npx tsc --noEmit` marca todas las que quedaron mal."

**Probalo:** `http://localhost:3000/es/cualquier-cosa` → 404 con el header y "No encontramos esa página". `/en/cualquier-cosa` → "We could not find that page".

**ESLint:** desde hoy `Link` sale de `@/i18n/navigation`. Creá un archivo con `import Link from 'next/link'` y corré `npx eslint <archivo>`:

```text
'default' import from 'next/link' is restricted. Usá Link de @/i18n/navigation
```

(`notFound`, `useSearchParams` y `useParams` siguen viniendo de `next/navigation`.)

---

## Demo 2 — El proxy detecta el idioma (20 min)

**Abrí** `src/proxy.ts`: `handleI18n = createIntlMiddleware(routing)` corre **al final**, después de los pasos de la clase 7. `splitLocale()` separa `/en/dashboard` en `{ locale: 'en', path: '/dashboard' }`.

```powershell
curl.exe -I http://localhost:3000/                                         # 307 → /es
curl.exe -I -H "Accept-Language: en-US,en;q=0.9" http://localhost:3000/    # 307 → /en
curl.exe -I -b "NEXT_LOCALE=pt" http://localhost:3000/products             # 307 → /pt/products
curl.exe -I http://localhost:3000/en/dashboard                             # 307 → /en/sign-in?next=%2Fen%2Fdashboard
```

Todos verificados en clase. Orden de prioridad de next-intl: **cookie `NEXT_LOCALE` → `Accept-Language` → default**.

> "El orden importa: el redirect de auth necesita el idioma para mandarte al login **en tu idioma**. Si la URL todavía no tiene idioma (`/dashboard`), next-intl primero redirige a `/es/dashboard` y en ese segundo request actúa el chequeo de auth."

Los tests de la clase 7 ya están actualizados con idiomas: `npm test`.

---

## Demo 3 — Traducir (30 min)

### Server Components

- **async** → `getTranslations`: `features/catalog/components/related-products.tsx`.
- **no async** → `useTranslations` también funciona en el servidor: `components/layout/site-header.tsx`.

> "`useTranslations` en un Server Component **no** es un hook de cliente: se resuelve en el servidor. El header es Server Component; solo `LocaleSwitcher`, `ThemeToggle` y `MobileNav` son islas cliente, y reciben los textos ya traducidos por props (`label`)."

### Client Components

`features/catalog/components/add-to-cart-button.tsx` (`'use client'`) usa `useTranslations('Product')`. Los mensajes le llegan por `NextIntlClientProvider` en el layout.

**Mirá el costo:** en `http://localhost:3000/en/lab/formatos` → "Ver código fuente" → buscá `"Dashboard":{` → **todos** los mensajes viajan al cliente. Con un JSON grande, se limita:

```tsx
<NextIntlClientProvider messages={pick(await getMessages(), ['Product', 'Cart'])}>
```

### Plurales, variables, rich text

**Corré** `http://localhost:3000/es/lab/formatos`, después `/en/` y `/pt/` (o el selector del header). Salida real:

| | es | en | pt |
|---|---|---|---|
| `Price cents={1250000}` | `12.500,00 ARS` | `ARS 12,500.00` | `ARS 12.500,00` |
| `stockLeft` 0 / 1 / 12 | Sin stock · Queda 1 unidad · Quedan 12 unidades | Out of stock · 1 unit left · 12 units left | Sem estoque · Resta 1 unidade · Restam 12 unidades |
| `dateTime(…, { dateStyle: 'long' })` | 14 de octubre de 2026 | October 14, 2026 | 14 de outubro de 2026 |
| `relativeTime` (hace 3 días) | hace 3 días | 3 days ago | há 3 dias |

El mensaje ICU que hace eso (`messages/es.json`):

```json
"stockLeft": "{count, plural, =0 {Sin stock} one {Queda # unidad} other {Quedan # unidades}}"
```

Rich text: `/es/cart` → "Al comprar aceptás los **términos y condiciones**" (es un `<Link>`):

```tsx
t.rich('terms', { link: (chunks) => <Link href="/about">{chunks}</Link> })
```

### Formatos: `Price` reemplaza a `formatPrice`

`src/components/price.tsx` usa `useFormatter()`. Ya no existe `lib/money.ts` con `es-AR` hardcodeado.

> "`relativeTime` depende de **ahora**. Abrí `lab/formatos/page.tsx`: está dentro de un `<Suspense>` con `await connection()`. Sin eso, Next no deja prerenderizar algo que cambia cada segundo."

---

## Demo 4 — No perder el prerender (25 min)

```powershell
npm run build
```

**Mirá la salida:** cada ruta aparece por triplicado y **ninguna con `ƒ`**:

```text
├   /[locale]/products/[slug]
│ ├ ◐ /es/products/queso-azul
│ └ ◐ [+28 more paths]
├   /[locale]/cart
│ ├ ○ /es/cart
│ ├ ○ /en/cart
│ └ ○ /pt/cart
ƒ Proxy (Middleware)
○ (Static)  ◐ (Partial Prerender)
```

### `setRequestLocale` — qué hace

Sin él, next-intl averigua el idioma leyendo un header → **dato de request** → la página deja de ser estática. Con `setRequestLocale(locale)` se lo decimos nosotros. Regla: **en cada layout y page que traduce, antes de traducir.**

### Root params: el idioma sin `await params`

En las páginas de Comarca vas a ver esto:

```ts
import { locale } from 'next/root-params'

export default async function ProductPage({ params }: PageProps<'/[locale]/products/[slug]'>) {
  setRequestLocale((await locale()) as Locale)
  ...
}
```

> "`[locale]` está **antes** del root layout, entonces es un *root param*: se lee con `locale()` desde cualquier Server Component, sin pasar props. ¿Por qué no `(await params).locale`? Porque `params` incluye el `slug`, y en Next 16.4 hacer `await params` en el cuerpo de una página con `[slug]` (fuera de un `<Suspense>`) bloquea el static shell. Lo descubrimos rompiendo el build: el error dice `Route "/[locale]/categories/[slug]": Next.js encountered URL data during prerendering`."

`root-params` **no** anda en Client Components, Server Actions ni Route Handlers (todavía).

### El bug que rompe el build — reproducilo

`src/app/(frontend)/[locale]/(shop)/products/[slug]/not-found.tsx` llama a `setRequestLocale` con root params. **Sacá esas dos líneas** y dejalo como un componente común con `useTranslations`. Corré `npm run build`:

```text
Error: Route "/[locale]/products/[slug]": Next.js encountered uncached or runtime data during prerendering.
    at LocaleLayout (src\app\(frontend)\[locale]\layout.tsx:41:9)
> 41 |         <NextIntlClientProvider>
```

> "El error apunta al layout, pero el culpable es el `not-found.tsx`: no recibe `params`, nadie le dijo el idioma, next-intl lo busca en los headers... y eso es dato de request. Lección: cuando el stack apunta a un provider, sospechá de un hijo que traduce sin `setRequestLocale`. Para encontrarlo: `npx next build --debug-prerender`."

Volvelo a dejar como estaba.

### Params combinados

`products/[slug]/page.tsx` → `generateStaticParams` devuelve solo `{ slug }`; Next lo combina con cada locale del layout: `/es/products/queso-azul`, `/en/...`, `/pt/...`.

---

## Demo 5 — Metadata, selector y contenido de la base (30 min)

### `hreflang`

```powershell
curl.exe -s http://localhost:3000/en/products/queso-azul | Select-String -Pattern '<html lang="[a-z]+"|hrefLang="[^"]+"' -AllMatches | % { $_.Matches.Value }
```

Salida real:

```text
<html lang="en"
hrefLang="es"
hrefLang="en"
hrefLang="pt"
hrefLang="x-default"
```

Y el `<title>` es `Tandil blue cheese · Comarca`. La metadata vive en el layout (`alternates.languages` del sitio) y en `generateMetadata` de la página de producto (las URLs con slug). En la clase 20 lo generalizamos.

> `metadataBase` sale de `NEXT_PUBLIC_SITE_URL` (si no está, `http://localhost:3000`). En producción hay que definirla.

### Selector de idioma

`src/components/layout/locale-switcher.tsx`. **Probalo:** en `/es/products/queso-azul` elegí English → vas a `/en/products/queso-azul` (mantiene la ruta). DevTools → Application → Cookies → buscá `NEXT_LOCALE`. Después abrí `http://localhost:3000/`.

> ⚠ Probalo vos antes de la clase: el selector y la cookie no se verificaron en navegador al preparar el material. Lo que sí está verificado con curl: con la cookie `NEXT_LOCALE=pt`, `/products` redirige a `/pt/products`.

### Contenido de la base en varios idiomas

Los textos de la interfaz están en JSON. ¿Y el nombre de cada producto?

| Opción | Cuándo |
|---|---|
| Tabla de traducciones en Prisma (lo que usa Comarca) | Contenido que cargan los productores |
| Localización de Payload (clase 19) | Contenido editorial |

**Abrí** `prisma/schema.prisma` → `model ProductTranslation` (`@@id([productId, locale])`), y `prisma/seed.ts` → `translations` (5 productos con en/pt).

**Abrí** `src/server/dal/catalog.ts`:

```ts
const lang = await rootLocale()                       // next/root-params, dentro de la DAL
translations: { where: { locale: lang }, select: { name: true } }
...
name: row.translations[0]?.name ?? row.name,          // fallback al español
```

**Probalo:**

- `/en/products/queso-azul` → "Tandil blue cheese" (tiene traducción)
- `/en/products/reggianito` → "Reggianito estacionado" (no tiene → fallback)

> "El fallback al español es una decisión de **producto**, no técnica. Otra opción válida: no mostrar el producto en ese idioma. Lo importante es que la decisión está en **un** lugar: la DAL."

---

## Errores comunes (los vimos casi todos)

| Síntoma | Causa | Solución |
|---|---|---|
| Rutas que pasan a `ƒ` / build falla con "runtime data" | Falta `setRequestLocale` en algún layout, page **o not-found** | Agregarlo con `locale()` de `next/root-params` |
| "encountered URL data during prerendering" | `await params` en el cuerpo de una página con `[slug]` | Leer el idioma con `locale()`; los params adentro del `<Suspense>` |
| Links que pierden el idioma | `next/link` en vez de `@/i18n/navigation` | Regla de ESLint |
| `/es/xyz` muestra 404 sin estilos | Falta el `[...rest]` | Agregarlo |
| Una clave falta en `pt.json` y nadie se entera | TS solo valida contra `es.json` | `messages/messages.test.ts` (`npm test`) |
| `root-params` da error en una Server Action | No soportado ahí | Pasar el locale como argumento |
| `setRequestLocale(params.locale)` no compila | `params.locale` es `string`, no `Locale` | `as Locale` (el layout ya lo validó con `hasLocale`) |

---

## Tarea

1. Traducí los textos que quedan en duro (`features/about`, la página `/about`, las etiquetas del `/lab`) a los tres idiomas. `npm test` tiene que seguir pasando.
2. Agregá una clave a `es.json` sin agregarla en `en.json` y mostrá en el PR la salida de `npm test` que lo detecta.
3. Cargá traducciones de los 4 productos restantes en el seed y mostrá con `curl.exe` que ya no hay fallback en `/pt/products`.
4. Limitá los mensajes que viajan al cliente a los namespaces que usan los Client Components y mostrá en el PR cuántos bytes bajó el HTML de `/es/products`.
5. Opcional: URLs traducidas con `pathnames` (`/es/productos`, `/en/products`, `/pt/produtos`).

**Terminado cuando:** `npm run build` genera `/es`, `/en` y `/pt` de cada página del catálogo sin `ƒ`, y el HTML de `/en/products/queso-azul` tiene `<html lang="en">` y los `hrefLang`.
