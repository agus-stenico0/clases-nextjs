# Clase 1 — Por qué Next.js 16 y el modelo mental

**Duración:** 2:30 hs
**Branch de salida:** `clase-01-setup`
**Lo que se llevan:** el proyecto Comarca creado con los defaults de Next 16.4, la capacidad de explicar qué pasa entre que el usuario pide una URL y ve la página, y la costumbre de leer la salida de `next build`.

**Requisito:** Node 22 (`node -v`), Git, VS Code. En Windows trabajamos **dentro de WSL2** (`~/proyectos`), no en `/mnt/c/...`: Turbopack con el filesystem de Windows es lentísimo y el watcher pierde cambios.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:15 | El dolor: la tienda en React SPA |
| 0:15 – 0:35 | Qué es Next.js y qué cambió en la 16 |
| 0:35 – 1:00 | `create-next-app` y recorrido archivo por archivo |
| 1:00 – 1:30 | **En vivo:** primera página async que lee datos en el servidor |
| 1:30 – 1:50 | El viaje de un request (diagrama) + RSC payload en DevTools |
| 1:50 – 2:15 | `next build`: leer la leyenda ○ ◐ ƒ y provocar el primer error de Cache Components |
| 2:15 – 2:30 | Herramientas (DevTools de Next, AGENTS.md) + tarea |

---

## Bloque 1 — El dolor: la tienda en React SPA (15 min)

Mostrá esto (lo tienen del curso de React, es un Vite + React):

```tsx
// App.tsx (SPA)
export function App() {
  const [products, setProducts] = useState<Product[]>([])
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    fetch('https://api.comarca.dev/products')
      .then((r) => r.json())
      .then((data) => {
        setProducts(data)
        setLoading(false)
      })
  }, [])

  if (loading) return <Spinner />
  return <ProductGrid products={products} />
}
```

Abrí DevTools → Network → recargá con "Slow 4G":

1. Llega un HTML vacío (`<div id="root"></div>`).
2. Baja el bundle JS (React + router + tu app).
3. Se ejecuta React, monta, corre el `useEffect`.
4. **Recién ahí** sale el fetch de productos.
5. Spinner → productos.

Ver código fuente (Ctrl+U): **no hay ni un producto en el HTML**.

> "Cuatro viajes en serie antes de que el usuario vea un queso. Google ve un div vacío. Y todo el código que arma la grilla —formateo de precios, la lib de fechas, el markdown de la descripción— viajó al navegador aunque el usuario nunca lo necesite ejecutar. Next existe para cortar esa cadena."

---

## Bloque 2 — Qué es Next.js y qué cambió en la 16 (20 min)

**Next es un framework de React que corre en el servidor y en el cliente.** El mismo árbol de componentes se reparte entre los dos lados:

- **Server Components** (por defecto): corren solo en el servidor, pueden ser `async`, leer la base, usar secretos. Su JS **no viaja** al navegador.
- **Client Components** (`'use client'`): los de toda la vida. Se renderizan en el server para el HTML inicial y después se hidratan en el navegador para tener interactividad.

Lo que cambió en la 16 y vamos a usar todo el curso:

| Cambio | Qué significa para nosotros |
|---|---|
| **Cache Components** (`cacheComponents: true`) | Todo es dinámico por defecto. Lo que querés cachear lo marcás con `'use cache'`. Se acabó la cache implícita de `fetch`. |
| **Partial Prerendering** como modelo por defecto | Una misma página tiene partes estáticas (shell) y partes que se renderizan por request (huecos con `<Suspense>`). |
| **Partial Prefetching** (16.3) | `<Link>` precarga el "App Shell" de la ruta destino. Navegaciones instantáneas. |
| **`proxy.ts`** reemplaza a `middleware.ts` | Mismo concepto, nuevo nombre, runtime Node fijo. |
| **Turbopack** por defecto en `dev` y `build` | Sin flags. Cache en disco entre reinicios. |
| **APIs async** obligatorias | `params`, `searchParams`, `cookies()`, `headers()` siempre con `await`. |
| **Nuevas APIs de cache** | `updateTag`, `revalidateTag(tag, perfil)`, `refresh`, `cacheLife`, `cacheTag` estables. |
| **React 19.3** (16.4) | View Transitions estables, `<Activity>`, `useEffectEvent`. |
| `next lint` eliminado | ESLint (o Biome) se corre directo. |

> "Desde 16.4, el equipo de Next recomienda Cache Components para **toda** app nueva, y en Next 17 va a ser el único modelo. No vamos a aprender el modelo viejo como principal. Lo vamos a ver en la clase 11 solo para que sepan leer código legacy."

---

## Bloque 3 — Crear el proyecto (25 min)

### 3.1 — Versión de Node y create-next-app

