# Arquitectura — LUCA

> Cómo se conectan las piezas: app, base de datos, Fintoc, WhatsApp, IA y tareas programadas. Las razones de cada herramienta están en [decisiones.md](decisiones.md); las tablas, en [esquema-db.md](esquema-db.md).

## Vista general

```mermaid
flowchart LR
  subgraph usuario["Usuario"]
    PWA["📱 LUCA instalada (PWA)"]
    WAU["💬 WhatsApp del usuario"]
  end

  subgraph vercel["Vercel — Next.js 16.4"]
    PROXY["proxy.ts<br/>renueva la sesión"]
    RSC["Páginas<br/>(Server Components)"]
    SA["Server Actions<br/>(crear, editar, borrar)"]
    CHAT["/api/chat<br/>(streaming)"]
    WH["/api/whatsapp/webhook"]
    FWH["/api/fintoc/webhook<br/>(extra)"]
    CRON["/api/cron/diario"]
  end

  subgraph supabase["Supabase"]
    AUTH["Auth<br/>(sesiones, usuarios demo)"]
    DB[("Postgres + RLS<br/>esquema public")]
    PRIV[("esquema private<br/>tokens de Fintoc")]
  end

  CLAUDE["🤖 Claude API<br/>Haiku 5.5"]
  FINTOC["🏦 Fintoc API<br/>(modo test)"]
  META["WhatsApp Cloud API<br/>(Meta)"]
  PUSHS["Servicios Web Push<br/>(Apple / Google)"]
  VCRON["⏰ Vercel Cron<br/>1 vez al día"]

  PWA --> PROXY --> RSC
  PWA --> SA
  PWA --> CHAT
  PWA -. widget .-> FINTOC
  PROXY --> AUTH
  RSC & SA & CHAT -->|token del usuario → RLS| DB
  SA & CHAT --> CLAUDE
  SA <-->|link intent, canje, movimientos| FINTOC
  SA --> PRIV
  WAU --> META -->|webhook firmado| WH
  WH -->|llave secreta| DB
  WH --> CLAUDE
  WH -->|respuesta| META
  FINTOC -. evento .-> FWH --> DB
  VCRON --> CRON
  CRON --> DB & PRIV & FINTOC
  CRON --> PUSHS --> PWA
```

**Cómo leerlo:** las flechas continuas son llamadas que hace tu código, y las punteadas, llamadas que hace el navegador o un servicio externo. Todo lo que está dentro de "Vercel" es **un solo proyecto Next.js**: no hay un backend aparte.

## Las piezas

| Pieza | Dónde corre | Qué hace | Quién la puede llamar |
|---|---|---|---|
| Páginas (Server Components) | Vercel | Leen datos con la sesión del usuario y entregan HTML listo | El navegador del usuario logueado |
| Server Actions | Vercel | Crean, editan y borran datos. Validan con Zod y revisan la sesión | Formularios y botones de la app |
| `proxy.ts` | Vercel | Renueva las cookies de sesión de Supabase en cada visita y redirige al login si no hay sesión | Todas las visitas |
| `/api/chat` | Vercel | Conversa con Claude usando herramientas de solo lectura y responde en streaming | Usuario logueado (máx. 30 preguntas al día) |
| `/api/whatsapp/webhook` | Vercel | Recibe los mensajes de WhatsApp, los convierte en movimientos y responde | Solo Meta (firma verificada) |
| `/api/fintoc/webhook` | Vercel | Avisa cuando un banco terminó de actualizarse (*extra*) | Solo Fintoc (firma verificada) |
| `/api/cron/diario` | Vercel | Hace el mantenimiento diario | Solo Vercel Cron (`CRON_SECRET`) |
| Supabase Auth | Supabase | Usuarios, sesiones, usuarios anónimos de demo | La app |
| Postgres (`public`) | Supabase | Todos los datos, protegidos con RLS | La app con el token del usuario; webhooks y cron con la llave secreta |
| Postgres (`private`) | Supabase | El `link_token` de Fintoc. La API pública no expone este esquema | Solo el servidor (llave secreta) |
| Claude API | Anthropic | Categoriza, lee mensajes de WhatsApp y responde el chat | Solo el servidor (`ANTHROPIC_API_KEY`) |
| Service worker (`public/sw.js`) | El celular | Recibe notificaciones push y muestra la página offline | El navegador |

## Capas de seguridad

