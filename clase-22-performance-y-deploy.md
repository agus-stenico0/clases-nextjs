# Clase 22 — Performance y deploy (Docker + Coolify)

**Duración:** 2:30 hs
**Branch de salida:** `clase-22-deploy`
**Lo que se llevan:** React Compiler activado y medido, el bundle bajo control, métricas reales de usuarios (Web Vitals), una imagen Docker `standalone` de producción con migraciones separadas, el deploy en un VPS con Coolify (Postgres, Redis, HTTPS, health check), lo necesario para correr más de una instancia, y un pipeline de CI que no deja pasar un build roto ni una dependencia vulnerable.

**Requisito:** tag `clase-21`. Un VPS con Coolify (el de la agencia o uno de práctica) o, como mínimo, Docker local.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:10 | El dolor: "en mi máquina anda" |
| 0:10 – 0:35 | Performance: React Compiler, bundle, Web Vitals |
| 0:35 – 1:15 | Docker: `standalone`, multi-stage, migraciones, el problema de la base en build |
| 1:15 – 1:50 | **En vivo:** deploy en Coolify |
| 1:50 – 2:10 | Más de una instancia: qué se comparte y qué no |
| 2:10 – 2:30 | CI con GitHub Actions + política de actualizaciones + tarea |

---

## Bloque 1 — El dolor (10 min)

> "Hasta hoy, Comarca vive en `npm run dev`. Producción tiene otras reglas: no hay `.env.local`, la base está en otro lado, puede haber dos contenedores, el build corre en una máquina distinta, y si algo falla a las 3 AM nadie está mirando. Hoy lo llevamos a un servidor de verdad."

---

## Bloque 2 — Performance (25 min)

### 2.1 — React Compiler

Memoiza automáticamente componentes y valores (lo que antes se hacía a mano con `useMemo`, `useCallback`, `memo`). En Next 16 es estable:

```ts
// next.config.ts
reactCompiler: true,
```

```bash
npm i -D babel-plugin-react-compiler
```

> "Ojo: corre a través de Babel, así que dev y build se ponen un poco más lentos. En 16.3+ existe una versión experimental en Rust dentro de Turbopack (`experimental.turbopackRustReactCompiler: true`) que elimina gran parte de ese costo. Prueben las dos y midan."

**Medición:** React DevTools → Profiler → grabá agregar 3 productos al carrito y cambiar cantidades. Antes/después: cantidad de renders de `CartLines`, `ProductCard`, `SiteHeader`. Anotá los números en el PR.

Regla: con el compiler activo **no** agreguen `useMemo`/`useCallback` nuevos "por las dudas". Los existentes no molestan.

### 2.2 — El bundle

- En 16, `next build` ya **no** muestra "First Load JS" (el equipo de Next lo sacó por impreciso con RSC). Para ver qué pesa:
  - El **analizador de bundles de Turbopack** (16.1+, muy mejorado en 16.4: resumen por ruta, tabla, snapshots y diff entre builds). Ver la guía *Package Bundling* de la doc de su versión para el comando exacto.
  - Lighthouse / Chrome DevTools → Coverage.
- Los sospechosos de siempre: librerías de fechas, de gráficos, de markdown, de íconos importadas enteras **en componentes cliente**.

```tsx
// ❌ trae toda la lib al cliente
'use client'
import { format } from 'date-fns'
import * as Icons from 'lucide-react'
```

Soluciones en orden de preferencia:

1. Mover el trabajo a un Server Component (clase 3).
2. Importar solo lo necesario (`import { Heart } from 'lucide-react'`). Next ya optimiza imports de libs conocidas (`lucide-react`, `date-fns`, etc.); para otras, `experimental.optimizePackageImports: ['mi-lib']`.
3. `next/dynamic` para lo que se usa tarde (un mapa, un editor):

```tsx
const ProducerMap = dynamic(() => import('./producer-map'), { ssr: false, loading: () => <MapSkeleton /> })
```

### 2.3 — Web Vitals reales

Lighthouse mide **tu** máquina. Los usuarios tienen otras.

```tsx
// src/components/web-vitals.tsx
'use client'
import { useReportWebVitals } from 'next/web-vitals'

export function WebVitals() {
  useReportWebVitals((metric) => {
    // LCP, INP, CLS, FCP, TTFB
    navigator.sendBeacon?.('/api/vitals', JSON.stringify({
      name: metric.name, value: metric.value, rating: metric.rating, path: location.pathname,
    }))
  })
  return null
}
```

Montalo una vez en el layout de `[locale]`. El handler `/api/vitals` loguea con pino (clase 21) y en Grafana/Loki se arma el p75 por ruta.

