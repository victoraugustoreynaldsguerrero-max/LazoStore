# lasoSTORE — arquitectura de producción

## 1. Tecnología elegida
**Next.js 14 (App Router) + Supabase** (Auth, Postgres con RLS, Storage).
Por qué: Next.js te da Server Components (leen la base de datos sin exponer
claves), Server Actions y Route Handlers (donde vive toda la lógica que no
debe correr en el navegador), y un `middleware.ts` que protege `/admin` antes
de que el HTML llegue al cliente. Supabase evita que tengas que escribir tu
propio servidor de autenticación, y su Row Level Security es la base de datos
misma verificando permisos, no solo tu código.

## 2. Crear la base de datos
1. Crea un proyecto en supabase.com.
2. Ve a **SQL Editor** → pega el contenido de `supabase/schema.sql` → Run.
   Esto crea las tablas, los triggers, las políticas RLS y la función
   `create_order` que descuenta stock de forma atómica.
3. Ve a **Storage** → crea un bucket llamado `products`, público para lectura.

## 3. Configurar Supabase
En **Project Settings → API** copia:
- `Project URL` → `NEXT_PUBLIC_SUPABASE_URL`
- `anon public key` → `NEXT_PUBLIC_SUPABASE_ANON_KEY`
- `service_role key` → `SUPABASE_SERVICE_ROLE_KEY` (solo servidor, nunca al frontend)

En **Authentication → URL Configuration**, agrega tu dominio (y
`http://localhost:3000` en desarrollo) como Redirect URL, apuntando a
`/auth/callback`.

## 4. Configurar Google OAuth
1. En Google Cloud Console crea credenciales OAuth 2.0 (tipo "Web application").
2. Authorized redirect URI: `https://TU-PROYECTO.supabase.co/auth/v1/callback`.
3. En Supabase: **Authentication → Providers → Google**, pega el Client ID y
   Client Secret que te dio Google, y actívalo.
   Supabase gestiona el intercambio OAuth; tu app nunca ve ni guarda la
   contraseña de Gmail del usuario.

## 5. Crear el primer administrador de forma segura
No hay usuarios demo. El primer admin se promueve manualmente, una sola vez:
1. Regístrate normalmente en la app con tu correo.
2. En Supabase, ve a **SQL Editor** y ejecuta:
   ```sql
   update profiles set role = 'admin' where email = 'tu-correo@gmail.com';
   ```
3. Cierra sesión y vuelve a entrar: ahora `/admin` te dejará pasar.
   Para futuros admins, hazlo desde el propio panel (`profiles: admin manages roles`).

## 6. Variables de entorno
Copia `.env.local.example` a `.env.local` y completa:
```
NEXT_PUBLIC_SUPABASE_URL=...       # pública
NEXT_PUBLIC_SUPABASE_ANON_KEY=...  # pública, protegida por RLS
SUPABASE_SERVICE_ROLE_KEY=...      # SECRETA, solo en el servidor (Vercel envs, no en el repo)
```
En Vercel: **Project Settings → Environment Variables**, agrega las tres.
`.env.local` ya está pensado para no subirse a git (agrégalo a `.gitignore`).

## 7. Desplegar
```bash
npm install
npm run dev      # local, http://localhost:3000
```
Producción: conecta el repo a **Vercel** (framework Next.js detectado
automáticamente), configura las env vars del paso 6, y despliega. No hay un
"backend aparte" que desplegar: los Route Handlers y Server Actions corren
como funciones serverless dentro del mismo despliegue de Next.js; la base de
datos vive en Supabase.

## 8. Reglas de seguridad que quedaron configuradas
- RLS activo en las 6 tablas; un cliente solo lee productos/categorías/banners
  **activos**, y solo ve **sus propios** pedidos y perfil.
- Un cliente no puede cambiar su propio `role` (la policy de `update` en
  `profiles` lo bloquea a nivel de base de datos, no solo de UI).
- `order_items` no tiene policy de INSERT para clientes: solo se escribe
  desde `create_order()`, que corre con los permisos del dueño de la función
  y revalida precio/stock reales antes de descontar.
- Las Server Actions de `/admin/*` usan el cliente autenticado por cookies
  (no la service key), así que aunque alguien manipule el frontend, RLS +
  `is_admin()` deciden en la base de datos si la operación se permite.
- Subida de imágenes valida tipo MIME, tamaño (4MB) y sanitiza el nombre
  de archivo antes de guardarlo en Storage.

## 9. Lo que falta para "producción completa" (siguiente iteración)
Para no entregarte una fachada, sé explícito sobre lo que este scaffold
todavía no incluye y que deberías añadir antes de lanzar con dinero real:
- **Pagos**: no hay pasarela conectada. Cuando la integres (Stripe, Culqi,
  MercadoPago...), el pedido debe pasar de `pending` a `paid` únicamente
  desde el webhook firmado del proveedor, nunca desde el navegador.
- **Rate limiting** en `/api/checkout` y en login (ej. con Upstash/Vercel).
- Página `/account/new-password` para completar el flujo de recuperación.
- Componentes visuales completos (Header, Sidebar, CartDrawer, Carrusel,
  Chat) — aquí están simplificados a su lógica funcional; el CSS de
  `globals.css` ya trae tu paleta (negro, morado, rosa, cyan) para que
  termines de portar el diseño del HTML original.

## 10. Cómo probar antes de producción
1. Crea un usuario de prueba, verifica el flujo de verificación de correo.
2. Intenta, con ese usuario (no admin), llamar `PATCH` a `/admin` por URL
   directa → debe redirigirte a `/`.
3. Desde la consola del navegador, intenta hacer
   `supabase.from('products').update({price:1}).eq('id', X)` como cliente
   no-admin → debe fallar por RLS.
4. Compra un producto con stock = 1 desde dos pestañas a la vez → solo un
   pedido debe completarse, el otro debe fallar por "stock insuficiente"
   (prueba de condición de carrera resuelta por `for update` en SQL).
5. Revisa en Supabase → Logs que no se esté imprimiendo ningún secreto.
