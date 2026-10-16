# Plan de 8 semanas — LUCA

> **Supuestos:** ~5 horas a la semana (≈40 h en total). Claude Code escribe la mayor parte del código; tú decides, revisas, pruebas en el celular y aprendes.
> **Lanzamiento:** lunes 14-12-2026.
> Cada issue está pensado para durar entre 30 y 90 minutos y dejar algo que se pueda ver o probar. El formato está listo para crearlos en Linear (ver [guia-configuracion.md](guia-configuracion.md#2-linear-paso-a-paso)).

## Calendario

| Semana | Fechas | Fase | Lo que se puede mostrar al terminar | Horas |
|---|---|---|---|---|
| S1 | 19–25 oct | Cimientos | App vacía en producción; Linear, CI y CLAUDE.md listos | 4,5 |
| S2 | 26 oct–1 nov | Datos y acceso | "Probar demo" crea un usuario con 3 meses de datos | 5 |
| S3 | 2–8 nov | Panel y movimientos | Panel con saldo, gráficos y lista de movimientos | 5,5 |
| S4 | 9–15 nov | Presupuestos + PWA | Presupuestos con alertas; instalable en el celular | 4,5 |
| S5 | 16–22 nov | IA | Chat con tus finanzas + categorización automática | 5,5 |
| S6 | 23–29 nov | Integraciones | Banco de prueba conectado (Fintoc) + gasto por WhatsApp | 5,5 |
| S7 | 30 nov–6 dic | Diferenciadores | Suscripciones, metas, push y biometría | 5 |
| S8 | 7–13 dic | Lanzamiento | Landing, README, video y v1.0 | 5 |

**Por qué este orden:**

1. Primero lo que todo lo demás necesita: repo, base de datos, login y datos demo.
2. Después lo que más se ve en una demo: panel, gráficos y movimientos.
3. La IA va cuando ya hay datos con qué conversar.
4. Las integraciones van a mitad de camino. Son lo más incierto, y así queda margen si algo se complica.
5. Al final, el pulido y la presentación.

## Prioridades y recorte

- **Alta:** sin esto no hay demo. **Media:** diferenciador. **Baja:** recortable.
- **Si vas atrasado,** no intentes recuperar todo de golpe. Recorta en este orden:
  1. LUC-40 bloqueo biométrico.
  2. LUC-41 resumen semanal.
  3. LUC-39 Web Push (queda la campanita in-app).
  4. LUC-31 mini-evaluación.
  5. LUC-44 E2E (deja solo el flujo "Probar demo").
  6. LUC-21 categorías personalizadas.
- **Nunca se recorta:** "Probar demo", panel, chat con IA, WhatsApp (al menos el simulador), Fintoc, PWA y README.

> Para igualar a Not Pato todo importa. Pero si hay que elegir, lo recortable es lo que menos se nota en una demo de 2 minutos.

## Rutina

- **Lunes (15 min):** abre el ciclo de la semana en Linear y pídele a Claude: *"resume el ciclo actual y propón el orden"*.
- **Cada sesión (1–2 h):** `/feature LUC-xx`. Ese comando hace todo el ciclo issue → rama → plan → código → checks → PR (ver la guía).
- **Domingo (10 min):** revisa el ciclo. Lo que no terminaste pasa solo al siguiente. Si algo cambió, actualiza este archivo.

## Etiquetas

`feature` · `diseño` · `ia` · `integración` · `db` · `infra` · `seguridad` · `calidad` · `docs` · `chore`

**Estimaciones en Linear:** escala lineal, donde 1 punto = 30 minutos.

---

## S1 · Cimientos (19–25 oct)

### LUC-1 · Preparar el Mac: Node 24, GitHub CLI y Supabase CLI
`chore` · Alta · 0,5 h
- **Qué:** `nvm install 24 && nvm alias default 24` · `brew install gh && gh auth login` · `brew upgrade supabase`.
- **Listo cuando:** `node -v` → v24.x · `gh auth status` muestra tu usuario · `supabase --version` ≥ 2.121.

### LUC-2 · Crear el proyecto Next.js 16.4 y el repo en GitHub
`infra` · Alta · 1 h
- **Qué:** sigue estos pasos en orden.
  ```bash
  # 1) Cierra VS Code. La carpeta debe estar en minúsculas: npm no acepta "LUCA" como nombre.
  cd ~/Desktop && mv LUCA luca-tmp && mv luca-tmp luca && cd luca

  # 2) create-next-app no acepta un CLAUDE.md en la carpeta: lo apartamos un momento.
  mv CLAUDE.md docs/CLAUDE.md.tmp
  npx create-next-app@16.4.2 . --ts --tailwind --biome --app --src-dir \
    --react-compiler --cache-components --import-alias "@/*" \
    --use-npm --agents-md --no-agent-feedback --yes
  mv docs/CLAUDE.md.tmp CLAUDE.md

  # 3) El template trae algunas versiones atrasadas: las subimos.
  npm install react@19.3 react-dom@19.3
  npm install -D @types/node@24 @biomejs/biome@2.5
  npx biome migrate --write
  echo "24" > .nvmrc

  # 4) Commit y repo público.
  git add -A && git commit -m "chore: agrega CLAUDE.md y actualiza versiones base"
  gh repo create luca --public --source=. --push
  code .
  ```
- **Después (con Claude):**
  - Agregar `"engines": { "node": "24.x" }` en `package.json`.
  - Agregar `noUncheckedIndexedAccess: true` en `tsconfig.json`.
  - Agregar `typedRoutes: true` en `next.config.ts`.
- **Listo cuando:** `npm run dev` abre `localhost:3000` · el repo `luca` existe en GitHub, es público e incluye `CLAUDE.md` y `docs/`.

### LUC-3 · Conectar Vercel y hacer el primer deploy
`infra` · Alta · 0,5 h
- **Qué:**
  1. vercel.com → *Add New → Project* → importa `luca` desde GitHub.
  2. Ponle `luca-demo` al proyecto: ese nombre define la URL.
  3. *Settings → Deployment Protection:* la producción debe quedar pública.
  4. *Settings → Functions:* región São Paulo (`gru1`), si tu plan lo permite, para quedar cerca de Supabase.
- **Listo cuando:** la URL de producción abre en una ventana de incógnito sin pedir login · un PR de prueba genera un preview.

### LUC-4 · Configurar Claude Code para el proyecto
`chore` · Alta · 0,5 h
- **Qué:**
  - `.claude/settings.json`: permisos para que Claude no pueda leer ni editar `.env.local`, y un hook que formatea con Biome cada archivo editado.
  - Skills: Supabase, `next-dev-loop` y la propia `/feature`.
  - `.worktreeinclude`.
  - `.gitignore`: agregar `.claude/worktrees/` y `.claude/settings.local.json`.
  - Pasos exactos en la [guía §3](guia-configuracion.md#3-claude-code-para-este-proyecto).
- **Listo cuando:** Claude responde "permiso denegado" si intenta leer `.env.local` · un archivo editado queda formateado solo · `/feature` aparece al escribir `/`.

### LUC-5 · Linear: espacio de trabajo, ciclos, GitHub y MCP + importar este plan
`chore` · Alta · 1 h
- **Qué:** [guía §2](guia-configuracion.md#2-linear-paso-a-paso).
- **Listo cuando:**
  - Los 47 issues están en Linear con etiqueta, prioridad, estimación y ciclo (LUC-1 a LUC-5 ya en *Done*).
  - `/mcp` muestra `linear-server` conectado.
  - Un PR con "Fixes LUC-6" mueve ese issue solo.

### LUC-6 · CI con GitHub Actions + proteger `main`
`infra` · `calidad` · Alta · 0,5 h
- **Qué:**
  - Instalar Vitest 5.
  - Scripts en `package.json`: `typecheck` (`tsc --noEmit`) y `test` (`vitest run`).
  - `.github/workflows/ci.yml`: `npm ci` → `npx biome ci` → `npm run typecheck` → `npm test`.
  - En GitHub, *Settings → Rules*: regla para `main` que exige PR + CI en verde.
- **Listo cuando:** un PR con un error de formato queda en rojo y no se puede mergear.

### LUC-7 · Variables de entorno validadas con Zod + `.env.example`
`infra` · `seguridad` · Alta · 0,5 h
- **Qué:**
  - `src/lib/env.ts` con Zod 4, separando variables de servidor y públicas.
  - `.env.example` con todos los nombres de la [tabla de arquitectura](arquitectura.md#variables-de-entorno).
  - El paquete `server-only` en los módulos con secretos.
- **Listo cuando:** si falta una variable, `npm run dev` falla con un mensaje claro en español.

## S2 · Datos y acceso (26 oct–1 nov)

### LUC-8 · Crear los proyectos de Supabase `luca-dev` y `luca-prod`
`infra` · `db` · Alta · 0,5 h
- **Qué:**
  1. Crea 2 proyectos en tu organización, en la región São Paulo (`sa-east-1`), la más cercana a Chile. Guarda las contraseñas de la base en un gestor de contraseñas.
  2. Copia la URL y las llaves *publishable* y *secret*:
     - las de `luca-dev` → `.env.local` y Vercel (*Development* y *Preview*);
     - las de `luca-prod` → Vercel (*Production*).
  3. Corre `npx supabase init` y `npx supabase link --project-ref <ref-de-dev>`.
- **Listo cuando:** `npx supabase projects list` muestra ambos proyectos · Vercel tiene las variables separadas por ambiente.

### LUC-9 · Migración `core`: perfiles, categorías, cuentas y movimientos con RLS
`db` · `seguridad` · Alta · 1 h
- **Qué:** tablas del bloque "Núcleo" de [esquema-db.md](esquema-db.md), más el trigger con las 16 categorías por defecto, la vista `monthly_category_spend` y RLS. Comandos:
  1. `npx supabase migration new core`
  2. Escribir el SQL.
  3. `npx supabase db push` (aplica a `luca-dev`).
  4. Advisors de Supabase.
- **Listo cuando:** los advisors no muestran alertas de seguridad · en el dashboard, cada tabla tiene RLS *enabled*.

### LUC-10 · Clientes de Supabase, `proxy.ts` y tipos generados
`infra` · Alta · 0,5 h
- **Qué:** instalar `@supabase/supabase-js` 2.118 y `@supabase/ssr` 0.12. Crear:
  - `src/lib/supabase/{browser,server,admin}.ts`;
  - `src/proxy.ts` (renueva la sesión y protege el grupo `(app)`);
  - el script `npm run db:types`.
- **Listo cuando:** sin sesión, `/inicio` redirige a `/ingresar` · existe `src/types/database.ts`.

### LUC-11 · Ingreso: "Probar demo" (anónimo), magic link y cerrar sesión
`feature` · `seguridad` · Alta · 1 h
- **Qué:**
  - Activar *Anonymous Sign-Ins* en ambos proyectos.
  - Página `/ingresar`.
  - Server Action `startDemo()` que crea el usuario anónimo y genera sus datos.
  - Magic link para tu cuenta personal.
- **Listo cuando:** desde incógnito, "Probar demo" entra a `/inicio` en menos de 3 s · "Cerrar sesión" vuelve a la landing.

### LUC-12 · Generador de datos demo (3 meses creíbles)
`feature` · Alta · 1 h
- **Qué:** `src/features/demo/generate.ts`, una función pura que usa números pseudoaleatorios sembrados con el `userId`. Genera:
  - Movimientos fijos: sueldo de fin de mes, arriendo, cuentas básicas.
  - Movimientos frecuentes: supermercado semanal (Lider, Jumbo, Unimarc), delivery, Uber, Red/Bip!, bencina.
  - 5 suscripciones; una sube de precio el tercer mes.
  - 2 metas de ahorro.
  - 4 presupuestos, uno de ellos al ~85 % este mes.
  - Todo se inserta en un solo lote.
- **Listo cuando:** un test verifica que todo monto es entero y distinto de 0, que las fechas están en rango y que los gastos son menores que los ingresos.

### LUC-13 · Formato chileno: `formatCLP`, `formatDate` y zona horaria + tests
`feature` · `calidad` · Alta · 0,5 h
- **Qué:** `src/lib/format/`.
  - **Dinero:** `Intl.NumberFormat('es-CL', { style: 'currency', currency: 'CLP' })` da `$3.500`, pero en negativos da `$-1.234`. El signo se pone a mano para obtener `-$1.234`.
  - **Fechas:** con `timeZone: 'America/Santiago'` dan `16-10-2026`.
  - **`todayInChile()`:** la fecha de hoy en Chile.
- **Listo cuando:** los tests cubren 0, negativos, millones y el cambio de día a medianoche UTC.

### LUC-14 · Test de RLS: el usuario A no puede ver ni tocar los datos de B
`seguridad` · `calidad` · Alta · 0,5 h
- **Qué:** `tests/integration/rls.test.ts` contra `luca-dev`. Crea usuarios temporales con la llave secreta y los borra al final.
- **Listo cuando:** pasan los 3 casos de [esquema-db.md](esquema-db.md#cómo-verificamos-que-funciona).

## S3 · Panel y movimientos (2–8 nov)

### LUC-15 · Sistema de diseño: colores, tipografía, modo oscuro y shadcn
`diseño` · Alta · 1 h
- **Qué:**
  - `npx shadcn@latest init` (base Radix).
  - Paleta propia, no el gris por defecto: un verde "luca" para ingresos, coral para gastos y neutros cálidos.
  - Tipografía Geist con `tabular-nums` en los montos.
  - Tokens `--chart-1` a `--chart-5`.
  - Componentes base: button, card, input, drawer, sonner, skeleton.
  - Una página `/dev/ui` con muestras, solo en desarrollo.
  - Claude usa la skill `frontend-design` para proponer la dirección visual.
- **Listo cuando:** modo claro y oscuro se ven bien en tu celular real · el contraste cumple el estándar AA.

### LUC-16 · Layout de la app: tab bar inferior, header y safe areas
`diseño` · Alta · 0,5 h
- **Qué:** el grupo `(app)` lleva:
  - Barra inferior: Inicio, Movimientos, Chat, Presupuestos, Más.
  - Header con la campanita.
  - Espacio para el notch (`env(safe-area-inset-*)`).
- **Listo cuando:** en iPhone, ni la barra ni el notch tapan contenido.

### LUC-17 · Panel de inicio: saldo, mes actual y últimos movimientos
`feature` · Alta · 1 h
- **Qué:** Server Component con:
  - Tarjeta de saldo total con animación de conteo.
  - Gastos vs. ingresos del mes.
  - Top 3 de categorías.
  - Últimos 5 movimientos.
  - *Skeletons* con `Suspense` mientras carga.
- **Listo cuando:** con los datos demo, carga en menos de 1 s y se ve como una app real.

### LUC-18 · Gráficos: gasto por categoría y por mes
`feature` · `diseño` · Alta · 1 h
- **Qué:** con el componente Chart de shadcn (Recharts 3):
  - Donut del mes por categoría.
  - Barras de los últimos 3 meses.
  - Tooltips con `formatCLP`.
  - Claude usa la skill `dataviz`.
- **Listo cuando:** tocar una categoría lleva a la lista de movimientos filtrada.

### LUC-19 · Lista de movimientos con filtros
`feature` · Alta · 1 h
- **Qué:** `/movimientos` con:
  - Movimientos agrupados por día ("Hoy", "Ayer", "lun 12-10").
  - Filtros por mes, categoría y texto, guardados en la URL.
  - Paginación simple.
- **Listo cuando:** buscar "uber" muestra solo esos movimientos y su total.

### LUC-20 · Agregar, editar y borrar un movimiento
`feature` · Alta · 1 h
- **Qué:**
  - Botón "+" que abre un Drawer con monto (teclado numérico, formatea "3.500" mientras escribes), categoría (grilla de emojis), fecha y nota.
  - Server Actions validadas con Zod.
  - Deslizar para borrar, con "Deshacer" en un toast.
- **Listo cuando:** agregar un gasto actualiza el panel y la lista sin recargar la página.

## S4 · Presupuestos y PWA (9–15 nov)

### LUC-21 · Categorías personalizadas
`feature` · Media · 0,5 h
- **Qué:** crear, renombrar, cambiar emoji o color, y archivar categorías. Se archivan en vez de borrarse.
- **Listo cuando:** una categoría archivada desaparece de los selectores, pero su historial sigue intacto.

### LUC-22 · Presupuestos mensuales con barra de progreso
`feature` · `db` · Alta · 1 h
- **Qué:**
  - Migración `budgets_notifications`.
  - `/presupuestos` con barras animadas: verde bajo 80 %, ámbar entre 80 % y 99 %, roja desde 100 %.
  - Texto "Te quedan $X para N días".
- **Listo cuando:** agregar un gasto mueve la barra que corresponde.

### LUC-23 · Notificaciones in-app + alertas al 80 % y 100 %
`feature` · Alta · 1 h
- **Qué:**
  - Campanita con contador.
  - `checkBudgetThresholds(userId, categoryId)`, que se llama después de cada movimiento nuevo (manual, WhatsApp o Fintoc).
  - `dedupe_key` para no repetir avisos.
- **Listo cuando:** cruzar el 80 % crea exactamente una notificación, aunque después agregues más gastos.

### LUC-24 · PWA instalable
`feature` · Alta · 1 h
- **Qué:**
  - `src/app/manifest.ts`: nombre, colores, `display: "standalone"`.
  - Íconos 192/512, versión *maskable* y `apple-touch-icon`.
  - `public/sw.js` mínimo, con página offline.
  - Banner "Instalar LUCA" en Android e instrucciones para iPhone.
- **Listo cuando:** se instala en Android y en iPhone y abre en pantalla completa.

### LUC-25 · Revisión en celular real y pulido de S1–S4
`diseño` · `calidad` · Alta · 1 h
- **Qué:** recorrer toda la app en tu celular, anotar los detalles y corregirlos con Claude. Correr Lighthouse en modo móvil.
- **Listo cuando:** Lighthouse móvil da 90 o más en Performance y en Accesibilidad para `/inicio`.

## S5 · IA (16–22 nov)

### LUC-26 · Consola de Claude, límite de gasto y cliente de IA
`ia` · `infra` · Alta · 0,5 h
- **Qué:**
  1. Crea la cuenta en platform.claude.com, carga el mínimo de créditos y fija un límite mensual de US$5.
  2. Pon la API key en `.env.local` y en Vercel.
  3. Migración `ai`.
  4. `src/lib/ai/client.ts` (*server-only*): lee los modelos de las variables de entorno, tiene modo `AI_MOCK=1` y registra cada uso en `ai_usage`.
  - Antes de escribir código, Claude carga la skill `claude-api`.
- **Listo cuando:** un script de prueba recibe respuesta de Haiku 5.5 y queda registrado en `ai_usage`.

### LUC-27 · Categorización automática: reglas + diccionario + Haiku 5.5
`ia` · Alta · 1,5 h
- **Qué:**
  - `normalizeMerchant()`.
  - Diccionario de comercios chilenos.
  - `categorizeBatch()`, con structured output (Zod) y `effort: "low"`.
  - Corre cada vez que entran movimientos sin categoría.
- **Listo cuando:** "COMPRA LIDER 4421" queda en Supermercado sin llamar a la IA · un comercio desconocido recibe categoría de la IA, con su nivel de confianza guardado.

### LUC-28 · Corregir una categoría y aprender la regla
`ia` · `feature` · Alta · 0,5 h
- **Qué:** al cambiar la categoría de un movimiento, la app pregunta "¿Aplicar a todos los de Lider?". Si dices que sí, crea la regla en `merchant_rules` y recategoriza los anteriores.
- **Listo cuando:** la próxima vez, ese comercio se categoriza solo (`categorized_by = rule`).

### LUC-29 · Chat: endpoint con herramientas, streaming y límite diario
`ia` · `seguridad` · Alta · 1,5 h
- **Qué:** `/api/chat` con:
  - Las 6 herramientas de [arquitectura.md](arquitectura.md#4-chat-con-tus-finanzas), usando el tool runner del SDK y Zod.
  - Streaming y prompt caching.
  - Máximo 5 herramientas por pregunta y 30 preguntas al día.
- **Listo cuando:** responde bien "¿cuánto gasté en Uber este mes?", "¿en qué puedo recortar?" y "¿cuál es mi suscripción más cara?" · `/security-review` no encuentra problemas críticos.

### LUC-30 · Chat: interfaz
`ia` · `diseño` · Alta · 1 h
- **Qué:**
  - Burbujas con texto en streaming.
  - Sugerencias iniciales en forma de chips.
  - Estados de "pensando" y de error.
  - Montos con `formatCLP`.
  - Historial por conversación.
- **Listo cuando:** en el celular se siente como el chat de una app real, y el teclado no tapa el campo de texto.

### LUC-31 · Mini-evaluación de la categorización (30 casos)
`ia` · `calidad` · Media · 0,5 h
- **Qué:** `tests/evals/categorize.json` con 30 descripciones de cartola y su categoría esperada, más un script que mide los aciertos. Se corre a mano y cuesta menos de US$0,01.
- **Listo cuando:** acierta 90 % o más, o queda una lista de ajustes al prompt o al diccionario.

## S6 · Integraciones (23–29 nov)

### LUC-32 · Fintoc: Link Intent, widget y canje del token
`integración` · `seguridad` · Alta · 1,5 h
- **Qué:**
  - Cuenta en Fintoc (modo test), con llaves `sk_test_` y `pk_test_`.
  - Migración `integrations`, con el esquema `private`.
  - Server Actions `createLinkIntent()` y `exchangeLink()`.
  - Botón "Conectar banco" que muestra las credenciales de prueba para la demo.
- **Listo cuando:** conectar un banco de prueba crea el `bank_link`, y el token queda solo en `private`.

### LUC-33 · Fintoc: sincronizar cuentas y movimientos sin duplicados
`integración` · Alta · 1 h
- **Qué:** `syncBankLink(bankLinkId)`:
  - Trae las cuentas y los movimientos desde la última fecha.
  - Hace *upsert* por `(account_id, fintoc_movement_id)`.
  - Categoriza y revisa los presupuestos.
  - Se activa con un botón "Sincronizar".
- **Listo cuando:** sincronizar dos veces seguidas no duplica nada.

### LUC-34 · Cron diario en Vercel
`infra` · Alta · 0,5 h
- **Qué:**
  - `vercel.json` con `crons` (`0 11 * * *`) apuntando a `/api/cron/diario`, protegido con `CRON_SECRET`.
  - Pasos 1, 2 y 6: mantener viva la base, sincronizar y limpiar demos. El resto se agrega en S7.
- **Listo cuando:** el job aparece en Vercel → *Cron Jobs* · una ejecución manual responde 200 · sin el secreto responde 401.

### LUC-35 · WhatsApp: núcleo "texto → movimiento" + simulador en la app
`ia` · `integración` · Alta · 1 h
- **Qué:** `handleIncomingText(userId, texto)` con Haiku 5.5 (structured output).
  - Entiende frases como "gasté 3.500 en completos", "uber 4990", "me pagaron 450 lucas" (= $450.000) y "ayer 12 mil en el súper".
  - Responde con un texto de confirmación.
  - Pantalla `/ajustes/whatsapp` con un simulador tipo chat.
- **Listo cuando:** 10 frases de prueba (con test de IA simulada + prueba manual) crean el movimiento correcto.

### LUC-36 · WhatsApp real: app de Meta, número de prueba, webhook firmado y vinculación
`integración` · `seguridad` · Alta · 1,5 h
- **Qué:**
  1. App en developers.facebook.com con el producto WhatsApp, o la de tu agente si ya usa Meta.
  2. Número de prueba, con tu número como destinatario.
  3. Webhook en producción (`/api/whatsapp/webhook`):
     - GET para la verificación inicial.
     - POST con la firma `X-Hub-Signature-256` verificada.
     - Idempotencia y `after()`.
  4. Vinculación por código.
- **Listo cuando:** envías "gasté 3.500 en completos" desde tu WhatsApp, aparece en la app y recibes la confirmación.

## S7 · Diferenciadores (30 nov–6 dic)

### LUC-37 · Detección de suscripciones + alerta de subida de precio
`feature` · `db` · Media · 1 h
- **Qué:**
  - Migración `differentiators`.
  - `detectRecurringCharges()`, con la regla de [esquema-db.md](esquema-db.md) y tests.
  - Pantalla "Suscripciones" con el total mensual.
  - Paso 3 del cron.
- **Listo cuando:** con los datos demo detecta al menos 4 suscripciones y avisa la que subió de precio.

### LUC-38 · Metas de ahorro con proyección
`feature` · Media · 1 h
- **Qué:** `/metas` con:
  - Crear una meta y agregar aportes.
  - Gráfico de progreso.
  - `projectGoal()` con tests: "A este ritmo llegas en marzo de 2027".
  - Paso 4 del cron (avisar meta alcanzada).
- **Listo cuando:** los tests cubren ritmo 0, meta ya cumplida y fecha objetivo imposible.

### LUC-39 · Web Push
`feature` · Baja · 1 h
- **Qué:**
  - Llaves VAPID (`npx web-push generate-vapid-keys`).
  - Activar push en Ajustes.
  - `sendPush()` con `web-push` 3.6, que se llama al crear una notificación.
  - Suscripciones vencidas: si el envío responde 410, se borran.
- **Listo cuando:** llega un push al iPhone (con la PWA instalada) y al Android al cruzar un presupuesto.

### LUC-40 · Bloqueo biométrico con WebAuthn
`feature` · `seguridad` · Baja · 1,5 h
- **Qué:** con SimpleWebAuthn 14:
  - Registro de la llave en Ajustes.
  - Pantalla de bloqueo en `(app)/layout`.
  - Verificación en el servidor y cookie firmada de 15 minutos.
  - Se bloquea tras 5 minutos en segundo plano.
- **Listo cuando:** Face ID o la huella desbloquean · sin verificar no se ve ningún dato, ni siquiera difuminado.

### LUC-41 · Resumen semanal
`feature` · Baja · 0,5 h
- **Qué:** paso 5 del cron, los lunes: "La semana pasada gastaste $X (−12 % vs. la anterior). Tu categoría top: Comida". Llega in-app y por push.
- **Listo cuando:** forzar el cron con una fecha de lunes genera una notificación por usuario.

## S8 · Lanzamiento (7–13 dic)

### LUC-42 · Landing pública con "Probar demo"
`diseño` · Alta · 1 h
- **Qué:** la página `/` explica LUCA en 5 segundos:
  - Titular, 3 capturas y 4 beneficios.
  - Un botón grande: "Probar demo — sin registro".
  - Imagen Open Graph para que el link se vea bien al compartirlo.
- **Listo cuando:** pegar el link en WhatsApp o LinkedIn muestra una vista previa atractiva.

### LUC-43 · Sentry + Vercel Analytics
`infra` · Media · 0,5 h
- **Qué:** `npx @sentry/wizard@latest -i nextjs` · el componente `<Analytics />` de `@vercel/analytics`.
- **Listo cuando:** un error de prueba aparece en Sentry y las visitas aparecen en Vercel.

### LUC-44 · Tests end-to-end con Playwright (3 flujos)
`calidad` · Media · 1 h
- **Qué:** Playwright 1.64, con Chromium y WebKit móvil. Los 3 flujos de [decisiones.md](decisiones.md#23-tests-end-to-end--playwright-164), corriendo en el CI.
- **Listo cuando:** pasan en verde en el CI.

### LUC-45 · Auditoría final de seguridad y rendimiento
`seguridad` · Alta · 0,5 h
- **Qué:** `/security-review` de todo el repo, advisors de Supabase en `luca-prod`, revisión de las variables en Vercel y Lighthouse.
- **Listo cuando:** no quedan hallazgos críticos ni altos.

### LUC-46 · README profesional
`docs` · Alta · 1 h
- **Qué:** el README incluye:
  - Qué es LUCA y un GIF de 10 segundos.
  - Capturas.
  - El stack y por qué se eligió.
  - El diagrama de arquitectura.
  - Cómo correrlo.
  - Decisiones destacadas: RLS, IA segura y demo anónima.
  - Qué aprendiste.
  - Link a la demo.
- **Listo cuando:** alguien que no te conoce entiende el proyecto en 30 segundos.

### LUC-47 · Video demo + release v1.0
`docs` · Alta · 1 h
- **Qué:**
  - Video de 60 a 90 segundos con la demo, el WhatsApp real y el chat.
  - `git tag v1.0.0` + GitHub Release.
  - Post en LinkedIn.
- **Listo cuando:** el link de la demo, el repo y el video están listos para mandar a startups el lunes 14-12-2026.

---

## Extras (solo si sobra tiempo; no se crean en Linear hasta que se usen)

- Actualización en vivo con *Refresh Intents* de Fintoc + webhook.
- Login con Google.
- Cloudflare Turnstile (captcha invisible) en "Probar demo".
- PostHog con grabación de sesiones.
- Captura automática de compras con **Atajos de iPhone**, equivalente a la captura desde Wallet de Not Pato: una automatización de "Transacción" envía cada compra a LUCA con un token personal.
- Exportar movimientos a CSV.
- Migrar a TypeScript 7.
- Subir el chat a Sonnet 5.5 si la mini-evaluación lo justifica.