> "Objetivos de Google (p75): **LCP < 2,5 s, INP < 200 ms, CLS < 0,1**. Con lo que hicimos en las clases 9–12 (shell estático) y 20 (imágenes), Comarca debería estar cómoda en LCP y CLS. INP es donde pegan los Client Components pesados."

---

## Bloque 3 — Docker de producción (40 min)

### 3.1 — `output: 'standalone'`

```ts
// next.config.ts
output: 'standalone',
```

`next build` genera `.next/standalone/` con un `server.js` y **solo** las dependencias que se usan (trazadas). La imagen final pesa una fracción de copiar `node_modules` entero. Lo que **no** copia solo: `public/` y `.next/static/` (los copiamos nosotros).

### 3.2 — El problema de la base en build

Con Cache Components, `next build` **ejecuta** las funciones `'use cache'` para armar los static shells, y `generateStaticParams` consulta los slugs. O sea: **el build necesita la base** (y Payload, para la landing).

| Opción | Pros | Contras |
|---|---|---|
| Buildear con acceso a la base de producción (o una réplica de lectura) | El shell sale con datos reales | El build depende de la red; hay que pasar credenciales sin filtrarlas en la imagen |
| Buildear en CI contra una base de staging y publicar la imagen | Build reproducible, deploy = pull | El shell sale con datos de staging hasta la primera invalidación |
| Que la DAL no consulte en build (`generateStaticParams` vacío, etc.) | Build sin base | Perdés el prerender: el primer visitante de cada página paga el costo |

Para el curso: **buildear en el VPS con acceso a la base**, pasando la URL como **secreto de BuildKit** (no queda en las capas de la imagen).

### 3.3 — Dockerfile

```dockerfile
# syntax=docker/dockerfile:1.7
# (la línea de syntax tiene que ser la PRIMERA del archivo)
FROM node:22-alpine AS base
RUN apk add --no-cache libc6-compat
WORKDIR /app
ENV NEXT_TELEMETRY_DISABLED=1

# ─── Dependencias ───
FROM base AS deps
COPY package.json package-lock.json prisma.config.ts ./
COPY prisma ./prisma
RUN npm ci   # postinstall → prisma generate

# ─── Build ───
FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
# Las NEXT_PUBLIC_* se pegan en el JS en build: tienen que estar acá
ARG NEXT_PUBLIC_SITE_URL
ENV NEXT_PUBLIC_SITE_URL=$NEXT_PUBLIC_SITE_URL
# Secretos solo durante este RUN (no quedan en la imagen)
RUN --mount=type=secret,id=build_env,target=/app/.env.production.local \
    npm run build

# ─── Migraciones (imagen aparte, corre una vez por deploy) ───
FROM builder AS migrator
CMD ["sh", "-c", "npx prisma migrate deploy && npx payload migrate"]

# ─── Runtime ───
FROM base AS runner
ENV NODE_ENV=production PORT=3000 HOSTNAME=0.0.0.0
RUN addgroup -S nodejs && adduser -S nextjs -G nodejs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder --chown=nextjs:nodejs /app/cache-handlers ./cache-handlers
USER nextjs
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s CMD wget -qO- http://127.0.0.1:3000/api/health || exit 1
CMD ["node", "server.js"]
```

```text
# .dockerignore
node_modules
.next
.git
.env*
!.env.example
src/generated
e2e
test-results
```

Puntos para explicar:

- **Tres imágenes del mismo Dockerfile**: `builder` (no se publica), `migrator` (corre migraciones y termina), `runner` (la app). El runner no tiene la CLI de Prisma ni de Payload: menos superficie, menos peso.
- **Usuario no root** (`nextjs`).
- **`HEALTHCHECK`** contra el health check honesto de la clase 16.
- El cache handler de Redis (clase 11) es un archivo que Next carga en runtime: hay que copiarlo. **Y su dependencia también**: el trazado de `standalone` puede no detectar que `redis` se usa (el handler se carga por ruta, no por `import`). Verificá con `ls .next/standalone/node_modules | grep redis`; si no está, agregalo con `outputFileTracingIncludes: { '/*': ['./node_modules/redis/**', './node_modules/@redis/**'] }` en `next.config.ts`.

### 3.4 — Compose de producción