Cada capa funciona aunque falle la anterior:

1. **Sesión:**
   - Supabase guarda la sesión en cookies `httpOnly`, que el JavaScript de la página no puede leer.
   - Cada página y Server Action llama a `getClaims()`, que verifica la firma del token.
   - El `user_id` **siempre** sale de la sesión, nunca de lo que envía el navegador ni de lo que diga la IA.
2. **RLS en Postgres:** aunque haya un bug en el código, la base de datos rechaza leer o escribir filas de otro usuario. Las llaves foráneas compuestas impiden además apuntar a la categoría o cuenta de otra persona (detalle en [esquema-db.md](esquema-db.md)).
3. **Claves:**
   - Solo existen en variables de entorno (`.env.local` en tu Mac, el panel de Vercel en la nube).
   - Los archivos que las usan empiezan con `import 'server-only'`, así que Next.js da error si alguien los importa desde el navegador.
   - Claude Code tiene prohibido leer o editar `.env.local`, y GitHub bloquea cualquier push que contenga una clave.
4. **Webhooks:**
   - Se verifica la firma de cada request: Meta usa `X-Hub-Signature-256`, un HMAC calculado con tu App Secret.
   - Cada mensaje se procesa una sola vez, gracias a un ID único (*idempotencia*).
   - Se responde `200` de inmediato y el trabajo se hace después con `after()` de Next.js, para que Meta no reintente.
