# Clase 20 — Metadata, SEO e imágenes

**Duración:** 2:30 hs
**Branch de salida:** `clase-20-seo`
**Lo que se llevan:** metadata completa y por idioma (títulos, descripciones, canonical, `hreflang`, Open Graph), imágenes OG generadas por código, `sitemap.xml` y `robots.txt` dinámicos, datos estructurados (JSON-LD) de producto, y `next/image` bien configurado con los cambios de Next 16 — todo **sin sacar nada del static shell**.

**Requisito:** tag `clase-19`.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:10 | El dolor: compartir un producto en WhatsApp muestra "Create Next App" |
| 0:10 – 0:40 | Metadata API: estática, `generateMetadata`, herencia y merge |
| 0:40 – 1:00 | Metadata y Cache Components: cuándo bloquea y cómo se resuelve |
| 1:00 – 1:25 | Imágenes Open Graph con `ImageResponse` |
| 1:25 – 1:45 | `sitemap.ts`, `robots.ts` y `noindex` donde corresponde |
| 1:45 – 2:00 | JSON-LD de producto (y cómo no inyectar HTML) |
| 2:00 – 2:25 | `next/image` en Next 16 |
| 2:25 – 2:30 | Tarea |

---

## Bloque 1 — El dolor (10 min)

Pegá `https://comarca.dev/es/products/queso-azul` en WhatsApp/Slack (o en https://www.opengraph.xyz). Sale el título genérico, sin imagen.

> "Para una tienda, el link compartido **es** marketing. Y Google decide cuánto mostrarte en resultados según el título, la descripción y los datos estructurados. Hoy hacemos que cada página se presente bien."

---

## Bloque 2 — Metadata API (30 min)

### 2.1 — Dónde se define

- `export const metadata: Metadata` → estática.
- `export async function generateMetadata(props, parent)` → depende de params o datos.
- Solo en `layout.tsx` y `page.tsx`, solo en **Server Components**. No se pueden exportar las dos en el mismo segmento.
- Archivos especiales (`favicon.ico`, `icon.png`, `opengraph-image.tsx`, `sitemap.ts`, `robots.ts`) tienen prioridad sobre lo que digan los objetos.

### 2.2 — Merge de arriba hacia abajo

Se evalúa del root al page y se mezcla **superficialmente**: si una página define `openGraph`, **reemplaza todo** el `openGraph` del layout (no lo combina campo por campo).

```ts
// src/lib/seo.ts
import type { Metadata } from 'next'
import { routing, type Locale } from '@/i18n/routing'

export const siteUrl = process.env.NEXT_PUBLIC_SITE_URL!

/** canonical + hreflang para una ruta SIN prefijo de idioma, ej: '/products/queso-azul' */
export function localeAlternates(locale: Locale, path: string): Metadata['alternates'] {
  const clean = path === '/' ? '' : path
  return {
    canonical: `/${locale}${clean}`,
    languages: {
      ...Object.fromEntries(routing.locales.map((l) => [l, `/${l}${clean}`])),
      'x-default': `/${routing.defaultLocale}${clean}`,
    },
  }
}

/** Base de Open Graph compartida; las páginas la extienden con spread */
export const ogBase = {
  siteName: 'Comarca',
  type: 'website' as const,
}
```

Layout raíz de `[locale]` (amplía lo de la clase 8):

```tsx
export async function generateMetadata(props: LayoutProps<'/[locale]'>): Promise<Metadata> {
  const { locale } = await props.params
  const t = await getTranslations({ locale, namespace: 'Metadata' })
  return {
    metadataBase: new URL(siteUrl),
    title: { default: t('title'), template: '%s · Comarca' },
    description: t('description'),
    alternates: localeAlternates(locale as Locale, '/'),
    openGraph: { ...ogBase, locale: { es: 'es_AR', en: 'en_US', pt: 'pt_BR' }[locale] },
    twitter: { card: 'summary_large_image' },
  }
}
```

Página de producto:

```tsx
export async function generateMetadata(props: PageProps<'/[locale]/products/[slug]'>): Promise<Metadata> {
  const { locale, slug } = await props.params
  const product = await getProductBySlug(slug) // 'use cache' → resuelto en build
  if (!product) return { title: 'Producto no encontrado', robots: { index: false } }

  const description = product.description.replace(/[#*_>\[\]]/g, '').slice(0, 155)
  return {
    title: product.name,
    description,
    alternates: localeAlternates(locale as Locale, `/products/${slug}`),
    openGraph: {
      ...ogBase,
      title: product.name,
      description,
      url: `/${locale}/products/${slug}`,
      // la imagen la pone opengraph-image.tsx (bloque 4)
    },
  }
}
```

### 2.3 — `viewport`

`themeColor` y `colorScheme` ya no van en `metadata` (deprecado desde 14): van en `viewport`.

```ts
// en el layout de [locale]
export const viewport: Viewport = {
  themeColor: [
    { media: '(prefers-color-scheme: light)', color: '#ffffff' },
    { media: '(prefers-color-scheme: dark)', color: '#0a0a0a' },
  ],
}
```

---

## Bloque 3 — Metadata con Cache Components (20 min)

`generateMetadata` sigue **las mismas reglas que cualquier componente**:

| Caso | Qué pasa |
|---|---|
| Lee datos cacheados (`'use cache'`) o params conocidos | La metadata entra al shell, en el `<head>` del HTML inicial ✅ |
| Lee datos sin cache o runtime APIs, y **el resto de la página también tiene huecos** | La metadata se streamea junto con los huecos (se agrega al final del `<body>`; Googlebot la lee) |
| Lee datos sin cache o runtime APIs, y **el resto de la página es totalmente estático** | **Error**: Next te pide que decidas |

El tercer caso tiene dos salidas (de la doc):

```tsx
// 1. Si depende de datos externos pero no del request: cachearla
export async function generateMetadata() {
  'use cache'
  cacheLife('cms')
  const page = await getPageBySlug('about')
  return { title: page?.title }
}
```

```tsx
// 2. Si de verdad depende del request: marcador dinámico explícito
import { connection } from 'next/server'

async function Connection() { await connection(); return null }

export default function Page() {
  return (
    <>
      <article>Contenido estático</article>
      <Suspense><Connection /></Suspense>   {/* "esta página tiene una parte dinámica a propósito" */}
    </>
  )
}
```

Cuidado con `'use cache'` directo en `generateMetadata`: el retorno tiene que ser serializable. `new URL(...)` **no** lo es → `metadataBase` como string (`url.toString()`), o dejarlo en el layout raíz sin cachear.

> "**Bots limitados** (el scraper de WhatsApp/Facebook, que no ejecuta JS): Next los detecta por user agent y les manda la metadata en el `<head>` aunque sea dinámica, bloqueando el render. Por eso la vista previa funciona igual. La lista se ajusta con `htmlLimitedBots` en `next.config`."

---

## Bloque 4 — Imágenes Open Graph por código (25 min)

Un archivo `opengraph-image.tsx` en el segmento genera la imagen y Next agrega solo los `<meta property="og:image">`.

```tsx
// src/app/(frontend)/[locale]/(shop)/products/[slug]/opengraph-image.tsx
import { ImageResponse } from 'next/og'
import { readFile } from 'node:fs/promises'
import path from 'node:path'
import { getProductBySlug } from '@/server/dal/catalog'

export const size = { width: 1200, height: 630 }
export const contentType = 'image/png'
export const alt = 'Producto de Comarca'

// Fuente leída una vez a nivel de módulo (predecible, clase 9)
const fraunces = readFile(path.join(process.cwd(), 'src/assets/fonts/Fraunces-SemiBold.ttf'))

export default async function OgImage({ params }: { params: Promise<{ locale: string; slug: string }> }) {
  const { slug } = await params            // en Next 16, params es promesa también acá
  const product = await getProductBySlug(slug)
  const price = product
    ? new Intl.NumberFormat('es-AR', { style: 'currency', currency: 'ARS' }).format(product.priceCents / 100)
    : ''

  return new ImageResponse(
    (
      <div style={{ width: '100%', height: '100%', display: 'flex', background: '#f6f3ea', padding: 64 }}>
        <div style={{ display: 'flex', flexDirection: 'column', justifyContent: 'space-between', flex: 1 }}>
          <div style={{ fontSize: 32, color: '#5b6b2f' }}>Comarca</div>
          <div style={{ display: 'flex', flexDirection: 'column' }}>
            <div style={{ fontSize: 72, fontFamily: 'Fraunces', color: '#1c1c1c', lineHeight: 1.1 }}>
              {product?.name ?? 'Comarca'}
            </div>
            <div style={{ fontSize: 36, color: '#444', marginTop: 16 }}>{product?.producerName}</div>
          </div>
          <div style={{ fontSize: 48, fontWeight: 700 }}>{price}</div>
        </div>
      </div>
    ),
    { ...size, fonts: [{ name: 'Fraunces', data: await fraunces, weight: 600 }] },
  )
}
```

- `ImageResponse` usa un subconjunto de CSS (flexbox, nada de grid). Cada `div` con más de un hijo necesita `display: 'flex'`.
- `getProductBySlug` está cacheado: la imagen se genera con datos de la cache y se prerenderiza para los slugs de `generateStaticParams`.
- Para la foto real del producto: `<img src={absoluteUrl} />` con URL absoluta.

**Validación:** `http://localhost:3000/es/products/queso-azul/opengraph-image` devuelve el PNG; el HTML de la página tiene `og:image` con esa URL (absoluta gracias a `metadataBase`).

> "Una imagen por producto, generada en build, sin diseñador. Cuando cambia el precio, el tag `product:queso-azul` invalida también la imagen."

---

## Bloque 5 — Sitemap y robots (20 min)

### 5.1 — `sitemap.ts`

```ts
// src/app/sitemap.ts
import type { MetadataRoute } from 'next'
import { routing } from '@/i18n/routing'
import { listProductSlugsWithDates, listCategorySlugs } from '@/server/dal/catalog'
import { listPostSlugsWithDates } from '@/server/dal/cms'
import { siteUrl } from '@/lib/seo'

function entry(path: string, lastModified?: Date): MetadataRoute.Sitemap[number] {
  return {
    url: `${siteUrl}/${routing.defaultLocale}${path}`,
    lastModified,
    alternates: {
      languages: Object.fromEntries(routing.locales.map((l) => [l, `${siteUrl}/${l}${path}`])),
    },
  }
}

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const [products, categories, posts] = await Promise.all([
    listProductSlugsWithDates(),
    listCategorySlugs(),
    listPostSlugsWithDates(),
  ])
  return [
    entry(''),
    entry('/products'),
    entry('/blog'),
    ...categories.map((c) => entry(`/categories/${c.slug}`)),
    ...products.map((p) => entry(`/products/${p.slug}`, p.updatedAt)),
    ...posts.map((p) => entry(`/blog/${p.slug}`, p.updatedAt)),
  ]
}
```

Las funciones de la DAL que usa tienen `'use cache'` + `cacheTag(tags.catalog)` / `tags.cms('posts')`: el sitemap se regenera cuando cambia el catálogo o el blog.

> Con más de 50.000 URLs (límite del protocolo), `generateSitemaps()` parte en varios archivos. En Next 16, el `id` que recibe la función `sitemap({ id })` es una **promesa**: `const n = Number(await id)`.

### 5.2 — `robots.ts`

```ts
// src/app/robots.ts
import type { MetadataRoute } from 'next'
import { siteUrl } from '@/lib/seo'

export default function robots(): MetadataRoute.Robots {
  const isProd = process.env.VERCEL_ENV === 'production' || process.env.APP_ENV === 'production'
  return {
    rules: isProd
      ? [{ userAgent: '*', allow: '/', disallow: ['/admin', '/api/', '/*/account', '/*/dashboard', '/*/checkout'] }]
      : [{ userAgent: '*', disallow: '/' }], // staging: que Google no lo indexe nunca
    sitemap: `${siteUrl}/sitemap.xml`,
  }
}
```

> "`APP_ENV` y no `NODE_ENV`: en staging también corremos con `NODE_ENV=production`. Si usan `NODE_ENV`, el staging termina indexado en Google con contenido de prueba. Pasa más de lo que creen."

### 5.3 — `noindex` en lo privado

```ts
// src/app/(frontend)/[locale]/account/layout.tsx (y dashboard, checkout, sign-in)
export const metadata: Metadata = { robots: { index: false, follow: false } }
```

---

## Bloque 6 — JSON-LD de producto (15 min)

Google entiende datos estructurados `Product` con precio y disponibilidad (estrellitas, precio en el resultado).

```tsx
// src/features/catalog/components/product-json-ld.tsx
import type { ProductDetailDTO } from '@/server/dal/catalog'
import { siteUrl } from '@/lib/seo'

export function ProductJsonLd({ product, locale }: { product: ProductDetailDTO; locale: string }) {
  const data = {
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: product.name,
    image: [`${siteUrl}${product.imageUrl}`],
    description: product.description.slice(0, 500),
    brand: { '@type': 'Brand', name: product.producerName },
    offers: {
      '@type': 'Offer',
      url: `${siteUrl}/${locale}/products/${product.slug}`,
      priceCurrency: 'ARS',
      price: (product.priceCents / 100).toFixed(2),
      availability: product.stock > 0 ? 'https://schema.org/InStock' : 'https://schema.org/OutOfStock',
    },
  }

  return (
    <script
      type="application/ld+json"
      // Escapamos '<' para que un "</script>" en la descripción no cierre la etiqueta (XSS)
      dangerouslySetInnerHTML={{ __html: JSON.stringify(data).replace(/</g, '\\u003c') }}
    />
  )
}
```

> "`JSON.stringify` no escapa `</script>`. Si un productor escribe eso en la descripción, sin el `replace` rompe la etiqueta y puede inyectar HTML. Es el único `dangerouslySetInnerHTML` 'legítimo' del proyecto, y aun así lleva escape."

La disponibilidad sale del dato **cacheado** (puede tener minutos de atraso): para Google es suficiente; el stock real sigue en el hueco dinámico.

**Validación:** https://search.google.com/test/rich-results con la URL desplegada (o pegando el HTML).

---

## Bloque 7 — `next/image` en Next 16 (25 min)

### 7.1 — Qué hace

Redimensiona, convierte a WebP/AVIF, genera `srcset`, lazy-load por defecto, reserva espacio (sin saltos de layout). La optimización corre en el servidor (`/_next/image`) y se cachea.

### 7.2 — Cambios de la 16 que pegan

| Cambio | Consecuencia |
|---|---|
| `priority` **deprecado** → `preload` | Para la imagen LCP, la doc recomienda en general `loading="eager"` o `fetchPriority="high"`; `preload` solo si de verdad hace falta un `<link rel=preload>` |
| `qualities` por defecto = `[75]` | `quality={90}` se redondea a 75. Si necesitan otra, agregarla a `images.qualities` |
| `minimumCacheTTL` por defecto = 4 h | Imágenes que cambian en la misma URL tardan en refrescarse: versionen la URL |
| `imageSizes` sin `16` | Irrelevante salvo íconos minúsculos |
| Imágenes locales con query string (`/a.jpg?v=2`) | Requieren `images.localPatterns` con `search` |
| `dangerouslyAllowLocalIP: false` por defecto | No optimiza imágenes de IPs privadas (anti-SSRF) |
| `maximumRedirects: 3` | Antes ilimitado |
| `images.domains` deprecado | `remotePatterns` |
| `next/legacy/image` deprecado | `next/image` |

### 7.3 — Configuración de Comarca

```ts
// next.config.ts
images: {
  formats: ['image/avif', 'image/webp'],
  qualities: [60, 75, 85],
  remotePatterns: [
    { protocol: 'https', hostname: 'media.comarca.com', pathname: '/**' }, // si migran media a S3/CDN
  ],
},
```

### 7.4 — Uso correcto

```tsx
// Hero (LCP): ancho completo, carga inmediata
<Image
  src={hero.url}
  alt={hero.alt}
  fill
  sizes="100vw"
  loading="eager"
  fetchPriority="high"
  className="object-cover"
/>

// Card de la grilla: le decimos al navegador qué ancho va a ocupar
<Image
  src={p.imageUrl}
  alt={p.name}
  width={600}
  height={450}
  sizes="(min-width: 1024px) 25vw, (min-width: 768px) 33vw, 50vw"
  quality={60}
/>
```

> "`sizes` es el prop más ignorado y el que más ancho de banda ahorra. Sin `sizes`, el navegador asume 100vw y en desktop baja una imagen de 1920 px para una card de 300. Con `sizes`, baja la de 384."

**Validación:** Lighthouse en `/es` y `/es/products/queso-azul` (producción local): LCP < 2,5 s con throttling "Slow 4G", CLS ≈ 0, ninguna advertencia de "Properly size images".

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| `metadataBase property in metadata export is not set` | URLs relativas en OG sin base | `metadataBase` en el layout raíz |
| El `openGraph` del layout "desaparece" en una página | Merge superficial | Spread de `ogBase` |
| Error de metadata en página estática | `generateMetadata` sin cache | `'use cache'` o marcador dinámico |
| `Only plain objects…` en metadata cacheada | `new URL()` en el retorno | String |
| Staging indexado | `robots` basado en `NODE_ENV` | `APP_ENV` |
| `quality={90}` se ve igual que 75 | `qualities` por defecto `[75]` | Configurar `images.qualities` |

---

## Tarea

1. Metadata completa (title, description, canonical, hreflang, OG) en: home, listado, categoría, producto, productor, blog y receta.
2. `opengraph-image.tsx` para producto y receta, en los tres idiomas (el texto del precio formateado según el locale).
3. `sitemap.ts` con alternates por idioma y `robots.ts` con la lógica de `APP_ENV`.
4. JSON-LD `Product` y `Recipe` (schema.org) en sus páginas, validado con la herramienta de Google.
5. Auditoría de imágenes: todas con `sizes`; la LCP de cada página con `loading="eager"` + `fetchPriority="high"`. Lighthouse ≥ 95 en Performance y SEO en home y producto.

**Terminado cuando:** pegar un link de producto en WhatsApp muestra la imagen generada con nombre y precio, y Lighthouse da los puntajes pedidos.
