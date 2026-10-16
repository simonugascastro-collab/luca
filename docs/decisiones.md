# Decisiones de stack — LUCA

> **Fecha:** 16-10-2026. Versiones verificadas ese día en el registro de npm y en la documentación oficial de cada herramienta.
> **Regla de versiones:** fijamos la versión *mayor.menor* (por ejemplo, Next.js 16.4). Los parches (16.4.**x**) se actualizan con `npm update` en un PR propio, revisando que el CI quede en verde.

## Contexto que condicionó las decisiones

| Restricción | Consecuencia |
|---|---|
| Pocas horas a la semana (~5 h) | Menos herramientas, cada una bien aprendida. Nada que no luzca en la demo. |
| Plan Claude Pro | Una sesión de Claude Code a la vez casi siempre. Nada de revisión automática que gaste tu límite en cada PR. |
| Mac con 8 GB de RAM | Sin Docker local: Supabase corre en la nube también para desarrollo. |
| Costo cero (salvo la API de Claude) | Solo planes gratuitos. La IA usa Haiku 5.5: el proyecto completo debería costar menos de US$3. |
| Es una demo para reclutadores | Cualquiera debe poder probarla en 10 segundos, sin crear cuenta ni instalar nada. |

## Resumen

| # | Área | Elegido | Versión | Descartado |
|---|---|---|---|---|
| 1 | Runtime | Node.js LTS | 24.x (24.22) | Node 26 |
| 2 | Paquetes | npm | 11.x (viene con Node 24) | pnpm 12 |
| 3 | Framework | Next.js, App Router | 16.4.2 + React 19.3.1 | Expo, React Router 7 |
| 4 | Lenguaje | TypeScript estricto | 5.9.3 | TypeScript 7.0 |
| 5 | Validación | Zod | 4.6.x | Valibot |
| 6 | Estilos | Tailwind CSS | 4.3.x | CSS Modules |
| 7 | Componentes | shadcn/ui (base Radix) | CLI 4.22 | Mantine, HeroUI |
| 8 | Animaciones | Motion | 14.3.x | Solo CSS |
| 9 | Íconos | Lucide | lucide-react 1.58 | Heroicons, Phosphor |
| 10 | Gráficos | Recharts (componente Chart de shadcn) | 3.10.x | Tremor, Nivo, ECharts |
| 11 | Base de datos + login | Supabase (Postgres 17 + RLS) | supabase-js 2.118 · @supabase/ssr 0.12 · CLI 2.121 | Firebase, Neon + Clerk |
| 12 | Acceso a datos | Cliente de Supabase + tipos generados | — | Drizzle 0.45, Prisma |
| 13 | IA (librería) | SDK oficial de Anthropic | @anthropic-ai/sdk 0.135 | AI SDK 7 de Vercel |
| 14 | IA (modelo) | Claude Haiku 5.5 para todo | `claude-haiku-5-5` | Sonnet 5.5 / Opus 5.5 para el chat |
| 15 | WhatsApp | WhatsApp Cloud API (Meta) | Graph API | Twilio, librerías no oficiales |
| 16 | Bancos | Fintoc en modo test | SDK `fintoc` 1.27 + widget | Belvo, datos 100 % inventados |
| 17 | App instalable | PWA con manifest nativo de Next + service worker propio | — | Serwist 9.5, app nativa |
| 18 | Bloqueo biométrico | WebAuthn con SimpleWebAuthn | 14.0.x | Passkeys de Supabase (beta) |
| 19 | Notificaciones | In-app + Web Push (VAPID) | web-push 3.6.7 | Plantillas de WhatsApp |
| 20 | Tareas programadas | Vercel Cron, 1 vez al día | — | Supabase Cron, GitHub Actions cron |
| 21 | Lint + formato | Biome | 2.5.x | ESLint 10 + Prettier |
| 22 | Tests unitarios | Vitest | 5.0.x | Jest |
| 23 | Tests end-to-end | Playwright | 1.64.x | Cypress |
| 24 | Errores | Sentry (plan gratis) | @sentry/nextjs 11.x | Solo logs de Vercel |
| 25 | Analítica | Vercel Web Analytics | @vercel/analytics 2.0 | PostHog (queda como extra) |
| 26 | Deploy | Vercel Hobby, preview por PR | — | Netlify, Railway |
| 27 | Repo + CI | GitHub (repo público) + GitHub Actions | — | Solo checks de Vercel |
| 28 | Gestión de tareas | Linear (plan Free) + MCP oficial | — | GitHub Projects, Notion |
| 29 | Agentes en paralelo | Worktrees nativos de Claude Code, ocasional | — | Conductor (por ahora) |

