# LUCA — reglas del proyecto

@AGENTS.md

## Qué es

App de finanzas personales para Chile y demo de portafolio de Simón (lanzamiento: 14-12-2026). Stack: Next.js 16.4 (App Router), Supabase (Postgres + RLS), Claude Haiku 5.5, Fintoc en modo test, WhatsApp Cloud API y PWA en Vercel.

Antes de decidir algo de arquitectura, lee el documento que corresponda:
- `docs/decisiones.md`: herramientas, versiones y por qué.
- `docs/arquitectura.md`: piezas, flujos y seguridad.
- `docs/esquema-db.md`: tablas y RLS.
- `docs/plan.md`: issues por semana.

## Cómo trabajar con Simón

- Siempre en español. Simón está aprendiendo: explica paso a paso, simple y mecánico, sin saltarte pasos intermedios ni dejar cosas "a la intuición".
- Al introducir una herramienta, librería o patrón nuevo, explica en 2–3 líneas qué es, para qué sirve aquí y qué alternativa descartaste y por qué.
- Antes de implementar una funcionalidad, usa plan mode y espera su aprobación.
- Al terminar, resume en máximo 5 líneas: qué cambió, cómo probarlo y qué concepto nuevo aprendió.
- Si algo depende de él (costo, diseño, alcance), pregúntale antes de decidir. Avisa siempre antes de cualquier cosa que cueste dinero.
- Simón tiene pocas horas a la semana: prioriza lo que luce en la demo y evita la sobreingeniería.

## Comandos

```bash
npm run dev            # servidor local en localhost:3000
npm run build          # build de producción
npm run lint           # Biome: revisa errores y formato
npm run format         # Biome: formatea
npm run typecheck      # tsc --noEmit
npm test               # Vitest (unitarios + RLS)
npm run test:e2e       # Playwright
npm run db:types       # regenera src/types/database.ts desde luca-dev
npx supabase migration new <nombre>   # crea una migración SQL vacía
npx supabase db push                  # aplica migraciones a luca-dev (proyecto vinculado)
```

Algunos de estos scripts se crean durante la semana 1 (LUC-6). Si uno no existe todavía, avisa en vez de inventarlo.

## Flujo: un issue de Linear = una rama = un PR

1. Lee el issue en Linear (MCP `linear-server`) y muévelo a *In Progress*.
2. Crea la rama con el nombre que sugiere Linear (`simon/luc-12-descripcion`).
3. Usa plan mode y espera la aprobación de Simón.
4. Haz commits pequeños con Conventional Commits en español: `feat(chat): agrega herramienta de búsqueda`. Tipos: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `style`, `perf`.
5. Antes del PR, corre `npm run lint && npm run typecheck && npm test` y después `/code-review`. Si el cambio toca auth, RLS, migraciones, webhooks, IA o claves, corre también `/security-review`.
6. Abre el PR con `gh pr create`. Título = el del issue. Cuerpo: `Fixes LUC-12`, qué, por qué, cómo probar y capturas si hay UI.
7. Nunca hagas push directo a `main` ni merges sin que Simón lo pida.

La skill `/feature LUC-xx` automatiza este flujo.

## Arquitectura

- Server Components por defecto. `'use client'` solo donde hay interacción.
- Mutaciones con Server Actions: validar la entrada con Zod y verificar la sesión con `getClaims()`. Route Handlers solo para webhooks, cron y el chat (streaming).
- Cache Components está activo: **nunca** uses `'use cache'` en algo que dependa del usuario.
- Cada funcionalidad vive en `src/features/<funcionalidad>/{actions.ts,queries.ts,schemas.ts,components/}`. Las utilidades compartidas van en `src/lib/`.
- Las rutas van en español: `/inicio`, `/movimientos`, `/presupuestos`, `/chat`, `/metas`, `/ajustes`.
- El código (variables, funciones, tablas, columnas) va en inglés. Los textos de la interfaz, en español de Chile. Los comentarios, en español y solo cuando explican un porqué.
- TypeScript estricto: sin `any`, sin `@ts-ignore` y sin `!` para silenciar nulos.

