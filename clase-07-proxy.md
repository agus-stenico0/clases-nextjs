# Clase 7 — Proxy (lo que antes era middleware)

**Duración:** 2:30 hs
**Formato:** clase práctica sobre `comarca-web` (branch `feat/clases-06-08`). Se lee `src/proxy.ts` paso por paso y cada paso se prueba con `curl.exe`, con el navegador o con un test.
**Lo que se llevan:** qué hace el proxy y dónde corre, las cinco salidas posibles, un matcher testeado, y la convicción de que **el proxy no es la capa de autorización**.

> El proxy de Comarca ya incluye el handler de next-intl (`handleI18n`, clase 8). Hoy ignoralo: solo hace que `/` redirija a `/es`. Por eso todas las URLs de hoy empiezan con `/es`.

---

## Antes de entrar al aula

```powershell
Copy-Item .env.example .env   # solo la primera vez (si no tenés .env)
npm run infra:up
npm run dev
```

Todo con **`curl.exe`**, no `curl`: en Windows PowerShell 5.1 `curl` es un alias de `Invoke-WebRequest` y no entiende `-I` ni `-b`.

---

## Guion

| Tiempo | Demo |
|---|---|
| 0:00 – 0:15 | El dolor: decidir **qué** renderizar **antes** de renderizar |
| 0:15 – 0:30 | Dónde está el proxy en el pipeline y por qué cambió de nombre |
| 0:30 – 1:30 | Demos 1–4: los cuatro pasos del proxy de Comarca |
| 1:30 – 1:50 | Demo 5 — El matcher |
| 1:50 – 2:10 | Demo 6 — Por qué NO es tu capa de auth |
| 2:10 – 2:25 | Demo 7 — Tests |
| 2:25 – 2:30 | Tarea |

---

## El dolor (15 min)

Casos reales de Comarca que **no** se resuelven bien adentro de una página:

1. Marketing quiere probar dos heroes en la home (A/B) **sin** que la home deje de ser estática.
2. Necesitamos un `x-request-id` en cada request para correlacionar logs (clase 21).
3. Deploy con migración → modo mantenimiento con un switch.
4. Un usuario sin sesión entra a `/dashboard` → al login **sin** renderizar nada del panel.
5. (Clase 8) Detectar el idioma y redirigir a `/en` o `/pt`.

> "Todas deciden **qué** se va a renderizar, antes de renderizar. Para eso está el proxy."

## Dónde corre y por qué se llama así (15 min)

```text
1. headers de next.config
2. redirects de next.config
3. ▶ proxy.ts ◀
4. rewrites beforeFiles
5. rutas del filesystem (public/, _next/static, app/)
6. rewrites afterFiles / rutas dinámicas / fallback
```

| `middleware.ts` (≤15) | `proxy.ts` (16) |
|---|---|
| `export function middleware()` | `export function proxy()` |
| Runtime Edge por defecto | **Runtime Node.js**, no configurable |
| `skipMiddlewareUrlNormalize` | `skipProxyUrlNormalize` |

Migrar un proyecto viejo: `npx @next/codemod@canary middleware-to-proxy .`

> "'Middleware' hacía pensar en Express y la gente metía **todo** ahí. 'Proxy' describe lo que es: una frontera de red delante de tu app. Next lo considera último recurso: si sale con `redirects`/`rewrites` de `next.config`, mejor. Y ojo: corre en **cada** request que matchee, incluidos los prefetch. Cada milisegundo acá se suma a toda la navegación."

**Mostralo en la terminal de `npm run dev`:** cada línea `GET` trae `proxy.ts: 5ms`. Ese es su costo.

---

## La API en 30 segundos — las cinco salidas

Abrí `src/proxy.ts`. Todo lo que hace termina en una de estas:

```ts
NextResponse.next({ request: { headers } })      // dejar pasar, agregando headers AL REQUEST
NextResponse.rewrite(new URL('/es/home-b', url)) // el navegador ve /es, se renderiza /es/home-b
NextResponse.redirect(new URL('/es/sign-in', url)) // el navegador cambia de URL (307)
new NextResponse(html, { status: 503 })          // responder directo
// o no devolver nada → sigue
```

> "La trampa más común: `NextResponse.next({ headers })` (sin `request:`) pone los headers en la **respuesta**: se los mandás al navegador. Para pasarle info a la app es `next({ request: { headers } })`."

---

## Demo 1 — Request id (paso 1: `withRequestId`)

**Abrí** `withRequestId` en `src/proxy.ts` y `src/app/(frontend)/[locale]/(shop)/lab/request-id/page.tsx`.

**Corré:** `http://localhost:3000/es/lab/request-id` y recargá varias veces.

**Mirá:** un UUID distinto en cada recarga (también en la terminal: `[lab/request-id] x-request-id=...`).