```yaml
# docker/compose.prod.yml
services:
  migrate:
    build:
      context: ..
      target: migrator
      network: host                   # el build alcanza la base expuesta en 127.0.0.1 del VPS
      secrets: [build_env]
      args: { NEXT_PUBLIC_SITE_URL: '${NEXT_PUBLIC_SITE_URL}' }
    environment:
      DATABASE_URL: ${DATABASE_URL}
      PAYLOAD_DATABASE_URL: ${PAYLOAD_DATABASE_URL}
      PAYLOAD_SECRET: ${PAYLOAD_SECRET}
    restart: 'no'

  web:
    build:
      context: ..
      target: runner
      network: host
      secrets: [build_env]
      args: { NEXT_PUBLIC_SITE_URL: '${NEXT_PUBLIC_SITE_URL}' }
    depends_on:
      migrate: { condition: service_completed_successfully }
    environment:
      APP_ENV: production
      DATABASE_URL: ${DATABASE_URL}
      PAYLOAD_DATABASE_URL: ${PAYLOAD_DATABASE_URL}
      PAYLOAD_SECRET: ${PAYLOAD_SECRET}
      REDIS_URL: ${REDIS_URL}
      BETTER_AUTH_SECRET: ${BETTER_AUTH_SECRET}
      BETTER_AUTH_URL: ${NEXT_PUBLIC_SITE_URL}
      NEXT_SERVER_ACTIONS_ENCRYPTION_KEY: ${NEXT_SERVER_ACTIONS_ENCRYPTION_KEY}
      MP_WEBHOOK_SECRET: ${MP_WEBHOOK_SECRET}
      CATALOG_WEBHOOK_SECRET: ${CATALOG_WEBHOOK_SECRET}
      PREVIEW_SECRET: ${PREVIEW_SECRET}
    volumes:
      - next-cache:/app/.next/cache    # cache de imágenes optimizadas entre reinicios
    restart: unless-stopped

secrets:
  build_env:
    environment: BUILD_ENV_FILE_CONTENT   # contenido de un .env con DATABASE_URL, PAYLOAD_*, etc. para el build

volumes:
  next-cache:
```

> "`depends_on` con `service_completed_successfully`: la app nueva **no arranca** si la migración falla. Es la diferencia entre un deploy fallido (la versión vieja sigue andando) y una caída."

**Validación local:**

```bash
docker compose -f docker/compose.prod.yml build
docker compose -f docker/compose.prod.yml up -d
docker compose -f docker/compose.prod.yml ps        # migrate: exited (0), web: healthy
curl -s localhost:3000/api/health                   # {"status":"ok","db":"up"}
docker image ls | grep comarca                      # runner: ~200 MB, no 1.5 GB
```

---

## Bloque 4 — En vivo: deploy en Coolify (35 min)

### 4.1 — Servicios

En el proyecto de Coolify:

1. **PostgreSQL 17** (recurso de base de datos de Coolify). Creá las dos bases (`comarca`, `comarca_cms`) y un usuario con permisos solo sobre ellas. Puerto publicado **solo en 127.0.0.1** del VPS (para el build), nunca a internet.
2. **Redis 7** (recurso). Sin puerto público.
3. **Aplicación** → Docker Compose → repo de GitHub → `docker/compose.prod.yml`.

### 4.2 — Variables

En la pestaña de variables de la aplicación:

| Variable | Tipo | Notas |
|---|---|---|
| `NEXT_PUBLIC_SITE_URL` | **Build** + runtime | Se pega en el JS |
| `BUILD_ENV_FILE_CONTENT` | **Build**, secreta | `DATABASE_URL=...` (con `127.0.0.1`), `PAYLOAD_DATABASE_URL=...`, `PAYLOAD_SECRET=...` |
| `DATABASE_URL`, `PAYLOAD_DATABASE_URL` | Runtime, secretas | Con el host **interno** de la red de Coolify |
| `BETTER_AUTH_SECRET`, `PAYLOAD_SECRET`, `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` | Runtime, secretas | `openssl rand -base64 32` cada una |
| `REDIS_URL` | Runtime | Host interno |
| `MP_WEBHOOK_SECRET`, `CATALOG_WEBHOOK_SECRET`, `PREVIEW_SECRET` | Runtime, secretas | |

> "Dos URLs de base distintas a propósito: el **build** corre con `network: host` y ve la base en `127.0.0.1`; el **contenedor** en runtime está en la red interna de Coolify y la ve por su nombre de servicio."

### 4.3 — Dominio, HTTPS y proxy reverso

- Dominio `comarca.<tu-dominio>` en el servicio `web`, puerto 3000. Coolify (Traefik) saca el certificado de Let's Encrypt.
- Traefik setea `X-Forwarded-For` y `X-Forwarded-Proto`: ahora sí el `clientIp()` de la clase 18 es confiable (porque **nadie** llega al contenedor sin pasar por Traefik).
- Health check de Coolify: `/api/health`.

### 4.4 — Primer deploy y verificación

```bash
curl -sI https://comarca.<dominio>/es | head -20   # 200, headers de seguridad, CSP
curl -s https://comarca.<dominio>/api/health
curl -s https://comarca.<dominio>/sitemap.xml | head
```

- https://securityheaders.com → A.
- Lighthouse contra producción.
- `/admin` → crear el primer editor del CMS de producción.
- Webhook de MercadoPago apuntando a `https://comarca.<dominio>/api/webhooks/mercadopago`.