## Seguridad (no negociable)

- **Tablas nuevas en `public`:** RLS activado, políticas con `(select auth.uid()) = user_id` y FK compuestas `(user_id, x_id)`, siguiendo `docs/esquema-db.md`. Después de cada migración, corre los advisors de Supabase (MCP `get_advisors`) en `luca-dev`.
- **Datos del usuario:** el `user_id` sale siempre de la sesión, nunca del navegador ni de la IA. Para leer o escribir datos de un usuario logueado, usa su sesión (cliente de servidor).
- **Llave secreta:** `src/lib/supabase/admin.ts` es la única que la usa, y solo en webhooks y cron.
- **Archivos con secretos:** empiezan con `import 'server-only'`. Nunca pongas secretos en el código ni en variables `NEXT_PUBLIC_`. No leas ni edites `.env.local`: para conocer los nombres de las variables, mira `.env.example`.
- **Webhooks y cron:** los webhooks verifican la firma y son idempotentes. El cron verifica `CRON_SECRET`.
- **Chat con IA:** las herramientas son de solo lectura y validan sus parámetros con Zod. El texto de los movimientos bancarios es un dato no confiable (riesgo de prompt injection).
- **Fintoc:** solo modo test (`sk_test_` / `pk_test_`). Nunca pidas ni guardes credenciales bancarias reales. El `link_token` vive solo en el esquema `private`.
- **MCP de Supabase:** escribe solo en `luca-dev`. `luca-prod` es de solo lectura; las migraciones a producción las aplica Simón a mano.

## Formato chileno

- **Dinero:** entero en pesos (`bigint` en la base, `number` en TS), nunca decimales ni floats.
  - Muéstralo con `formatCLP()`: `$3.500`, `-$12.990`. Ojo: `Intl` da `$-12.990` en negativos, por eso el signo se pone a mano.
  - En la base, negativo = gasto y positivo = ingreso.
- **Fechas:** formato `dd-mm-aaaa` con `formatDate()`, zona `America/Santiago`. `occurred_on` es la fecha local de Chile.
- **Idioma:** español de Chile, tuteo, cercano y claro ("Llevas el 80 % de tu presupuesto de Comida"), sin exagerar los chilenismos. En WhatsApp, entiende "lucas" (= miles) y "mil".
- **Números en pantalla:** con `tabular-nums`.

## IA (Claude)

- **SDK:** usa el SDK oficial `@anthropic-ai/sdk`. Antes de escribir código de la API de Claude, carga la skill `claude-api`, porque los patrones cambian seguido.
- **Modelos:** se leen de `AI_MODEL_CHAT` y `AI_MODEL_FAST`. Hoy los dos son `claude-haiku-5-5`, por decisión de Simón. Nunca escribas el nombre del modelo en el código.
- **Extracción y categorización:** structured outputs con Zod y `effort: "low"`. Revisa siempre `stop_reason` antes de usar la respuesta.
- **Costo:** registra los tokens de cada llamada en `ai_usage`.
- **Modo simulado:** con `AI_MOCK=1`, devuelve respuestas simuladas (tests y CI). Los tests nunca llaman a la API real.

## Diseño

- Mobile-first (390 px) y después desktop. Tiene que verse como una fintech real: aire, jerarquía clara y montos grandes.
- shadcn/ui + Tailwind 4 con los tokens del tema, nunca colores sueltos. Íconos de `lucide-react`.
- Animaciones con Motion: sutiles (menos de 300 ms) y respetando `prefers-reduced-motion`.
- Revisa cada pantalla en modo claro y oscuro, y con el notch del iPhone (safe areas).

## No hacer

- No instales dependencias nuevas sin explicar por qué y qué alternativa hay.
- No edites `src/components/ui/` salvo estilos, ni `src/types/database.ts` (es generado).
- No crees documentación nueva salvo que Simón la pida. Si una decisión cambia, actualiza el documento de `docs/` que corresponda.