```bash
node -v          # v22.x
mkdir -p ~/proyectos && cd ~/proyectos

npx create-next-app@latest comarca-web \
  --ts --tailwind --eslint --app --src-dir \
  --import-alias "@/*" --use-npm
```

Si pregunta por **React Compiler**: **No** por ahora (lo activamos en la clase 22, y queremos ver primero los renders "de verdad").

```bash
cd comarca-web
npm ls next react        # next@16.4.x, react@19.3.x
```

**Si `next` es 16.4.0 o menor, actualizá ya:**

```bash
npm install next@latest
npm ls next              # tiene que decir 16.4.1 o mayor
```

> "Esto no es burocracia. El 14 de octubre salió un parche para dos ejecuciones remotas de código **sin autenticación**. Una en `use cache`, que es justo lo que vamos a usar todo el curso. Antes de escribir una línea, la versión tiene que estar parcheada."

### 3.2 — Recorrido archivo por archivo

```text
comarca-web/
├── AGENTS.md              ← instrucciones para agentes de IA (Next lo mantiene)
├── eslint.config.mjs      ← ESLint flat config
├── next.config.ts
├── next-env.d.ts          ← generado, NO va al repo
├── package.json
├── postcss.config.mjs     ← Tailwind 4 vía PostCSS
├── public/                ← archivos servidos tal cual en /
├── src/
│   └── app/
│       ├── favicon.ico    ← metadata file: genera el <link rel="icon">
│       ├── globals.css    ← @import "tailwindcss"
│       ├── layout.tsx     ← root layout: <html> y <body>
│       └── page.tsx       ← la ruta "/"
└── tsconfig.json
```

Abrí `next.config.ts`:

```ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  cacheComponents: true,
  partialPrefetching: true,
}

export default nextConfig
```

> "Estas dos líneas son el modelo nuevo. `create-next-app` las pone por defecto desde 16.4. Si algún día abren un proyecto 16.0–16.3 y no están, ese proyecto usa el modelo viejo de cache. Lo primero que hacen en un proyecto ajeno es mirar este archivo."

`package.json`:

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint ."
  }
}
```

Señalá: no hay `--turbopack` (es el default) y `lint` llama a ESLint directo (`next lint` ya no existe).

### 3.3 — Correrlo

```bash
npm run dev
```

**Validación:** http://localhost:3000 muestra la página de bienvenida. En la terminal aparece `Ready in ...ms`. En la esquina de abajo a la izquierda del navegador está el ícono de **Next DevTools**.

Mostrá dos cosas que sorprenden:

```bash
ls .next            # dev/  ← next dev escribe en .next/dev
npm run build       # en otra terminal, CON el dev corriendo
```

> "Desde la 16, `dev` y `build` usan carpetas distintas, así que pueden correr a la vez. Y hay un lockfile: si intentan levantar dos `next dev` en el mismo proyecto, el segundo se niega. Antes eso corrompía la carpeta `.next`."

---

## Bloque 4 — En vivo: la primera página que lee datos en el servidor (30 min)

Todavía no hay base de datos. Usamos un JSON local para ver el mecanismo puro.

```json
// src/data/products.json
[
  { "id": "1", "slug": "queso-azul", "name": "Queso azul de Tandil", "priceCents": 1250000, "producer": "La Quesera" },
  { "id": "2", "slug": "miel-monte", "name": "Miel de monte", "priceCents": 680000, "producer": "Apícola Sur" },
  { "id": "3", "slug": "malbec-joven", "name": "Malbec joven", "priceCents": 940000, "producer": "Finca Los Álamos" }
]
```

```tsx
// src/app/page.tsx
import { readFileSync } from 'node:fs'
import path from 'node:path'

type Product = {
  id: string
  slug: string
  name: string
  priceCents: number
  producer: string
}

const formatter = new Intl.NumberFormat('es-AR', { style: 'currency', currency: 'ARS' })

function getProducts(): Product[] {
  const file = readFileSync(path.join(process.cwd(), 'src/data/products.json'), 'utf-8')
  return JSON.parse(file)
}

