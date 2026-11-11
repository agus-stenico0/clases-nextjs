# Clase 17 — Autenticación con Better Auth + Prisma

**Duración:** 2:30 hs
**Branch de salida:** `clase-17-auth`
**Lo que se llevan:** registro e ingreso con email y contraseña usando Server Actions, sesiones guardadas en nuestra base, la DAL resolviendo al usuario real (`requireUser`, `requireProducer`), autorización por rol y por dueño, datos por usuario cacheados de forma segura, y el chequeo optimista del proxy con la cookie real.

**Requisito:** tag `clase-16`.

---

## Guion de la clase

| Tiempo | Qué pasa |
|---|---|
| 0:00 – 0:10 | El dolor: todos son "cliente demo" |
| 0:10 – 0:25 | Sesiones: cookie opaca vs JWT, y por qué Better Auth |
| 0:25 – 0:55 | Instalación: config, tablas, route handler |
| 0:55 – 1:25 | **En vivo:** registro, ingreso y salida con Server Actions |
| 1:25 – 1:55 | La DAL con sesión real: `requireUser`, `requireProducer`, ownership |
| 1:55 – 2:15 | Sesión + Cache Components: dónde leerla sin romper el shell |
| 2:15 – 2:30 | Proxy con la cookie real + tarea |

---

## Bloque 1 — El dolor (10 min)

```ts
// src/server/dal/session.ts (provisorio desde la clase 13)
export const requireUser = cache(async () => db.user.findUnique({ where: { email: 'cliente@comarca.dev' } }))
```

> "Toda la app ya pregunta '¿quién es el usuario?' en un solo lugar. Hoy cambiamos la respuesta de ese lugar y el resto del sistema no se entera. Esto es lo que compramos en la clase 6 cuando decidimos que la identidad se resuelve **adentro** de la DAL."

---

## Bloque 2 — Conceptos (15 min)

| | Sesión en base (cookie opaca) | JWT en cookie |
|---|---|---|
| Qué guarda la cookie | Un token aleatorio | Los datos del usuario firmados |
| Validar | Consulta a la base (o a una cache) | Verificar firma, sin base |
| Cerrar sesión / banear | Inmediato: se borra la fila | Hay que esperar a que venza (o mantener una lista negra) |
| Costo | Una query por request (mitigable) | Cero queries |

Better Auth usa **sesiones en base**, con un **cookie cache** opcional (guarda la sesión firmada unos minutos en otra cookie para no consultar la base en cada request). Lo mejor de los dos.

Por qué Better Auth para el curso:

- Las tablas viven **en nuestra Postgres** con Prisma (nada de servicio externo, nada de datos de usuarios en terceros).
- Email+contraseña, OAuth, 2FA, magic links, organizaciones: por plugins.
- Rate limit de login incluido.
- Integración documentada con Next (route handler, plugin para cookies en Server Actions, proxy).

> La guía oficial de Next sobre auth con Cache Components usa `iron-session` (cookie cifrada sin base) y Auth.js es otra opción muy usada. Los **patrones** de esta clase (DAL, `requireUser`, verificar en cada action) son los mismos con cualquier librería. Si el template de la agencia usa otra, cambia el archivo `src/server/auth.ts` y nada más.

---

## Bloque 3 — Instalación (30 min)

```bash
npm i better-auth
openssl rand -base64 32   # → BETTER_AUTH_SECRET
```

```bash
# .env.local
BETTER_AUTH_SECRET="..."
BETTER_AUTH_URL="http://localhost:3000"
```

Sumalas al schema de env de la clase 4 (`BETTER_AUTH_SECRET: z.string().min(32)`, `BETTER_AUTH_URL: z.url()`).

### 3.1 — Configuración

```ts
// src/server/auth.ts
import 'server-only'
import { betterAuth } from 'better-auth'
import { prismaAdapter } from 'better-auth/adapters/prisma'
import { nextCookies } from 'better-auth/next-js'
import { db } from '@/server/db'
import { env } from '@/lib/env/server'

export const auth = betterAuth({
  secret: env.BETTER_AUTH_SECRET,
  baseURL: env.BETTER_AUTH_URL,
  database: prismaAdapter(db, { provider: 'postgresql' }),

  emailAndPassword: {
    enabled: true,
    minPasswordLength: 10,
    autoSignIn: true,
  },

  user: {
    additionalFields: {
      // El rol existe en nuestro schema (clase 5). input: false → el usuario NO lo puede mandar al registrarse.
      role: { type: 'string', defaultValue: 'customer', input: false },
    },
  },

  session: {
    expiresIn: 60 * 60 * 24 * 7,  // 7 días
    updateAge: 60 * 60 * 24,      // renueva la expiración como mucho una vez por día
    cookieCache: { enabled: true, maxAge: 5 * 60 }, // 5 min sin ir a la base
  },

  rateLimit: { enabled: true, window: 60, max: 10 }, // por IP, en los endpoints de auth

  plugins: [nextCookies()], // SIEMPRE el último: permite setear cookies desde Server Actions
})

export type Session = typeof auth.$Infer.Session
```