---

## 1. Runtime — Node.js 24 LTS

- **Qué es:** el programa que ejecuta JavaScript fuera del navegador. Next.js corre encima de él. *LTS* significa versión con soporte largo: recibe parches de seguridad por años.
- **Para qué en LUCA:** correr Next.js en tu Mac y en Vercel.
- **Por qué:** es el LTS activo y la versión por defecto de Vercel, que hoy ofrece 24.x y 22.x. Tener el mismo Node en tu Mac y en el servidor evita errores que "solo pasan en producción".
- **Descartado:** Node 26, el que tienes instalado. Todavía no es LTS (lo será a fines de octubre de 2026) y Vercel aún no lo ofrece.
- **Cómo:** `nvm install 24`, un archivo `.nvmrc` con `24` y `"engines": { "node": "24.x" }` en `package.json`.

## 2. Gestor de paquetes — npm

- **Qué es:** el programa que descarga e instala librerías. Viene incluido con Node.
- **Por qué:** no hay que instalar nada extra, todas las guías lo usan y Vercel lo detecta solo.
- **Descartado:** pnpm 12. Es más rápido y ahorra disco cuando hay muchos worktrees, pero es otra herramienta más, su versión mayor 12 salió recién (26-08-2026) y su ventaja real aparece en monorepos, que no es nuestro caso.

## 3. Framework — Next.js 16.4 con App Router

- **Qué es:** un framework de React para hacer apps web completas (pantallas + servidor) en un solo proyecto. Ya lo usaste.
- **Por qué:** la versión 16.4.2 (15-10-2026) es estable, se despliega en Vercel sin configurar nada y trae herramientas para agentes. Por ejemplo, `AGENTS.md` le indica a Claude que lea la documentación exacta de tu versión, instalada en `node_modules/next/dist/docs/`.
- **Cómo lo usamos (5 piezas):**
  1. **Server Components (por defecto):** las páginas leen los datos de Supabase en el servidor y el celular recibe la pantalla lista. Las claves nunca llegan al navegador.
  2. **Client Components (`'use client'`):** solo donde hay interacción: gráficos, formularios, chat y pantalla de bloqueo.
  3. **Server Actions:** para todo lo que cambia datos (crear un gasto, editar un presupuesto). Son funciones del servidor que un formulario llama directo. Adentro siempre validamos con Zod y revisamos la sesión.
  4. **Route Handlers (`src/app/api/.../route.ts`):** solo para quien no es un formulario: los webhooks de WhatsApp y Fintoc, el cron y el chat (que responde en *streaming*, palabra por palabra).
  5. **`proxy.ts`** (en Next 16 reemplaza a `middleware.ts`): renueva la sesión de Supabase en cada visita.
- **Opciones activadas** (las trae `create-next-app` 16.4):
  - **Cache Components:** nada se guarda en caché a menos que lo pidas con `'use cache'`. En una app de finanzas es lo correcto: los datos de un usuario nunca se le muestran a otro por accidente. Solo cachearemos la landing pública.
  - **React Compiler** (estable desde la 1.0): optimiza los componentes solo, así no necesitas `useMemo` ni `useCallback`.
  - **`typedRoutes`:** TypeScript avisa si un `<Link>` apunta a una ruta que no existe.
- **Descartado:**
  - Expo / React Native: una app nativa necesita las tiendas o TestFlight, y la PWA cumple "instalable sin tiendas".
  - React Router 7: es buen framework, pero ya conoces Next y Vercel lo integra mejor.

## 4. Lenguaje — TypeScript 5.9 estricto