export default async function HomePage() {
  const products = getProducts()

  return (
    <main className="mx-auto max-w-3xl p-8">
      <h1 className="mb-6 text-3xl font-bold">Comarca</h1>
      <ul className="space-y-3">
        {products.map((p) => (
          <li key={p.id} className="rounded-lg border p-4">
            <p className="font-medium">{p.name}</p>
            <p className="text-sm text-neutral-500">{p.producer}</p>
            <p className="mt-2 font-semibold">{formatter.format(p.priceCents / 100)}</p>
          </li>
        ))}
      </ul>
    </main>
  )
}
```

Marcá en voz alta:

1. **El componente es `async`.** En React SPA eso es imposible. (Hoy no espera nada, pero en la clase 5 va a esperar a Postgres.)
2. **Usa `node:fs`.** Es código de servidor. Nunca llega al navegador.
3. **No hay `useState` ni `useEffect`.** No hay estado de carga porque los datos están antes de que el HTML salga.

> **Ojo con un detalle que los va a confundir si lo cambian:** usamos `readFileSync` a propósito. Con Cache Components, la E/S **síncrona** de recursos locales se considera *predecible* y entra sola en el prerender. Si lo cambian por `await readFile(...)` (async) **dentro del render**, Next lo trata como dato no cacheado y les pide `'use cache'` o `<Suspense>`. Para async local, la doc recomienda leerlo una vez a nivel de módulo. Si alguien lo prueba, perfecto: es un adelanto de la clase 9.

**Validación (hacela con ellos, uno por uno):**

1. Ver código fuente → los tres productos **están en el HTML**.
2. Network → filtrá por Fetch/XHR → **no hay ningún request** a una API de productos.
3. Buscá en los JS descargados (Sources → Ctrl+Shift+F) "readFile" o "products.json" → **no aparece**.

> "El navegador nunca supo que existe un archivo JSON. Recibió HTML ya armado. Eso es un Server Component."

### 4.1 — ¿Y esto se cachea o se lee en cada request?

Agregá un log:

```tsx
function getProducts(): Product[] {
  console.log('[getProducts] leyendo JSON')
  // ...
}
```

Recargá varias veces en dev: el log sale **en la terminal** (no en la consola del navegador) en cada recarga. Ojo que desde 16.2 Next reenvía los logs del navegador a la terminal y viceversa en dev, así que también lo pueden ver con prefijo en la consola. Explicá que en dev todo se recalcula siempre; la pregunta real es qué pasa en **build**, y eso lo vemos en el bloque 6.

---

## Bloque 5 — El viaje de un request (20 min)

Dibujá esto en el pizarrón (y dejalo dibujado todo el curso, lo vamos a ir completando):

```text
 Navegador                         Servidor Next.js
 ─────────                         ────────────────
 GET /productos/queso-azul ──────▶ 1. next.config: headers / redirects
                                   2. proxy.ts            (clase 7)
                                   3. match de ruta en app/
                                   4. ¿Hay static shell?  (clase 9)
                                      sí → se manda YA (CDN / disco)
                                   5. Render de Server Components
                                      - await de datos     (clase 6)
                                      - 'use cache'        (clase 10)
                                      - <Suspense> → stream (clase 6)
                                   6. Serializa:
                                      - HTML (primera carga)
                                      - RSC payload (navegaciones)
 ◀──────────────────── stream ────
 7. Pinta HTML
 8. Baja JS de los Client Components
 9. Hidrata las islas interactivas
10. <Link> visibles → prefetch del App Shell (clase 12)
```

### 5.1 — El RSC payload

Agregá una segunda página para navegar:

```tsx
// src/app/about/page.tsx
import Link from 'next/link'