> "`input: false` en `role` es seguridad, no prolijidad. Sin eso, alguien manda `role: 'admin'` en el body del registro y se auto-promueve. Es un clásico de *mass assignment*."

### 3.2 — Tablas

```bash
npx auth@latest generate     # agrega Session, Account, Verification a prisma/schema.prisma
git diff prisma/schema.prisma
```

**Revisen el diff con los alumnos.** En la clase 5 dejamos `User` con la forma que espera Better Auth (`id String @id`, `name`, `email`, `emailVerified`, `image`, timestamps), así que el generador solo debería **agregar** las otras tablas y las relaciones (`sessions`, `accounts`) en `User`. Si propone cambiar algo de `User` (por ejemplo el tipo de `role`), conservá nuestro enum.

```bash
npx prisma migrate dev --name better_auth && npx prisma generate
```

### 3.3 — El route handler

```ts
// src/app/api/auth/[...all]/route.ts
import { toNextJsHandler } from 'better-auth/next-js'
import { auth } from '@/server/auth'

export const { GET, POST } = toNextJsHandler(auth)
```

Todos los endpoints de Better Auth (`/api/auth/sign-in/email`, `/api/auth/get-session`, OAuth callbacks…) pasan por acá. El proxy ya excluye `/api`.

**Validación:**

```bash
curl -s localhost:3000/api/auth/get-session   # null (sin sesión)
```

---

## Bloque 4 — En vivo: registro, ingreso y salida (30 min)

Usamos **Server Actions** que llaman a `auth.api.*` del lado del servidor. Ventajas: formularios que andan sin JS, validación con nuestros schemas, y nada de lógica de auth en el cliente.

### 4.1 — Schemas

```ts
// src/features/auth/schemas.ts
import { z } from 'zod'

export const signUpSchema = z.object({
  name: z.string().trim().min(2).max(80),
  email: z.email().max(200),
  password: z.string().min(10).max(128),
})

export const signInSchema = z.object({
  email: z.email(),
  password: z.string().min(1),
  next: z.string().optional(),
})

export type AuthFormState = { error?: string; fieldErrors?: Record<string, string[]>; email?: string }
```

### 4.2 — Redirect seguro

```ts
// src/lib/safe-redirect.ts
/** Solo rutas internas. Evita ?next=https://phishing.com y ?next=//phishing.com */
export function safeNextPath(next: string | undefined | null, fallback = '/') {
  if (!next || !next.startsWith('/') || next.startsWith('//') || next.startsWith('/\\')) return fallback
  return next
}
```

> "El `?next=` del proxy de la clase 7 es input del usuario. Si redirigimos a lo que diga sin validar, cualquiera arma un link `comarca.com/sign-in?next=https://comarca-falso.com` que, después de un login real, manda a la víctima a un sitio clonado. Eso es un *open redirect*."

### 4.3 — Actions