- **Qué es:** JavaScript con tipos. Te avisa los errores antes de ejecutar el código.
- **Por qué 5.9.3:** es lo que instala `create-next-app` 16.4 (`typescript@^5`) y lo que usa hoy la mayoría de las apps Next en producción. Además, todas las herramientas (VS Code, shadcn, el plugin de Next) lo soportan completo.
- **Configuración:** `strict: true` + `noUncheckedIndexedAccess: true`. Esta segunda opción te obliga a considerar que `lista[0]` puede no existir.
- **Descartado:**
  - TypeScript 7.0, el compilador nuevo escrito en Go: es unas 10 veces más rápido y salió el 08-07-2026. Next.js ya lo soporta, pero todavía no expone la API de JavaScript que usan varias herramientas, y el editor necesita una extensión aparte. Lo revisamos en enero: el cambio será un comando.
  - TypeScript 6.0: es una versión puente que no nos da nada extra.

## 5. Validación — Zod 4.6

- **Qué es:** una librería para describir la forma que deben tener los datos y verificarla mientras la app corre.
- **Para qué en LUCA:**
  - Entradas de formularios y Server Actions.
  - Variables de entorno: `src/lib/env.ts` hace que la app falle al arrancar si falta una clave, en vez de fallar a medio camino.
  - Webhooks.
  - Respuestas de la IA: el SDK de Anthropic convierte un esquema Zod en *structured output*, es decir, un JSON garantizado.
- **Descartado:** Valibot. Es más liviano, pero Zod es el estándar y se integra directo con el SDK de Anthropic.

## 6. Estilos — Tailwind CSS 4.3

- **Qué es:** estilos con clases cortas (`p-4`, `text-sm`) en vez de escribir CSS aparte.
- **Por qué:** shadcn/ui está hecho con Tailwind, y la versión 4 define el tema (colores, modo oscuro) con variables CSS en un solo archivo.
- **Descartado:** CSS Modules. Implica más código y no tiene un ecosistema de componentes listo.

## 7. Componentes — shadcn/ui (CLI 4, base Radix)

- **Qué es:** no es una librería que se instala. Copia el código de cada componente (botón, tarjeta, hoja inferior) a tu repo y tú pasas a ser el dueño de ese código.
- **Por qué:** es lo que usan muchas startups con Next. Se ve profesional y se puede personalizar a fondo para que LUCA no parezca "otra app genérica". Trae los toasts (Sonner), la hoja inferior tipo app (Drawer) y el componente de gráficos (Chart).
- **Base Radix:** es la opción por defecto del CLI 4. Base UI es la alternativa, pero hay más ejemplos con Radix.
- **Descartado:** Mantine y HeroUI. Son librerías cerradas: cuesta más darles identidad propia.

## 8. Animaciones — Motion 14

- **Qué es:** librería de animaciones para React, antes llamada "Framer Motion". Se importa desde `motion/react`.
- **Para qué en LUCA:**
  - El saldo que "cuenta" hacia arriba al cargar.
  - Barras de presupuesto que se llenan.
  - Transiciones y gestos (deslizar para borrar).
- **Nota:** la versión mayor 14 salió el 02-10-2026. Si da problemas, la 13.5 es la anterior estable. React 19.3 trae además *View Transitions* estables para animar el paso entre páginas.
- **Descartado:** usar solo CSS. Alcanza para lo básico, pero los gestos y las animaciones de números son mucho más fáciles con Motion.

## 9. Íconos — Lucide 1.58

- **Qué es:** íconos en SVG. Es el set por defecto de shadcn y solo se incluyen los que importas.
- **Descartado:** Heroicons y Phosphor. Son buenos, pero mezclar sets se ve desprolijo.

## 10. Gráficos — Recharts 3.10 (vía componente Chart de shadcn)

- **Qué es:** librería de gráficos para React basada en SVG.
- **Para qué en LUCA:**
  - Donut de gasto por categoría.
  - Barras por mes.
  - Línea de saldo.
  - Proyección de metas.