**Mirá también** DevTools → Network → headers de la respuesta: **no está**. Viaja en el request hacia la app, no al navegador.

> "`headers()` es una runtime API: es dato del request. Por eso la página lo lee dentro de un `<Suspense>`. Si lo leyera afuera, el build falla (clase 9)."

---

## Demo 2 — Modo mantenimiento (paso 2: `maintenance`)

```powershell
# cortá npm run dev y levantá:
npm run dev:maintenance        # = MAINTENANCE_MODE=1 next dev
curl.exe -I http://localhost:3000/es/products
```

```text
HTTP/1.1 503 Service Unavailable
content-type: text/html; charset=utf-8
retry-after: 600
```

En el navegador: "Estamos actualizando Comarca".

> "¿Por qué no un rewrite a una página de Next? Porque devolvería **200**, y Google entiende 'esta página cambió'. 503 + `Retry-After` dice 'volvé en 10 minutos'."

Volvé a `npm run dev`.

---

## Demo 3 — A/B de la home por rewrite (paso 4: `abHome`)

**Abrí** `abHome` en `src/proxy.ts` y `src/features/home/components/home-view.tsx` (las dos variantes, `data-variant="a"|"b"`).

```powershell
curl.exe -s -b "ab-home=b" http://localhost:3000/es | Select-String -Pattern 'data-variant="."' -AllMatches | % { $_.Matches[0].Value }
# data-variant="b"
curl.exe -s -b "ab-home=a" http://localhost:3000/es | Select-String -Pattern 'data-variant="."' -AllMatches | % { $_.Matches[0].Value }
# data-variant="a"
```

Sin cookie, seis veces seguidas (medido en clase):

```text
Set-Cookie: ab-home=b  → data-variant="b"
Set-Cookie: ab-home=b  → data-variant="b"
Set-Cookie: ab-home=a  → data-variant="a"
...
```

**En el navegador:** abrí `/es` en incógnito varias veces → a veces "Lo mejor de cada comarca", a veces "Comprale directo a quien lo hace". **La URL nunca cambia.** DevTools → Application → Cookies → `ab-home` es `HttpOnly`.

> "¿Por qué rewrite y no leer la cookie en la página? Si la home leyera `cookies()`, dejaría de estar en el static shell (clase 9). Con rewrite, **las dos variantes son páginas prerenderizadas** (mirá `npm run build`: `/es/home-b` aparece como ruta); el proxy solo elige cuál servir."

---

## Demo 4 — Redirección optimista (paso 3: `optimisticAuth`)

**Abrí** `optimisticAuth` en `src/proxy.ts`. Leé en voz alta el comentario: *solo mira si EXISTE la cookie*.

```powershell
curl.exe -I http://localhost:3000/en/dashboard
# HTTP/1.1 307 Temporary Redirect
# location: /en/sign-in?next=%2Fen%2Fdashboard
```

En incógnito: `http://localhost:3000/es/dashboard` → caés en `/es/sign-in`, que muestra "Después de ingresar volverías a: /es/dashboard" (stub hasta la clase 17).

**Ahora rompelo — esta es la demo más importante de la clase:**

```powershell
curl.exe -s -o NUL -w "%{http_code}" -b "better-auth.session_token=lo-que-sea" http://localhost:3000/es/dashboard
# 200
```

> "Con una cookie inventada entré al panel. El proxy no validó nada: no puede, no debe ir a la base en cada request. Por eso **el proxy redirige y la DAL autoriza**."

---

## Demo 5 — El matcher (20 min)

```ts
export const config = {
  matcher: ['/((?!api|_next|_vercel|admin|cms-api|.*\\..*).*)'],
}
```

**Sin matcher, el proxy corre en TODO**: `_next/static/*.js`, imágenes, `favicon.ico`. Una redirección de auth sin matcher bloquea el CSS de la página de login.

**Rompelo:** comentá la línea del `matcher` (dejá `config = {}` vacío) y recargá `/es/products`: **desaparecen las imágenes**. Mirá por qué:

```powershell
curl.exe -I http://localhost:3000/products/queso-azul.svg
# HTTP/1.1 307 Temporary Redirect
# location: /es/products/queso-azul.svg     ← next-intl le agregó idioma a una imagen
curl.exe -I http://localhost:3000/favicon.ico
# location: /es/favicon.ico
```

Volvelo y repetí: `HTTP/1.1 200 OK`.

| Patrón | Matchea |
|---|---|
| `/about` | `/about` y `/about/team` |
| `/dashboard/:path*` | `/dashboard`, `/dashboard/a/b` |
| `/products/:slug` | `/products/a` (no `/products/a/b`) |
| `/((?!api).*)` | todo lo que no empiece con `/api` |