```ts
// src/features/auth/actions.ts
'use server'

import { z } from 'zod'
import { headers } from 'next/headers'
import { APIError } from 'better-auth/api'
import { getLocale } from 'next-intl/server'
import { auth } from '@/server/auth'
import { redirect } from '@/i18n/navigation'
import { safeNextPath } from '@/lib/safe-redirect'
import { signInSchema, signUpSchema, type AuthFormState } from './schemas'

export async function signUpAction(_prev: AuthFormState, formData: FormData): Promise<AuthFormState> {
  const parsed = signUpSchema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) {
    return { fieldErrors: z.flattenError(parsed.error).fieldErrors, email: String(formData.get('email') ?? '') }
  }

  try {
    await auth.api.signUpEmail({ body: parsed.data, headers: await headers() })
  } catch (err) {
    if (err instanceof APIError) {
      // No revelar si el email existe: mismo mensaje genérico
      return { error: 'No pudimos crear la cuenta. Revisá los datos o ingresá si ya tenés una.', email: parsed.data.email }
    }
    throw err
  }

  redirect({ href: '/account', locale: await getLocale() })
}

export async function signInAction(_prev: AuthFormState, formData: FormData): Promise<AuthFormState> {
  const parsed = signInSchema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return { error: 'Email o contraseña inválidos' }

  try {
    await auth.api.signInEmail({
      body: { email: parsed.data.email, password: parsed.data.password },
      headers: await headers(), // IP y user-agent para el rate limit y la sesión
    })
  } catch (err) {
    if (err instanceof APIError) return { error: 'Email o contraseña inválidos', email: parsed.data.email }
    throw err
  }

  redirect({ href: safeNextPath(parsed.data.next, '/account'), locale: await getLocale() })
}

export async function signOutAction() {
  await auth.api.signOut({ headers: await headers() })
  redirect({ href: '/', locale: await getLocale() })
}
```

> "`nextCookies()` es lo que hace que la cookie de sesión que crea `signInEmail` llegue al navegador desde una Server Action. Sin el plugin, el login 'funciona' pero el usuario sigue deslogueado."

### 4.4 — Página de ingreso

```tsx
// src/app/(frontend)/[locale]/(auth)/sign-in/page.tsx
import { Suspense } from 'react'
import { setRequestLocale } from 'next-intl/server'
import { SignInForm } from '@/features/auth/components/sign-in-form'

export default async function SignInPage(props: PageProps<'/[locale]/sign-in'>) {
  const { locale } = await props.params
  setRequestLocale(locale)
  return (
    <div className="mx-auto w-full max-w-sm space-y-6">
      <h1 className="text-2xl font-semibold">Ingresar</h1>
      <Suspense>
        <SignInFormWithNext searchParams={props.searchParams} />
      </Suspense>
    </div>
  )
}

async function SignInFormWithNext({ searchParams }: Pick<PageProps<'/[locale]/sign-in'>, 'searchParams'>) {
  const { next } = await searchParams
  return <SignInForm next={typeof next === 'string' ? next : undefined} />
}
```

```tsx
// src/features/auth/components/sign-in-form.tsx
'use client'

import { useActionState } from 'react'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Field, FieldLabel } from '@/components/ui/field'
import { signInAction } from '../actions'

export function SignInForm({ next }: { next?: string }) {
  const [state, action, pending] = useActionState(signInAction, {})
  return (
    <form action={action} className="grid gap-4">
      {next && <input type="hidden" name="next" value={next} />}
      {state.error && <p role="alert" className="text-sm text-destructive">{state.error}</p>}
      <Field>
        <FieldLabel htmlFor="email">Email</FieldLabel>
        <Input id="email" name="email" type="email" autoComplete="email" defaultValue={state.email} required />
      </Field>
      <Field>
        <FieldLabel htmlFor="password">Contraseña</FieldLabel>
        <Input id="password" name="password" type="password" autoComplete="current-password" required />
      </Field>
      <Button disabled={pending}>{pending ? 'Ingresando…' : 'Ingresar'}</Button>
    </form>
  )
}
```

**Validación:**

1. Registrate en `/es/sign-up` → redirige a `/es/account`.
2. DevTools → Application → Cookies: `better-auth.session_token` (`HttpOnly`, `SameSite=Lax`) y la del cookie cache.
3. Prisma Studio: fila en `user` (con `role = customer`), `session` y `account` (con el hash de la contraseña — **nunca** en `user`).
4. 11 intentos de login fallidos en un minuto → el rate limit responde con error (Better Auth devuelve 429).
5. `/es/sign-in?next=https://google.com` → después de ingresar, termina en `/es/account`, no en Google.

---

## Bloque 5 — La DAL con sesión real (30 min)

### 5.1 — `session.ts` definitivo