- **Por qué:** el componente `Chart` de shadcn usa Recharts v3 y toma los colores del tema (`--chart-1` a `--chart-5`), así los gráficos combinan con el resto de la app.
- **Descartado:**
  - Tremor: usa Recharts por debajo, así que sería una capa extra.
  - Nivo y visx: más potentes, pero con mucha más curva de aprendizaje.
  - ECharts: pesado para celular.

## 11. Base de datos y login — Supabase

- **Qué es:** Postgres administrado, con login, una API automática y almacenamiento de archivos.
- **Por qué:**
  - Ya lo tienes conectado.
  - El plan gratis alcanza: 2 proyectos activos, 500 MB por proyecto y 50.000 usuarios activos al mes.
  - Las políticas **RLS** (Row Level Security) de Postgres resuelven "cada usuario ve solo lo suyo" dentro de la base misma, no en el código.
- **Llaves:**
  - `sb_publishable_...`: va en el navegador y es segura gracias a RLS.
  - `sb_secret_...`: solo en el servidor. Salta RLS, así que solo la usan los webhooks y el cron.
- **Login:**
  - **"Probar demo"** usa *anonymous sign-in*: cada visitante recibe un usuario anónimo propio con 3 meses de datos generados. Nadie ve ni ensucia los datos de otro, y el cron borra estos usuarios después de 7 días. Supabase limita esto a 30 por hora por IP.
  - **Tu cuenta personal:** *magic link* (un enlace por correo). El correo que trae Supabase está pensado para pruebas y es muy limitado, por eso el público entra con "Probar demo".
  - La sesión vive en cookies (`@supabase/ssr`) y se verifica con `getClaims()`.
- **Dos proyectos:**
  - `luca-dev`: tu Mac y los previews.
  - `luca-prod`: la demo pública.
  - Tu proyecto pausado "Fintax" no cuenta como activo. No lo reactives o te quedas sin cupo.
- **Riesgo importante:** el plan gratis **pausa el proyecto después de 7 días sin actividad**. Si nadie visita la demo durante una semana de verano, un reclutador encontraría la app caída. Solución: el cron diario de Vercel hace una consulta y el proyecto nunca queda inactivo.
- **Sin Docker:** Supabase local necesita Docker y varios GB de RAM. Trabajamos contra `luca-dev` en la nube.
- **Migraciones:** archivos SQL en `supabase/migrations/`, creados con el CLI de Supabase y guardados en git.
- **Descartado:**
  - Firebase: es NoSQL, así que los reportes por mes y categoría cuestan mucho más, y la seguridad va en reglas aparte.
  - Neon + Clerk: son 2 servicios en vez de 1.

## 12. Acceso a datos — cliente de Supabase + tipos generados (sin ORM)

- **Qué es:** consultas del tipo `supabase.from('transactions').select(...)`, con tipos de TypeScript generados desde la base (`supabase gen types`).
- **Por qué:** el cliente viaja con el token del usuario, así que RLS se aplica siempre. Un ORM se conecta como "dueño" de la base y se salta RLS si no lo configuras con mucho cuidado.
- **Consultas complejas** (reportes): vistas SQL con `security_invoker = true` o funciones SQL llamadas con `rpc`.
- **Descartado:**
  - Drizzle 0.45: muy cómodo, pero duplica la definición del esquema y complica RLS.
  - Prisma: los mismos problemas y es más pesado.

## 13. IA (librería) — SDK oficial de Anthropic

- **Qué es:** `@anthropic-ai/sdk`, la librería oficial para llamar a Claude desde TypeScript.
- **Por qué:** da acceso directo a todas las funciones de Claude desde el primer día:
  - *Structured outputs*: respuesta JSON validada con Zod.
  - *Tool runner*: Claude llama a funciones tuyas.
  - *Prompt caching*: reutiliza la parte fija del prompt y la cobra más barata.
  - *Streaming*.
  - Es una sola capa que aprender.
- **Descartado:** AI SDK 7 de Vercel (`ai` + `@ai-sdk/anthropic`). Sirve para cambiar de proveedor y trae hooks de interfaz (`useChat`), pero agrega una abstracción que no necesitamos y a veces llega tarde a las funciones nuevas de Claude.