> "Deploys siguientes: push a `main` → Coolify buildea, corre `migrate`, levanta el `web` nuevo, verifica el health check y recién ahí corta el tráfico del viejo. Si algo falla, el viejo sigue."

---

## Bloque 5 — Más de una instancia (20 min)

Si escalan `web` a 2 réplicas (o a 2 VPS detrás de un balanceador), revisen esta lista:

| Qué | Por qué | Cómo |
|---|---|---|
| **Cache de `'use cache'`** | Por defecto es memoria de cada proceso | Cache handler de Redis (clase 11) con coordinación de tags |
| **Clave de cifrado de Server Actions** | Cada build/instancia genera la suya; una action iniciada en A y ejecutada en B falla | `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` igual en todas |
| **Mismo build en todas** | IDs de actions y chunks cambian por build | Desplegar la misma imagen; `deploymentId` en `next.config` si hay rollouts graduales |
| **Sesiones** | Better Auth guarda en Postgres | Ya compartido ✅ |
| **Rate limit** | En Redis | Ya compartido ✅ |
| **Archivos subidos (Payload media)** | En disco del contenedor | Adapter de storage S3/R2 de Payload |
| **Cache de imágenes optimizadas** | En `.next/cache` de cada instancia | Aceptable (se recalcula), o CDN adelante |

> "Lo que más se olvida es la clave de las actions: todo anda con una instancia, escalan a dos, y el 50 % de los formularios falla con un error críptico. Ahora ya saben por qué."

---

## Bloque 6 — CI y política de actualizaciones (20 min)

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17-alpine
        env: { POSTGRES_USER: comarca, POSTGRES_PASSWORD: comarca, POSTGRES_DB: comarca_test }
        ports: ['5432:5432']
        options: >-
          --health-cmd "pg_isready -U comarca" --health-interval 5s --health-retries 10
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
    env:
      DATABASE_URL: postgresql://comarca:comarca@localhost:5432/comarca_test
      REDIS_URL: redis://localhost:6379
      NEXT_PUBLIC_SITE_URL: http://localhost:3000
      BETTER_AUTH_SECRET: ci-secret-ci-secret-ci-secret-ci-secret
      BETTER_AUTH_URL: http://localhost:3000
      APP_ENV: test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm audit --omit=dev --audit-level=high
      - run: npm run lint
      - run: npm run typecheck
      - run: npx prisma migrate deploy && npx prisma db seed
      - run: npm test
      - run: npx playwright install --with-deps chromium
      - run: EXPOSE_TESTING_API=1 npx playwright test
```

(Payload en CI necesita su base y `PAYLOAD_SECRET`; agregalos igual que Prisma.)

### 6.1 — Actualizaciones

- **Renovate** (o Dependabot) con PRs automáticos agrupados: `next` + `react` + `react-dom` + `eslint-config-next` juntos.
- `experimental.agentUpgrade: 'security'` (default en 16.4): `next build` avisa cuando hay una versión que corrige una vulnerabilidad de la tuya.
- Upgrades de minor/major: `npx next@canary upgrade --agent=latest` prepara guías y codemods para un agente; ustedes revisan el PR y corren `npm run test:all`.
- **Regla del equipo:** parche de seguridad crítico de Next/React en producción en menos de 24 h. Con este pipeline y Coolify, es un merge.

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| Build falla en Docker con `Can't reach database server` | Prerender necesita la base | `network: host` + secreto de build, o buildear en CI |
| CSS/JS 404 en producción | No se copió `.next/static` | `COPY .next/static` |
| Imágenes de `public/` 404 | No se copió `public` | `COPY public` |
| `NEXT_PUBLIC_SITE_URL` undefined en el navegador | Se pasó solo en runtime | Variable de **build** |
| `Failed to find Server Action` después de escalar | Clave de cifrado distinta por instancia | `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` |
| La app nueva arranca con el schema viejo | Migraciones dentro del `CMD` sin control | Servicio `migrate` + `depends_on` |
| Secretos visibles en `docker history` | `ARG`/`ENV` con secretos | `--mount=type=secret` |

---

## Tarea

1. Dockerfile + `compose.prod.yml` funcionando en local con `docker compose up` desde cero.
2. Deploy en Coolify (o en el VPS de práctica) con dominio y HTTPS. Pegá en el PR: securityheaders.com, Lighthouse de producción, `curl` del health check.
3. React Compiler activado con la medición de renders antes/después.
4. Web Vitals reportando a `/api/vitals` y un log de p75 de LCP por ruta después de 50 visitas (podés generarlas con Playwright).
5. CI en verde en GitHub Actions, con `npm audit` incluido.

**Terminado cuando:** un push a `main` termina en producción sin tocar nada a mano, y romper una migración a propósito deja la versión anterior funcionando.