```ts
// src/server/dal/session.ts
import 'server-only'
import { cache } from 'react'
import { headers } from 'next/headers'
import { getLocale } from 'next-intl/server'
import { auth } from '@/server/auth'
import { db } from '@/server/db'
import { redirect } from '@/i18n/navigation'

export type CurrentUser = { id: string; name: string; role: 'customer' | 'producer' | 'admin' }

/** Una lectura de sesión por request, la pidan cuantos componentes la pidan. */
export const getCurrentUser = cache(async (): Promise<CurrentUser | null> => {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session) return null
  return { id: session.user.id, name: session.user.name, role: session.user.role as CurrentUser['role'] }
})

export async function requireUser(): Promise<CurrentUser> {
  const user = await getCurrentUser()
  if (!user) redirect({ href: '/sign-in', locale: await getLocale() })
  return user!
}

export async function requireProducer() {
  const user = await requireUser()
  if (user.role !== 'producer' && user.role !== 'admin') {
    redirect({ href: '/account?error=not_producer', locale: await getLocale() })
  }
  const producer = await db.producer.findUnique({
    where: { ownerId: user.id },
    select: { id: true, slug: true, name: true },
  })
  if (!producer) redirect({ href: '/account?error=no_producer', locale: await getLocale() })
  return { user, ...producer! }
}
```

> "Fíjense qué **no** devolvemos: ni el token, ni el email, ni la sesión entera. `CurrentUser` es un DTO con lo que la app usa. Si un día un componente cliente recibe el usuario, recibe esto."

> **Next 16 tiene `forbidden()` y `unauthorized()`** (con `forbidden.tsx` / `unauthorized.tsx`) detrás de `experimental.authInterrupts`. Son más correctos que redirigir (devuelven 403/401). Si tu versión los tiene estables, usalos en `requireProducer`.

### 5.2 — Autorización por dueño (anti-IDOR)

Volvé a la DAL del panel (clases 11 y 13), donde dejamos el comentario `⚠️ falta verificar`:

```ts
// src/server/dal/producer-products.ts
import 'server-only'
import { db } from '@/server/db'
import { requireProducer } from './session'

export async function setProductPrice(productId: string, priceCents: number) {
  const producer = await requireProducer()

  // El filtro por producerId ES la autorización. Si el producto no es suyo, no lo encuentra.
  const updated = await db.product.updateManyAndReturn({
    where: { id: productId, producerId: producer.id },
    data: { priceCents },
    select: { slug: true, producerId: true, category: { select: { slug: true } } },
  })
  if (updated.length === 0) throw new Error('NOT_FOUND') // mismo error que si no existiera: no revelamos que existe
  return updated[0]
}

export async function createProductForCurrentProducer(input: ProductFormValues) {
  const producer = await requireProducer()
  // producerId sale de la SESIÓN, nunca del formulario
  // ...
}
```

> "Regla: el `producerId` / `userId` de una escritura **sale de la sesión**, nunca del input. Y en lecturas/escrituras de recursos ajenos, el `where` incluye al dueño. Así un IDOR es imposible por construcción, no por acordarse de un `if`."

(`updateManyAndReturn` está disponible en Postgres desde Prisma 6.2. Alternativa: `updateMany` + `findFirst`.)

### 5.3 — Datos por usuario cacheados, de forma segura

Patrón de la guía oficial "Authentication with Cache Components":

```ts
// src/server/dal/orders.ts
import { cacheLife, cacheTag } from 'next/cache'
import { requireUser } from './session'
import { tags } from '@/server/cache-tags'

// Exportada: resuelve el usuario de la sesión. Nadie puede pedir pedidos de otro.
export async function listMyOrders() {
  const user = await requireUser()
  return listOrdersByUserId(user.id)
}

// NO exportada: cacheada por userId. Si se exportara, cualquiera podría pasarle otro ID.
async function listOrdersByUserId(userId: string) {
  'use cache'
  cacheLife('minutes')
  cacheTag(tags.orders(userId))
  return db.order.findMany({
    where: { userId },
    select: { id: true, status: true, totalCents: true, createdAt: true, _count: { select: { items: true } } },
    orderBy: { createdAt: 'desc' },
    take: 50,
  })
}
```

Y `placeOrderAction` / el webhook ya invalidan `tags.orders(userId)`.

> "Los tags y claves quedan en texto plano: el `userId` está bien (es un ID opaco), el email no."

---

## Bloque 6 — Sesión y Cache Components (20 min)

Leer la sesión usa `headers()` → es **dato de request** → todo componente que la lea va dentro de un `<Suspense>`.

### 6.1 — El menú de usuario del header

```tsx
// src/components/layout/user-menu.tsx
import { getCurrentUser } from '@/server/dal/session'
import { Link } from '@/i18n/navigation'
import { UserMenuDropdown } from './user-menu-dropdown' // cliente: DropdownMenu de shadcn + signOutAction

export async function UserMenu() {
  const user = await getCurrentUser()
  if (!user) return <Link href="/sign-in" className="text-sm">Ingresar</Link>
  return <UserMenuDropdown name={user.name} isProducer={user.role !== 'customer'} />
}

export function UserMenuFallback() {
  return <span className="inline-block h-5 w-16 animate-pulse rounded bg-muted" />
}
```