## 14. IA (modelos) — Claude Haiku 5.5 para todo (tu elección)

Haiku 5.5 (`claude-haiku-5-5`) cuesta US$0,10 por millón de tokens de entrada y US$0,50 por millón de salida.

| Tarea | Ajuste | Costo aproximado |
|---|---|---|
| Categorizar movimientos (lotes de 25) | `effort: "low"` + structured output | menos de US$0,001 por lote |
| Leer un mensaje de WhatsApp | `effort: "low"` + structured output | ~US$0,0002 por mensaje |
| Chat con tus finanzas | `effort: "medium"` (default) + herramientas + streaming | ~US$0,001 por pregunta |

- **Total esperado del proyecto completo:** menos de US$3.
- **Cambiar de modelo sin tocar código:** cada tarea lee su modelo de una variable de entorno (`AI_MODEL_CHAT`, `AI_MODEL_FAST`). Si el chat se queda corto en preguntas como "¿en qué puedo recortar?", subir a Sonnet 5.5 (`claude-sonnet-5-5`, ~US$0,02 por pregunta) es cambiar esa variable.
- **Protección de costos:**
  - Límite de gasto mensual en la consola de Claude.
  - Máximo 30 preguntas de chat por usuario al día, contadas en la tabla `ai_usage`.
- Cómo el chat accede a los datos de forma segura: ver [arquitectura.md](arquitectura.md#4-chat-con-tus-finanzas).

## 15. WhatsApp — Cloud API oficial de Meta

- **Qué es:** la API oficial de WhatsApp para empresas. Meta da un número de prueba gratis, y los mensajes llegan a tu webhook (`/api/whatsapp/webhook`).
- **Costo:** responder dentro de las 24 horas desde que el usuario te escribe es gratis. Meta solo cobra las *plantillas* enviadas fuera de esa ventana, y no las usamos.
- **Límite del número de prueba:** solo envía a 5 números verificados, y un reclutador no estará entre ellos. Por eso hacemos dos cosas:
  - Un **simulador de WhatsApp dentro de la app**, que usa exactamente el mismo procesamiento.
  - Un video corto con el WhatsApp real.
- **Política de Meta 2026:** prohíbe los asistentes de IA "de propósito general" en la API para empresas, pero un bot acotado a registrar gastos está permitido. Por eso el bot de LUCA solo habla de finanzas.
- **Tu agente existente:** pendiente que me confirmes qué proveedor usa (ver [Pendientes](#pendientes-que-dependen-de-ti)). Para no depender de eso, el resto de LUCA habla con una interfaz `WhatsAppProvider` (verificar webhook, leer mensaje, enviar texto).
- **Descartado:**
  - Twilio: cobra por mensaje y su sandbox obliga al usuario a enviar primero un "join ...".
  - Librerías no oficiales (Baileys y similares): necesitan un servidor encendido 24/7, así que no corren en Vercel. Además violan los términos de WhatsApp: riesgo de que bloqueen el número.

## 16. Bancos — Fintoc en modo test

- **Qué es:** una API chilena que conecta cuentas bancarias y entrega saldos y movimientos. En modo test usa bancos simulados con credenciales de prueba públicas: nunca hay una cuenta real.
- **Bancos del sandbox:** Banco de Chile, Santander, Itaú, BICE, Scotiabank, BCI, BancoEstado y Security.
- **Flujo** (detalle en [arquitectura.md](arquitectura.md#3-conectar-un-banco-con-fintoc)):
  1. El servidor crea un *Link Intent*.
  2. El navegador abre el widget de Fintoc.
  3. Al terminar, el widget entrega un `exchange_token`.
  4. El servidor lo canjea por el `link_token`.
  5. Con ese token leemos cuentas y movimientos.
- **Librerías:** SDK `fintoc` 1.27 en el servidor; el widget se carga con el script oficial de Fintoc en el navegador.
- **Sincronización:** botón "Sincronizar" + el cron diario. Los *Refresh Intents* con webhook (`account.refresh_intent.succeeded`) quedan como extra.
- **Seguridad:** el `link_token` equivale a las credenciales del banco. Vive solo en el servidor, en un esquema `private` que la API pública no expone.
- **Descartado:**
  - Belvo: más orientado a México y Brasil.
  - Inventar todos los datos: no demuestra una integración real, que es justamente lo que tiene Not Pato y lo que valora un reclutador.

## 17. App instalable — PWA

- **Qué es:** una web que se instala en el celular como app: ícono propio, pantalla completa y notificaciones.
- **Cómo:** sigue la guía oficial de PWA de Next.js:
  - `src/app/manifest.ts`, nativo de Next.js.
  - Íconos de la app.
  - Un `public/sw.js` escrito a mano, con push y una página offline simple.
- **Instalación:**
  - Android/Chrome: botón "Instalar" (evento `beforeinstallprompt`).
  - iPhone: Safari → Compartir → "Agregar a inicio". Mostramos las instrucciones en la app.
- **Descartado:**
  - Serwist 9.5: hace precarga offline completa, pero exige más configuración y da advertencias con Turbopack en desarrollo, y guardar datos financieros offline no aporta a la demo.
  - App nativa: necesita las tiendas.

## 18. Bloqueo biométrico — WebAuthn con SimpleWebAuthn 14

- **Qué es:** WebAuthn es el estándar del navegador para usar Face ID o la huella. SimpleWebAuthn (`@simplewebauthn/browser` + `@simplewebauthn/server`) resuelve la parte criptográfica.
- **Cómo funciona:**
  1. En Ajustes, "Activar bloqueo" registra una llave del dispositivo. Tu huella nunca sale del celular: solo guardamos una llave pública.
  2. Al abrir la app, o después de 5 minutos sin usarla, aparece la pantalla de bloqueo y pide Face ID o huella.
  3. El servidor verifica la firma y entrega una cookie "desbloqueado" que dura 15 minutos.
- **Ojo:** es un **candado encima de la sesión**, no reemplaza el login.
- **Descartado:** las passkeys de Supabase Auth. Están en beta desde mayo de 2026 y son un método de *login*, no un candado de app.

## 19. Notificaciones — in-app + Web Push

- **In-app:** tabla `notifications` + una campanita. Funciona siempre y sin instalar nada, que es lo que verá un reclutador.
- **Web Push:** el estándar del navegador, con llaves VAPID y la librería `web-push` 3.6.7. Es antigua, pero sigue siendo la referencia estable porque el protocolo no cambió. En iPhone solo funciona si la PWA está agregada a inicio (iOS 16.4 o posterior).
- **Cuándo avisamos:**
  - Presupuesto al 80 % y al 100 %.
  - Suscripción que subió de precio.
  - Meta alcanzada.
  - Resumen semanal.
- **Descartado:** avisos proactivos por WhatsApp. Requieren plantillas aprobadas y se cobran fuera de la ventana de 24 horas. WhatsApp sí responde las confirmaciones ("Listo: $3.500 en Comida").

## 20. Tareas programadas — Vercel Cron, 1 vez al día

- **Qué es:** Vercel llama a una URL tuya a una hora fija.
- **Límite del plan Hobby:** 1 ejecución al día por job, con hasta ±59 minutos de diferencia.
- **Un solo job diario**, `/api/cron/diario`, a las 11:00 UTC (08:00 en Chile en horario de verano). Hace 6 cosas:
  1. Mantiene viva la base de datos (evita la pausa de 7 días).
  2. Sincroniza los bancos de Fintoc.
  3. Detecta suscripciones y subidas de precio.
  4. Revisa las metas de ahorro.
  5. Los lunes, envía el resumen semanal.
  6. Borra los usuarios demo con más de 7 días.
- **Seguridad:** está protegido con `CRON_SECRET`, que Vercel envía en el header `Authorization`.
- **Descartado:**
  - Supabase Cron: permite correr cada minuto, pero serían 2 lugares donde programar tareas, y no necesitamos más de una vez al día.
  - Cron de GitHub Actions: GitHub lo desactiva solo después de 60 días sin actividad en el repo, justo lo que pasaría con la demo en verano.

## 21. Lint y formato — Biome 2.5

- **Qué es:** una sola herramienta que revisa errores comunes (*lint*) y ordena el formato del código.
- **Por qué:**
  - `create-next-app` la ofrece como opción oficial (`--biome`).
  - Es un solo archivo de configuración y es muy rápida.
  - No depende de la API de TypeScript.
- **Descartado:** ESLint 10 + Prettier. Son 2 herramientas con varios plugins, y Next 16 ya no trae `next lint`.

## 22. Tests unitarios — Vitest 5

- **Qué es:** corre tests rápidos de funciones sueltas. Es estable desde el 03-09-2026.
- **Qué testeamos:**
  - Formato CLP y fechas.
  - Proyección de metas.
  - Detección de suscripciones.
  - Lectura de mensajes de WhatsApp, con la IA simulada.
  - Alertas de presupuesto.
  - Un test de RLS contra `luca-dev`: el usuario A no puede leer los datos de B.
- **Descartado:** Jest. Es más lento y necesita configuración extra para TypeScript.

## 23. Tests end-to-end — Playwright 1.64

- **Qué es:** abre un navegador real y hace clics como si fuera un usuario.
- **Solo 3 flujos críticos:**
  1. "Probar demo" → el panel muestra datos.
  2. Agregar un gasto → aparece en la lista.
  3. Preguntar en el chat, con la IA simulada (`AI_MOCK=1`).
- **Descartado:** Cypress. Es más lento y soporta peor Safari/WebKit, que importa porque tu público usa iPhone.

## 24. Errores — Sentry

- **Qué es:** te avisa cuando la app falla en producción y muestra el error y dónde ocurrió.
- **Plan Developer gratis:** 5.000 errores al mes, 1 usuario.
- **¿Vale la pena para una demo?** Sí: el asistente (`npx @sentry/wizard@latest -i nextjs`) se configura en unos 30 minutos y demuestra prácticas de producción.
- **Descartado:** depender solo de los logs de Vercel. En Hobby duran poco y no te avisan.

## 25. Analítica — Vercel Web Analytics

- **Qué es:** cuenta visitas y páginas vistas, sin cookies. Se activa con un solo componente.
- **Por qué:** es gratis en Hobby y no requiere configuración.
- **Descartado por ahora:** PostHog. Es más potente: graba sesiones, así verías cómo un reclutador recorre la demo, y su plan gratis da 1 millón de eventos. Pero requiere más configuración y un aviso de privacidad. Queda como extra de la semana 8.

## 26. Deploy — Vercel Hobby

- **Flujo:** cada PR genera un *preview* automático con su propia URL; lo que entra a `main` va a producción.
- **Variables de entorno por ambiente:**
  - Development y Preview → `luca-dev`.
  - Production → `luca-prod`.
- **Uso permitido:** Hobby es para uso personal no comercial. Una demo de portafolio califica.
- **Ojo:** revisar *Deployment Protection*. La URL de producción debe abrirse sin pedir login de Vercel (pruébalo en incógnito). Tu proyecto `whatsapp-saas` tiene esa protección activada; en LUCA la producción tiene que quedar pública.
- **Descartado:** Netlify y Railway. Ya usas Vercel, y es la casa de Next.js.

## 27. Repo y CI — GitHub + GitHub Actions

- **Repo público:** es tu portafolio, y en repos públicos los minutos de GitHub Actions son gratis.
- **CI en cada PR:** `biome ci` + `tsc --noEmit` + `vitest run`. El build lo hace Vercel en el preview.
- **Rama `main` protegida:** solo se entra por PR y con el CI en verde.
- **Secret scanning con push protection:** gratis en repos públicos. GitHub bloquea el push si detecta una clave.
- **Revisión automática con Claude en GitHub** (`/install-github-app`): funciona con tu plan Pro, pero gasta tu límite en cada PR. Es mejor correr `/code-review` en tu Mac antes de abrir el PR, porque así tú decides cuándo.

## 28. Gestión — Linear

- **Plan Free:** 250 issues, 2 equipos y miembros ilimitados. El plan de LUCA tiene 47 issues, así que sobra espacio.
- **Conexión con Claude Code:** MCP oficial en `https://mcp.linear.app/mcp`. Paso a paso en [guia-configuracion.md](guia-configuracion.md).
- **Descartado:**
  - GitHub Projects: es gratis e integrado, pero Linear es lo que usan las startups y lo pediste aprender.
  - Notion: no es un gestor de issues.

## 29. Paralelismo — worktrees nativos, ocasional (Conductor: no por ahora)

- **Por qué no Conductor ahora:**
  - Con plan Pro, cada agente en paralelo consume tu límite al doble.
  - Con ~5 horas a la semana, el cuello de botella eres tú revisando, no Claude escribiendo.
  - Para aprender, conviene ver una cosa a la vez.
- **Cuándo sí:**
  - Tareas chicas e independientes (README, tests, datos demo): ahí usa `claude --worktree <nombre>`.
  - Conductor tendría sentido si pasas a Max, o en la semana 8.
- Detalle en [guia-configuracion.md](guia-configuracion.md#4-trabajar-en-paralelo-worktrees-y-conductor).

---

## Pendientes que dependen de ti

1. **Proveedor de tu agente de WhatsApp** (`whatsapp-saas`): ¿API oficial de Meta, Twilio o librería no oficial? Para responder, abre su código y busca `graph.facebook.com` (Meta), `twilio` o `baileys` / `whatsapp-web.js`.
   - **Si es Meta:** reutilizamos tu app de Meta, el número y el código de verificación del webhook. Ahorra ~1 hora en la semana 6.
   - **Si es otro:** solo reutilizamos la lógica y los prompts; la conexión se hace nueva con Meta.
2. **URL de la demo:** resuelto el 16-10-2026 → **https://luca-finanzas.vercel.app** (gratis).
3. **Repo público desde el día 1** (recomendado). La condición es que nunca subas un `.env.local`: el `.gitignore` y la push protection de GitHub te cuidan.

## Fuentes principales

- Next.js 16.4: [nextjs.org/blog/next-16-4](https://nextjs.org/blog/next-16-4) · TypeScript 7 en Next: [nextjs.org/docs/app/api-reference/config/typescript](https://nextjs.org/docs/app/api-reference/config/typescript) · Agentes: [nextjs.org/docs/app/guides/ai-agents](https://nextjs.org/docs/app/guides/ai-agents)
- Vercel: [Node.js soportado](https://vercel.com/docs/functions/runtimes/node-js/node-js-versions) · [Límites de Cron](https://vercel.com/docs/cron-jobs/usage-and-pricing)
- Supabase: [Precios y límites](https://supabase.com/pricing) · [Auth con Next.js](https://supabase.com/docs/guides/auth/server-side/nextjs) · [Usuarios anónimos](https://supabase.com/docs/guides/auth/auth-anonymous) · [Passkeys beta](https://supabase.com/changelog/46458-passkeys-for-supabase-auth-beta)
- Fintoc: [Modo test](https://docs.fintoc.com/docs/test-mode) · [Flujo con exchange token](https://docs.fintoc.com/guides/movements/integration-with-exchange-token.md) · [Refresh intents](https://docs.fintoc.com/guides/movements/refresh-intents-walkthrough/refresh-intents-webhook.md) · [Bancos soportados](https://docs.fintoc.com/guides/movements/overview-data-aggregation/products-and-institutions-movements.md)
- WhatsApp: [Precios de Meta](https://developers.facebook.com/docs/whatsapp/pricing) · [Primeros pasos Cloud API](https://developers.facebook.com/docs/whatsapp/cloud-api/get-started/) · [Política de chatbots 2026](https://respond.io/blog/whatsapp-general-purpose-chatbots-ban)
- Claude Code: [Worktrees](https://code.claude.com/docs/en/worktrees) · [GitHub Actions](https://code.claude.com/docs/en/github-actions) · Linear MCP: [linear.app/docs/mcp](https://linear.app/docs/mcp)
- Otros: [shadcn Chart](https://ui.shadcn.com/docs/components/chart) · [Linear precios](https://linear.app/pricing) · [Conductor](https://www.conductor.build) · [Web Push en iOS](https://webkit.org/?p=16535)