export default function AboutPage() {
  return (
    <main className="p-8">
      <h1 className="text-2xl font-bold">Sobre Comarca</h1>
      <Link href="/" className="underline">Volver</Link>
    </main>
  )
}
```

Y un link en la home: `<Link href="/about">Sobre nosotros</Link>`.

En Network: navegá de `/` a `/about` haciendo click. **No baja un HTML.** Baja un request con `?_rsc=...` y `content-type: text/x-component`. Abrilo:

```text
0:{"P":null,"b":"...","c":["","about"],...}
1:["$","main",null,{"className":"p-8","children":[["$","h1",null,...
```

> "Eso es el RSC payload: el árbol de React ya renderizado, en un formato que React del navegador sabe aplicar sin recargar. La primera carga es HTML para que se vea rápido y lo lea Google; las navegaciones son RSC para que no se pierda el estado del cliente."

---

## Bloque 6 — `next build`: leer la leyenda y el primer error (25 min)

### 6.1 — Build limpio

```bash
npm run build
```

Salida (resumida):

```text
Route (app)
┌ ○ /
├ ○ /_not-found
└ ○ /about

○  (Static)            prerendered as static content
◐  (Partial Prerender) prerendered as static HTML with dynamic server-streamed content
ƒ  (Dynamic)           server-rendered on demand
```

> "La home lee un archivo… y es estática. ¿Por qué? Porque leer un archivo local que no depende del request es **predecible**: Next lo ejecuta en build y guarda el resultado. El log de `getProducts` salió durante el build, no va a salir cuando un usuario entre."

Comprobalo:

```bash
npm run start
# recargá / varias veces → el log NO aparece
```

### 6.2 — Provocar el error a propósito

Agregá la hora en la home:

```tsx
export default async function HomePage() {
  const products = getProducts()
  const now = new Date().toLocaleTimeString('es-AR')

  return (
    <main className="mx-auto max-w-3xl p-8">
      <p className="text-xs text-neutral-400">Generado a las {now}</p>
      {/* ... */}
    </main>
  )
}
```

Con `npm run dev` abierto, el overlay de Next muestra un insight **blocking-prerender-current-time** con dos opciones:

- **Dynamic:** generarlo en cada request → `await connection()` + `<Suspense>`.
- **Cache:** cachear el valor → `'use cache'`.

Y `npm run build` falla con un error que apunta a la línea.

> "Esto es lo más importante de hoy. En el modelo viejo, Next decidía **solo** que tu página pasaba a ser dinámica porque usaste la hora, y vos te enterabas en producción cuando la factura del hosting subía. Ahora Next **no decide por vos**: te frena y te pregunta. ¿Querés la hora de cada request, o te alcanza con la hora del build? Las dos respuestas son válidas, pero las tenés que dar vos."

Resolvelo con la opción dinámica para mostrar el mecanismo (lo explicamos a fondo en la clase 9):

```tsx
import { Suspense } from 'react'
import { connection } from 'next/server'

async function GeneratedAt() {
  await connection() // "esperá a que haya un request real"
  return (
    <p className="text-xs text-neutral-400">
      Generado a las {new Date().toLocaleTimeString('es-AR')}
    </p>
  )
}

export default async function HomePage() {
  const products = getProducts()
  return (
    <main className="mx-auto max-w-3xl p-8">
      <Suspense fallback={<p className="text-xs text-neutral-400">…</p>}>
        <GeneratedAt />
      </Suspense>
      {/* lista de productos igual que antes */}
    </main>
  )
}
```

```bash
npm run build
```

```text
┌ ◐ /
```

> "Media luna: **Partial Prerender**. La lista de productos y todo el HTML están precalculados; solo el renglón de la hora se rellena por request. Ese símbolo es el que vamos a perseguir todo el curso: máximo contenido en el shell, mínimos huecos."

**Validación:** `npm run start`, recargá con Network abierto. El HTML llega con los productos y el `…`; un instante después, en el mismo response (streaming), aparece la hora. Mirá el response del documento: el HTML trae un `<template>` y un script que reemplaza el fallback.

---

## Bloque 7 — Herramientas (15 min)

### 7.1 — Next DevTools

Clic en el ícono de Next abajo a la izquierda:

- **Issues / Insights:** los errores de prerender como el que vimos, con "Copy prompt" para pasárselo a un agente.
- **Instant Insights** (16.3): navegaciones que no son instantáneas.
- **Navigation Inspector:** pausa la página en su estado de carga inicial. Lo usamos en la clase 12.
- **Route info:** si la ruta es estática, parcial o dinámica.

### 7.2 — AGENTS.md

```bash
cat AGENTS.md
```

Next mantiene un bloque que apunta a `node_modules/next/dist/docs/`. Si usan Claude, Cursor u OpenCode, el agente lee la documentación **de su versión exacta**.

> "Úsenlo, pero no le deleguen el modelo mental. Un agente les puede poner `'use cache'` donde no va, y si ustedes no saben leer la salida del build, no se van a dar cuenta."

### 7.3 — `.gitignore` y primer commit

Verificá que `.gitignore` tenga `.next/`, `node_modules/`, `.env*` y `next-env.d.ts`.

```bash
git init && git add -A && git commit -m "chore: create comarca-web"
git checkout -b clase-01-setup
git tag clase-01
```

---

## Errores comunes de esta clase

| Síntoma | Causa | Solución |
|---|---|---|
| `next dev` tarda minutos o no detecta cambios | Proyecto en `/mnt/c/` desde WSL | Moverlo a `~/proyectos` dentro de WSL |
| `Another next dev server is already running` | Lockfile de 16: ya hay un dev corriendo | Cerrar el otro proceso |
| `params.slug` da undefined / error de tipo | Copiaron código de Next 14 | `const { slug } = await params` |
| El log de un Server Component no aparece en la consola del navegador | Corre en el servidor | Miralo en la terminal |

---

## Tarea

1. Agregá a `products.json` 6 productos más con `category` (`quesos`, `miel`, `vinos`).
2. Creá `src/app/categories/page.tsx` que lea el JSON y muestre los productos **agrupados por categoría**, todo en el servidor.
3. Agregá en esa página un "Visitante número N" con un número aleatorio **distinto en cada request**, sin que la página deje de ser `◐`.
4. Pegá en el PR la salida de `npm run build` y explicá en dos líneas qué parte de `/categories` está en el shell y cuál no.

**Terminado cuando:** `npm run build` pasa, `/categories` aparece con `◐` y el número cambia al recargar en `npm run start`.