En `SiteHeader`, al lado del `CartBadge`, cada uno con su boundary.

### 6.2 — Páginas privadas sin shell vacío

```tsx
// src/app/(frontend)/[locale]/account/page.tsx
export default async function AccountPage(props: PageProps<'/[locale]/account'>) {
  const { locale } = await props.params
  setRequestLocale(locale)
  return (
    <div className="space-y-8">
      <h1 className="text-2xl font-semibold">Mi cuenta</h1>       {/* en el shell */}
      <Suspense fallback={<ProfileSkeleton />}><Profile /></Suspense>
      <Suspense fallback={<OrdersSkeleton />}><MyOrders /></Suspense>
    </div>
  )
}
```

Ahora podemos sacar el `instant = false` que pusimos en el panel en la clase 11: las lecturas de sesión están en boundaries.

### 6.3 — `use cache: private` para navegaciones instantáneas (opcional)

Si `getCurrentUser` se marca con `'use cache: private'` y una vida (`cacheLife({ stale: 300 })`), el resultado se guarda **solo en el navegador** y entra en el prefetch del App Shell: entrar a `/account` desde cualquier lado muestra el nombre al instante.

```ts
export async function getCurrentUserPrivate() {
  'use cache: private'
  cacheLife({ stale: 300 })
  const session = await auth.api.getSession({ headers: await headers() })
  return session ? { id: session.user.id, name: session.user.name, role: session.user.role } : null
}
```

Trade-off: durante el `stale`, un cambio de rol no se refleja en **ese navegador** hasta navegar con datos frescos. Por eso lo usamos para **mostrar** (menú, saludo) y nunca para **autorizar**: `requireUser`/`requireProducer` siguen leyendo la sesión real.

---

## Bloque 7 — Proxy con la cookie real (10 min)

```ts
// src/proxy.ts
import { getSessionCookie } from 'better-auth/cookies'

function optimisticAuth(req: NextRequest) {
  const { locale, path } = splitLocale(req.nextUrl.pathname)
  if (!locale) return undefined
  if (!PRIVATE_PREFIXES.some((p) => path === p || path.startsWith(`${p}/`))) return undefined
  if (getSessionCookie(req)) return undefined // solo existencia, no validez
  // ...redirect a /{locale}/sign-in?next=...
}
```

> "La doc de Better Auth lo marca en mayúsculas: `getSessionCookie` **no es seguro**, solo mira si la cookie existe. Está bien: el proxy es UX. La seguridad está en `requireUser`/`requireProducer`, que validan contra la base en cada página, action y handler."

Sí se podría validar la sesión completa en el proxy (corre en Node desde la 16), pero eso suma una consulta (o una verificación de cookie cache) a **cada** request que matchee, incluidos prefetches. No lo hacemos.

---

## Errores comunes

| Síntoma | Causa | Solución |
|---|---|---|
| Login "exitoso" pero sigue deslogueado | Falta `nextCookies()` o no es el último plugin | Agregarlo al final |
| `role` llega como `admin` desde el registro | `input` no está en `false` | `input: false` |
| Build falla en páginas con `getCurrentUser` | Lectura de sesión fuera de Suspense | Boundary |
| `NEXT_REDIRECT` en un toast | `redirect` dentro del `try` | Afuera del try |
| Un productor edita productos de otro | `where` sin `producerId` | Filtro por dueño en la DAL |
| Mensaje "el email ya existe" | Enumeración de usuarios | Mensaje genérico |

---

## Tarea

1. Registro, ingreso, salida y `/account` con perfil y pedidos (de la DAL con cache por usuario).
2. Script `scripts/make-producer.ts <email> <producerSlug>` que pase un usuario a `producer` y le cree su `Producer`.
3. Todas las funciones de `producer-products.ts` con `requireProducer` y filtro por dueño. Test: un productor intenta cambiar el precio de un producto ajeno → `NOT_FOUND`.
4. Login con Google (plugin social de Better Auth) **opcional**.
5. Quitá todos los `instant = false` del proyecto sin que falle el build.

**Terminado cuando:** con dos productores, ninguno puede ver ni tocar productos o pedidos del otro (probado a mano con dos navegadores y con el test).