> "El matcher **tiene que ser un literal**: Next lo analiza en build. Si lo arman con variables, se ignora."

**`has` / `missing` para no correr en prefetch** (solo mostrar, Comarca no lo usa):

```ts
matcher: [{ source: '/((?!api|_next|.*\\..*).*)', missing: [{ type: 'header', key: 'next-router-prefetch' }] }]
```

> "Sirve para cosas que no tienen sentido en un prefetch (un nonce de CSP, clase 18). Para auth e i18n **no**: el prefetch tiene que recibir la misma redirección que la navegación real."

---

## Demo 6 — Por qué el proxy NO es tu capa de auth (20 min)

Ya lo vieron con la cookie inventada. Ahora la historia real:

**CVE-2025-29927 (marzo 2025).** Next usaba internamente el header `x-middleware-subrequest` para evitar loops. Si venía con cierto valor, **salteaba el middleware**. Nadie validaba que viniera de adentro:

```bash
curl -H "x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware" https://app-vulnerable.com/admin
```

Afectó a 11.1.4–15.2.2 en self-hosting (parche: 15.2.3 / 14.2.25).

> "Lo importante no es el header, es la arquitectura: las apps que además chequeaban la sesión **en la página y en el acceso a datos** no tuvieron incidente."

**Las Server Actions pasan por otro lado:** son POST a la ruta donde se usan. Si el matcher excluye esa ruta, la action **no pasa por el proxy**. Si alguien la mueve en un refactor, pierde la cobertura sin enterarse.

| ✅ Sí en el proxy | ❌ No en el proxy |
|---|---|
| Redirección optimista por presencia de cookie | Validar la sesión contra la base en cada request |
| Elegir variante / idioma / región | Lógica de negocio |
| Headers de correlación, CSP con nonce | Queries pesadas, APIs lentas |
| Rate limit grueso (clase 18) | Ser la única barrera de un recurso |
| Rewrites multi-tenant | Leer el body del request |

---

## Demo 7 — Tests del proxy (15 min)

```powershell
npm test
```

```text
 Test Files  2 passed (2)
      Tests  11 passed (11)
```

**Abrí** `src/proxy.test.ts`. Usa las utilidades de `next/experimental/testing/server`:

```ts
import { unstable_doesMiddlewareMatch, isRewrite, getRewrittenUrl, getRedirectUrl } from 'next/experimental/testing/server'

expect(unstable_doesMiddlewareMatch({ config, url: '/_next/static/chunk.js' })).toBe(false)
expect(getRedirectUrl(proxy(new NextRequest('http://localhost/es/dashboard'))))
  .toBe('http://localhost/es/sign-in?next=%2Fes%2Fdashboard')
```

> Ojo: en Next 16.4 la función se llama `unstable_doesMiddlewareMatch` (no "Proxy"), aunque el archivo se llame `proxy.ts`. Lo verificamos en `node_modules/next/dist/experimental/testing/server`.

**Rompelo en vivo:** en el matcher cambiá `.*\\..*` por `.*\\.svg`. Corré `npm test`: falla "no corre en assets ni en /api" por `/favicon.ico`. Un test de tres líneas atrapó un bug silencioso. Volvelo.

Detalle de config que vas a necesitar (`vitest.config.mts`): `server.deps.inline: ['next-intl']`, porque next-intl importa `next/server` sin extensión y Node solo no lo resuelve.

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| La página de login sin CSS | Proxy sin matcher redirige assets | Matcher negativo |
| Loop de redirecciones | La ruta de destino también matchea | `/sign-in` fuera de los prefijos privados |
| El header no llega a la página | `NextResponse.next({ headers })` | `next({ request: { headers } })` |
| `The 'runtime' config is not available in Proxy` | Copiaron `runtime: 'edge'` | Sacarlo: proxy es Node |
| El matcher "no funciona" | Armado con una variable | Tiene que ser literal |
| `curl -I` da error en PowerShell | `curl` es alias de `Invoke-WebRequest` | `curl.exe` |

---

## Tarea

1. Agregá un paso `legacyRedirects` que redirija con 308 `/es/producto/:slug` → `/es/products/:slug`. Después **movelo a `redirects()` de `next.config.ts`** y explicá en el PR por qué es mejor ahí.
2. Paso que bloquee con 403 requests con `User-Agent` vacío a `/checkout` (probalo con `curl.exe -A "" ...`).
3. Tests para: request-id preservado si ya viene en el request, y que `/es/account/orders` sin cookie redirige con el `next` correcto.
4. En el PR, con tus palabras, tres razones por las que la redirección de `/dashboard` **no** protege los datos del panel (una tiene que ser la demo de la cookie inventada).

**Terminado cuando:** `npm test` pasa y `npm run build` muestra `ƒ Proxy (Middleware)` sin warnings.