5. **Cron:** Vercel envía `Authorization: Bearer <CRON_SECRET>`; sin eso, la ruta responde 401.
6. **IA:**
   - Herramientas de solo lectura con parámetros validados con Zod.
   - Máximo 5 llamadas a herramientas por pregunta y 30 preguntas al día.
   - Límite de gasto en la consola de Claude.
   - El texto de los movimientos bancarios se trata como dato no confiable (ver [chat](#4-chat-con-tus-finanzas)).
7. **Bloqueo biométrico:** una cookie firmada de 15 minutos, encima de la sesión normal.
8. **Datos reales:** Fintoc solo en modo test (`sk_test_` / `pk_test_`). LUCA nunca pide ni guarda credenciales bancarias reales.

---

## Flujos principales

### 1. "Probar demo"

```mermaid
sequenceDiagram
  autonumber
  actor V as Visitante
  participant App as LUCA (navegador)
  participant SA as Server Action
  participant Auth as Supabase Auth
  participant DB as Postgres

  V->>App: Toca "Probar demo"
  App->>SA: startDemo()
  SA->>Auth: signInAnonymously()
  Auth->>DB: crea usuario anónimo
  DB->>DB: trigger: crea profile + 16 categorías por defecto
  Auth-->>SA: sesión (cookies)
  SA->>DB: generateDemoData(userId): ~300 movimientos de 3 meses,<br/>presupuestos, 2 metas, suscripciones
  SA-->>App: redirige a /inicio
  App-->>V: Panel con datos "vivos"
```

- Cada visitante tiene **su propia copia** de los datos demo, separada de las demás por RLS.
- El cron diario borra los usuarios anónimos con más de 7 días; al borrarse el usuario, se borran en cascada todos sus datos.
- Los datos se generan con números pseudoaleatorios a partir del `userId`: siempre parecen distintos pero creíbles. Por ejemplo, sueldo a fin de mes, arriendo el día 5, supermercado cada semana, Uber, delivery y suscripciones, con una que sube de precio.

### 2. Gasto por WhatsApp

```mermaid
sequenceDiagram
  autonumber
  actor U as Usuario
  participant Meta as WhatsApp Cloud API
  participant WH as /api/whatsapp/webhook
  participant DB as Postgres
  participant AI as Claude Haiku 5.5

  U->>Meta: "gasté 3.500 en completos"
  Meta->>WH: POST + header X-Hub-Signature-256
  WH->>WH: verifica firma (HMAC con App Secret)
  WH-->>Meta: 200 OK inmediato
  Note over WH: el resto corre en after()
  WH->>DB: ¿ya procesé este wa_message_id? (idempotencia)
  WH->>DB: busca el usuario por teléfono (whatsapp_links)
  WH->>AI: extrae {tipo, monto, categoría, fecha, descripción}<br/>(structured output, effort low)
  AI-->>WH: {expense, 3500, "Comida y delivery", hoy, "completos"}
  WH->>DB: inserta movimiento (source = whatsapp)
  WH->>DB: revisa presupuesto → notificación si cruza 80 % o 100 %
  WH->>Meta: "✅ Anoté $3.500 en Comida y delivery (completos).<br/>Llevas el 82 % de tu presupuesto."
  Meta-->>U: respuesta
```

- **Vincular el número:**
  1. En Ajustes → "Conectar WhatsApp", la app muestra un código (por ejemplo, `LUCA-4821`) que dura 10 minutos.
  2. El usuario se lo envía al número de LUCA.
  3. El webhook asocia ese teléfono a su cuenta.
- **Núcleo compartido:** la función `handleIncomingText(userId, texto)` es la misma para el webhook real y para el **simulador de WhatsApp** de la app. Así lo pruebas en tu Mac sin depender de Meta, y un reclutador lo ve funcionar sin estar en la lista de 5 números.
- **Mensajes ambiguos:** si la IA no entiende el mensaje (confianza baja), LUCA responde con un ejemplo de cómo escribirlo en vez de inventar datos.

### 3. Conectar un banco con Fintoc

```mermaid
sequenceDiagram
  autonumber
  actor U as Usuario
  participant App as LUCA (navegador)
  participant SA as Server Actions
  participant F as Fintoc API (test)
  participant W as Widget de Fintoc
  participant DB as Postgres
  participant P as esquema private
  participant AI as Claude Haiku 5.5

  U->>App: "Conectar banco"
  App->>SA: createLinkIntent()
  SA->>F: crea Link Intent (product movements, country cl,<br/>holder_type individual) con sk_test_
  F-->>SA: widget_token
  SA-->>App: widget_token
  App->>W: Fintoc.create({ publicKey: pk_test_, widgetToken, onSuccess })
  U->>W: elige banco + credenciales de prueba
  W-->>App: onSuccess → exchangeToken
  App->>SA: exchangeLink(exchangeToken)
  SA->>F: canjea exchange_token
  F-->>SA: Link con link_token
  SA->>P: guarda link_token (nunca va al navegador)
  SA->>DB: crea bank_link
  SA->>F: lista cuentas y movimientos
  SA->>AI: categoriza movimientos nuevos (lotes de 25)
  SA->>DB: upsert cuentas + movimientos (únicos por fintoc_movement_id)
  SA-->>App: "Listo: 2 cuentas, 143 movimientos"
```

- **Re-sincronizar:** el botón "Sincronizar" y el cron diario piden solo los movimientos desde la última fecha guardada. La restricción única `(account_id, fintoc_movement_id)` hace que repetir la sincronización nunca duplique movimientos.
- **Extra:** pedir una actualización en vivo al banco con un *Refresh Intent* y esperar el webhook `account.refresh_intent.succeeded`. La firma se verifica según la guía de webhooks de Fintoc.

### 4. Chat con tus finanzas

```mermaid
sequenceDiagram
  autonumber
  actor U as Usuario
  participant API as /api/chat
  participant AI as Claude Haiku 5.5
  participant DB as Postgres (RLS)

  U->>API: "¿cuánto gasté en Uber este mes?"
  API->>API: getClaims() → user_id · límite diario (ai_usage)
  API->>AI: system + historial + definición de herramientas
  AI-->>API: tool_use: search_transactions({ query: "uber",<br/>from: "2026-10-01", to: "2026-10-31" })
  API->>API: valida parámetros con Zod
  API->>DB: consulta con el token del usuario
  DB-->>API: 6 movimientos, total $34.200
  API->>AI: tool_result
  AI-->>API: (streaming) "Este mes llevas $34.200 en Uber, en 6 viajes…"
  API-->>U: texto palabra por palabra
  API->>DB: guarda mensajes + tokens usados (ai_usage)
```

**Herramientas del chat** (todas de solo lectura y todas filtradas al usuario de la sesión):

| Herramienta | Devuelve |
|---|---|
| `get_spending_summary({ from, to, group_by })` | Totales por categoría, comercio o mes |
| `search_transactions({ query?, category?, from, to, limit ≤ 50 })` | Movimientos que calzan |
| `get_budgets_status({ month })` | Presupuesto vs. gastado por categoría |
| `get_recurring_charges()` | Suscripciones detectadas, con su variación de precio |
| `get_savings_goals()` | Metas y fecha proyectada |
| `get_balances()` | Saldo por cuenta y total |

**Por qué es seguro, paso a paso:**

1. Claude **nunca recibe** claves, SQL ni acceso directo a la base. Solo puede pedir que ejecutemos una de estas 6 funciones, que escribimos nosotros.
2. Los parámetros que Claude elige (fechas, texto de búsqueda) se validan con Zod antes de usarse.
3. El `user_id` **no es un parámetro**: sale de la sesión. Aunque Claude "quisiera" ver datos de otro usuario, la consulta corre con el token del usuario y RLS la bloquea.
4. Las herramientas son de solo lectura, así que el chat no puede borrar ni cambiar nada.
5. **Prompt injection:** una descripción bancaria podría decir "IGNORA TUS INSTRUCCIONES". Como las herramientas son de solo lectura y están limitadas al usuario, lo peor que puede pasar es una respuesta rara, nunca una fuga de datos.
6. Las herramientas devuelven totales y resúmenes en vez de volcar todo el historial: menos datos expuestos y menos tokens.
7. **Detalles del prompt:**
   - El system prompt y las herramientas no cambian, así que se cachean (más barato y rápido).
   - La fecha de hoy (America/Santiago) va al final, para no romper el caché.
   - El bot responde solo sobre finanzas personales.

### 5. Categorización automática

```mermaid
flowchart TD
  A["Movimiento nuevo<br/>'COMPRA LIDER 4421 PTO VARAS'"] --> B["Normalizar<br/>→ 'lider'"]
  B --> C{"¿Regla del usuario?<br/>(merchant_rules)"}
  C -- sí --> OK["Categoría asignada<br/>categorized_by = rule"]
  C -- no --> D{"¿Comercio conocido?<br/>(diccionario chileno)"}
  D -- sí --> OK
  D -- no --> E["Claude Haiku 5.5<br/>lote de 25, structured output"]
  E --> F{"¿confianza ≥ 0,6?"}
  F -- sí --> OK2["categorized_by = ai"]
  F -- no --> G["'Sin categoría'<br/>+ pide confirmar"]
  OK2 & G --> H["El usuario corrige"] --> I["Se crea regla para ese comercio<br/>→ la próxima vez no se llama a la IA"]
```

- **El diccionario base** cubre los comercios más comunes en Chile: Lider, Jumbo, Unimarc, Tottus, Santa Isabel, Copec, Shell, Uber, Cabify, DiDi, Rappi, PedidosYa, Netflix, Spotify, Entel, Movistar, WOM, Enel, Aguas Andinas, Cruz Verde, Salcobrand, Falabella, Ripley, Paris, Mercado Libre, etc.
- **Ahorro:** la IA solo ve lo que el diccionario y las reglas no resuelven. Así es más barato, más rápido y más predecible.

### 6. Cron diario

```mermaid
flowchart LR
  T["⏰ Vercel Cron<br/>11:00 UTC"] --> R["/api/cron/diario<br/>verifica CRON_SECRET"]
  R --> K["1 · Consulta a la base<br/>(evita la pausa de 7 días)"]
  R --> S["2 · Sincroniza bancos Fintoc"]
  R --> D["3 · Detecta suscripciones<br/>y subidas de precio"]
  R --> M["4 · Revisa metas"]
  R --> W["5 · Lunes: resumen semanal"]
  R --> X["6 · Borra usuarios demo<br/>de más de 7 días"]
  D & M & W --> N["Notificación in-app + push"]
```

Cada paso va en su propio `try/catch`: si falla uno, los demás igual corren, y el error queda registrado en Sentry.

---

## Estructura de carpetas

```text
luca/
├── CLAUDE.md                     ← reglas para Claude Code
├── AGENTS.md                     ← lo genera Next.js (apunta a su documentación)
├── .claude/
│   ├── settings.json             ← permisos + hook de formato
│   └── skills/feature/SKILL.md   ← flujo "issue → PR"
├── .github/workflows/ci.yml
├── docs/                         ← estos documentos
├── supabase/
│   ├── config.toml
│   └── migrations/               ← un archivo .sql por cambio de esquema
├── public/
│   ├── sw.js                     ← service worker (push + offline)
│   └── icons/
├── src/
│   ├── proxy.ts                  ← sesión de Supabase
│   ├── app/
│   │   ├── manifest.ts           ← PWA
│   │   ├── (publico)/            ← landing, /ingresar
│   │   ├── (app)/                ← layout con tab bar + candado biométrico
│   │   │   ├── inicio/
│   │   │   ├── movimientos/
│   │   │   ├── presupuestos/
│   │   │   ├── chat/
│   │   │   ├── metas/
│   │   │   └── ajustes/
│   │   └── api/
│   │       ├── chat/route.ts
│   │       ├── whatsapp/webhook/route.ts
│   │       ├── fintoc/webhook/route.ts
│   │       └── cron/diario/route.ts
│   ├── features/                 ← una carpeta por funcionalidad
│   │   └── transactions/         ← actions.ts · queries.ts · schemas.ts · components/
│   ├── components/ui/            ← componentes de shadcn
│   ├── lib/
│   │   ├── supabase/             ← browser.ts · server.ts · admin.ts
│   │   ├── ai/                   ← cliente Claude, prompts, herramientas, modo simulado
│   │   ├── fintoc/
│   │   ├── whatsapp/             ← interfaz WhatsAppProvider + implementación Meta
│   │   ├── format/               ← formatCLP, formatDate
│   │   └── env.ts                ← variables de entorno validadas con Zod
│   └── types/database.ts         ← generado por Supabase (no editar)
└── tests/
    ├── unit/
    └── e2e/
```

**Por qué `features/`:** todo lo de una funcionalidad (sus Server Actions, sus consultas, sus esquemas Zod y sus componentes) vive junto. Para entender "presupuestos", abres una sola carpeta.

## Entornos

| Entorno | URL | Supabase | Fintoc | WhatsApp |
|---|---|---|---|---|
| Local | `localhost:3000` | `luca-dev` | test | simulador |
| Preview (cada PR) | `luca-git-<rama>-….vercel.app` | `luca-dev` | test | simulador |
| Producción | `luca-finanzas.vercel.app` | `luca-prod` | test | número de prueba de Meta |

## Variables de entorno

Los nombres van en `.env.example`, que sí se sube a git. Los valores van en `.env.local` (nunca se sube) y en el panel de Vercel.

| Variable | Dónde se usa | ¿Pública? |
|---|---|---|
| `NEXT_PUBLIC_APP_URL` | Links absolutos, WebAuthn | Sí |
| `NEXT_PUBLIC_SUPABASE_URL` | Clientes de Supabase | Sí |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | Clientes de Supabase (`sb_publishable_…`) | Sí (protegida por RLS) |
| `SUPABASE_SECRET_KEY` | `lib/supabase/admin.ts` (webhooks, cron) | **No** |
| `ANTHROPIC_API_KEY` | `lib/ai` | **No** |
| `AI_MODEL_CHAT` / `AI_MODEL_FAST` | `lib/ai` (hoy `claude-haiku-5-5`) | No |
| `AI_MOCK` | `1` = respuestas simuladas (tests, CI) | No |
| `NEXT_PUBLIC_FINTOC_PUBLIC_KEY` | Widget (`pk_test_…`) | Sí |
| `FINTOC_SECRET_KEY` | `lib/fintoc` (`sk_test_…`) | **No** |
| `FINTOC_WEBHOOK_SECRET` | `/api/fintoc/webhook` (extra) | **No** |
| `WHATSAPP_ACCESS_TOKEN` · `WHATSAPP_PHONE_NUMBER_ID` | Enviar mensajes | **No** |
| `WHATSAPP_APP_SECRET` | Verificar firma del webhook | **No** |
| `WHATSAPP_VERIFY_TOKEN` | Verificación inicial del webhook (GET) | **No** |
| `NEXT_PUBLIC_VAPID_PUBLIC_KEY` · `VAPID_PRIVATE_KEY` · `VAPID_SUBJECT` | Web Push | Solo la pública |
| `UNLOCK_COOKIE_SECRET` | Firmar la cookie del candado biométrico | **No** |
| `CRON_SECRET` | `/api/cron/diario` | **No** |
| `SENTRY_*` | Las agrega el asistente de Sentry | Depende |

> Las migraciones a `luca-prod` no usan variables: se aplican a mano con el CLI de Supabase (ver [guía §5](guia-configuracion.md#5-el-ciclo-de-un-issue-de-punta-a-punta)).

> Regla: una variable con prefijo `NEXT_PUBLIC_` termina en el navegador de cualquiera. **Nunca** le pongas ese prefijo a algo secreto.
