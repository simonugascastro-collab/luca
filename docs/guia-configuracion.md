# Guía de configuración — LUCA

> Paso a paso para dejar listo el entorno de trabajo "de startup": Linear, Claude Code, trabajo en paralelo y el ciclo de cada issue. Corresponde a los issues LUC-1 a LUC-5 del [plan](plan.md).

**Al terminar esta guía tendrás:**

- Un tablero en Linear con todos los issues.
- Claude Code conectado a Linear.
- Reglas y automatizaciones del proyecto.
- Un flujo claro para pasar de "issue" a "código en producción".

---

## 1. Preparar el Mac (LUC-1 y LUC-2)

Ya tienes Homebrew, nvm, git, el CLI de Vercel, el CLI de Supabase y Claude Code. Falta esto:

```bash
# Node 24 LTS (el mismo que usa Vercel). Tu Node 26 queda instalado; solo cambia el que se usa por defecto.
nvm install 24
nvm alias default 24
node -v                      # → v24.x

# GitHub CLI: Claude lo usa para abrir PRs.
brew install gh
gh auth login                # elige GitHub.com → HTTPS → login con el navegador

# Actualizar el CLI de Supabase (tienes la 2.109).
brew upgrade supabase
supabase --version           # → 2.121 o superior
```

Lo que encontramos al hacerlo (16-10-2026):

1. **`~/.npmrc` rompía `npm install`.** Tenía la línea `allow-scripts=@anthropic-ai/claude-code`, y el npm de Node 24 (11.19) rechaza esa opción dentro de un proyecto (error `EALLOWSCRIPTS`). Se renombró a `~/.npmrc.respaldo`.
2. **`claude` y `vercel` estaban instalados dentro de Node 26.** Al pasar a Node 24 habrían desaparecido de la terminal. Solución:
   ```bash
   nvm use 26 && claude install      # Claude Code nativo en ~/.local/bin: ya no depende de Node
   nvm use 24 && npm install -g vercel
   ```
3. **npm no acepta mayúsculas en el nombre del proyecto**, y la carpeta se llama `LUCA`. Por eso el proyecto se crea en una subcarpeta `luca` y se sube un nivel. Los comandos exactos están en **LUC-2** de [plan.md](plan.md#luc-2--crear-el-proyecto-nextjs-164-y-el-repo-en-github).

---

## 2. Linear paso a paso

**Qué es Linear:** un gestor de tareas (*issues*) para equipos de software. Cada tarea tiene un ID (`LUC-12`), un estado, una prioridad y una estimación. Las tareas se agrupan en **ciclos** (*sprints* de una semana) y en **proyectos**. Se conecta con GitHub, así que las tareas se mueven solas cuando abres o mergeas un PR.

### 2.1 Cuenta y espacio de trabajo (5 min)

1. Entra a [linear.app](https://linear.app) → *Sign up* con tu cuenta de Google o GitHub. El plan **Free** alcanza: 250 issues y 2 equipos.
2. Crea el *workspace* con el nombre `LUCA`.
3. Salta el paso de invitar personas.

### 2.2 Equipo (2 min)

1. Abre *Settings → Teams* y entra al equipo que Linear creó.
2. Cambia el nombre a `LUCA` y el **identificador** a `LUC`. El identificador es el prefijo de los issues: `LUC-1`, `LUC-2`…

### 2.3 Estados del flujo (2 min)

En *Settings → Teams → LUCA → Workflow*, deja estos estados (los trae por defecto):

`Backlog` → `Todo` → `In Progress` → `In Review` → `Done` (y `Canceled`)

Si no existe *In Review*, agrégalo dentro del grupo *Started*.

### 2.4 Ciclos semanales (3 min)

En *Settings → Teams → LUCA → Cycles*:

1. Activa los ciclos.
2. Duración: **1 semana**. Inicio: **lunes**. El primer ciclo parte el 19-10-2026.
3. Ciclos futuros visibles: 2 o 3.
4. Los issues no terminados pasan solos al ciclo siguiente.

### 2.5 Estimaciones (1 min)

En la configuración del equipo, activa las *Estimates* con la escala **Linear** (1, 2, 3, 4, 5). En LUCA, **1 punto = 30 minutos**.

### 2.6 Etiquetas (3 min)

En *Settings → Teams → LUCA → Labels*, crea:

`feature` · `diseño` · `ia` · `integración` · `db` · `infra` · `seguridad` · `calidad` · `docs` · `chore`

### 2.7 Proyecto (2 min)

En *Projects → New project*:

- **Nombre:** `LUCA v1 — Demo`.
- **Fecha objetivo:** 14-12-2026.
- **Responsable:** tú.
- **Milestones:** `S1 Cimientos`, `S2 Datos y acceso`, `S3 Panel`, `S4 Presupuestos + PWA`, `S5 IA`, `S6 Integraciones`, `S7 Diferenciadores`, `S8 Lanzamiento`.

### 2.8 Integración con GitHub (5 min)

Requisito: el repo `luca` tiene que existir (LUC-2).

1. En *Settings → Integrations → GitHub → Connect*, autoriza la app de Linear en GitHub y elige el repo `luca`.
2. En las automatizaciones de Git del equipo, configura:
   - PR abierto → **In Review**.
   - PR mergeado → **Done**.
3. **Nombre de la rama:** dentro de un issue, `Cmd + Shift + .` copia el nombre sugerido (por ejemplo, `simon/luc-12-agregar-gasto`). Cuando una rama tiene ese formato, Linear la asocia sola al issue.
4. **Palabras mágicas:** si la descripción del PR dice `Fixes LUC-12`, Linear cierra el issue al mergear.

### 2.9 Conectar Linear con Claude Code (MCP) (5 min)

**Qué es MCP:** *Model Context Protocol*, un estándar para que Claude use herramientas externas. Con el servidor MCP oficial de Linear, Claude puede leer, crear y actualizar issues directo desde VS Code.

1. En la terminal, dentro del proyecto:
   ```bash
   cd ~/Desktop/luca
   claude mcp add --transport http linear-server https://mcp.linear.app/mcp
   ```
   Por defecto, esto lo guarda solo para ti y para este proyecto. Los equipos suelen agregar `--scope project` para que quede en un archivo `.mcp.json` que comparte todo el equipo. Como usa OAuth (login con el navegador), ese archivo no contiene secretos.
2. Abre Claude Code y escribe `/mcp`.
3. Elige `linear-server` → *Authenticate*. Se abre el navegador: autoriza el acceso a tu workspace `LUCA`.
4. De vuelta en Claude Code, `/mcp` debe mostrar `linear-server` como conectado.
5. **Prueba:** pídele a Claude *"lista los issues del equipo LUC"*.

> Si en `/mcp` ves **dos** servidores de Linear (uno viene del plugin *product-management*), autentica solo `linear-server` para no duplicar herramientas.
> Si el panel de la extensión de VS Code no muestra `/mcp`, abre la terminal integrada, escribe `claude` y hazlo desde ahí.

### 2.10 Importar el plan (10 min)

Pídele esto a Claude:

> Lee `docs/plan.md` y crea en Linear, en el equipo LUC y el proyecto "LUCA v1 — Demo", un issue por cada LUC-x en orden. Cada uno con:
> - el título;
> - la descripción ("Qué" + "Listo cuando");
> - las etiquetas;
> - la prioridad (Alta = High, Media = Medium, Baja = Low);
> - la estimación (1 punto = 30 min);
> - el milestone y el ciclo de su semana.
>
> Marca como Done los que ya estén hechos. Antes de crear nada, muéstrame la lista completa y espera mi OK.

Si el MCP no permite asignar el ciclo, hazlo a mano: en la vista del proyecto, filtra por milestone, selecciona todos los issues con `Cmd + A`, abre el menú de acciones (clic derecho) → *Cycle* y elige la semana.

### 2.11 Rutina semanal

| Cuándo | Qué haces | Tiempo |
|---|---|---|
| Lunes | Abre el ciclo de la semana. Pídele a Claude: *"resume el ciclo actual y propón el orden"* | 15 min |
| Cada sesión | `/feature LUC-xx` (ver §5) | 1–2 h |
| Domingo | Revisa qué quedó pendiente (pasa solo al ciclo siguiente) y ajusta `docs/plan.md` si algo cambió | 10 min |

---

## 3. Claude Code para este proyecto

Esto se hace en **LUC-4**, después de crear el proyecto (si lo haces antes, `create-next-app` rechaza la carpeta).

### 3.1 `CLAUDE.md` (ya está hecho)

- **Qué es:** un archivo que Claude Code lee al empezar cada sesión: el contexto del proyecto, tus preferencias, los comandos y las reglas de seguridad.
- **Qué ganas:** no tienes que repetir "explícame en español", "usa RLS" o "los montos van sin decimales" en cada conversación.
- **AGENTS.md:** la línea `@AGENTS.md` importa ese archivo, que genera Next.js y apunta a la documentación de tu versión exacta.

### 3.2 Permisos y hook de formato — `.claude/settings.json`

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.local)",
      "Read(./.env.*.local)",
      "Edit(./.env)",
      "Edit(./.env.local)",
      "Edit(./.env.*.local)"
    ]
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path // empty' | xargs -I{} npx --no-install biome check --write --no-errors-on-unmatched {} >/dev/null 2>&1 || true"
          }
        ]
      }
    ]
  }
}
```

- **`permissions.deny`:** las herramientas de archivos de Claude no pueden leer ni editar tus claves. Para saber qué variables existen, Claude mira `.env.example`.
  - Ojo: no es una bóveda. Un comando de terminal podría leerlas igual. Por eso existen la regla en CLAUDE.md y el hecho de que Claude Code te pida permiso antes de ejecutar comandos nuevos.
- **Hook `PostToolUse`:** cada vez que Claude edita o crea un archivo, Biome lo formatea solo. Así el código siempre queda ordenado y el CI no falla por formato.
- Ya tienes `jq` en el Mac (`/usr/bin/jq`), que el hook usa para leer qué archivo se editó.

### 3.3 Skills

**Qué es una skill:** un paquete de instrucciones que Claude carga solo cuando lo necesita. Es como un "manual" para un tipo de tarea.

| Skill | Cómo se instala | Qué ganas |
|---|---|---|
| Supabase | `npx skills add supabase/agent-skills` | Buenas prácticas de seguridad y RLS de Supabase, oficiales |
| `next-dev-loop` | `npx skills add vercel/next.js --skill next-dev-loop` | Claude abre la app en un navegador y verifica que lo que cambió funcione |
| `claude-api` | Ya viene con Claude Code | Patrones actuales de la API de Claude (se usa en S5) |
| `/feature` (propia) | Archivo de abajo | El ciclo completo "issue → PR" con un comando |

Crea `.claude/skills/feature/SKILL.md`:

```markdown
---
name: feature
description: Lleva un issue de Linear de punta a punta (rama, plan, código, checks y PR). Úsala cuando Simón escriba "/feature LUC-12" o "trabajemos en LUC-12".
---

Issue: $ARGUMENTS

1. Lee el issue en Linear (MCP linear-server). Si no existe o no está claro, pregunta.
2. Muévelo a In Progress. Desde main actualizado, crea la rama con el nombre que sugiere Linear.
3. Usa plan mode: explica en español, paso a paso, qué vas a hacer, qué archivos tocas,
   qué herramienta o patrón nuevo aparece (qué es, para qué, qué alternativa descartas)
   y cómo se va a probar. Espera la aprobación.
4. Implementa en commits pequeños (Conventional Commits en español).
5. Corre npm run lint, npm run typecheck y npm test. Arregla lo que falle.
6. Corre /code-review. Si el cambio toca auth, RLS, migraciones, webhooks, IA o claves,
   corre también /security-review.
7. Si hay una migración nueva, recuérdale a Simón que la aplique a luca-prod antes de mergear.
8. Abre el PR con gh pr create ("Fixes LUC-xx", qué, por qué, cómo probar).
9. Cierra con un resumen de máximo 5 líneas: qué cambió, cómo probarlo en el celular
   (URL del preview), qué aprendió Simón y cuál es el siguiente issue del ciclo.
```

### 3.4 MCP que vas a usar

| Servidor | Estado | Para qué |
|---|---|---|
| `linear-server` | Lo agregas en §2.9 | Issues |
| Supabase | Ya conectado (conector de claude.ai) | Ver tablas, correr advisors, buscar en la documentación. **Escribir solo en `luca-dev`** |
| Vercel | Ya conectado | Ver deploys, logs y errores de producción |
| Next.js (`/_next/mcp`) | Viene en `next dev` 16.x | Claude ve las rutas, los errores de compilación y los logs del servidor local. Lo usa la skill `next-dev-loop` ([guía oficial](https://nextjs.org/docs/app/guides/mcp)) |

### 3.5 Funciones de Claude Code que vale la pena usar

| Función | Cómo | Qué ganas |
|---|---|---|
| **Plan mode** | `Shift + Tab` hasta ver "plan mode on" | Claude propone el plan **sin tocar archivos** hasta que apruebas. Es donde más aprendes |
| **`/code-review`** | Antes de cada PR | Una segunda mirada que busca bugs en tu diff |
| **`/security-review`** | PRs de auth, RLS, webhooks, IA o claves | Revisa justo lo que un reclutador técnico miraría primero |
| **`/resume` o `claude --continue`** | Al retomar | Sigue la conversación anterior sin re-explicar |
| **Subagentes** | *"usa un subagente para investigar X"* | La investigación no llena tu conversación principal. No hace falta crear subagentes propios: los que trae Claude Code alcanzan |

**Revisión automática de PRs en GitHub (`/install-github-app`):** funciona con tu plan Pro, pero cada PR gastaría tu límite de uso. Recomendación: no activarla por ahora y usar `/code-review` local, porque así tú decides cuándo gastar.

---

## 4. Trabajar en paralelo: worktrees y Conductor

### Qué es un worktree

Piensa en tu repo como un escritorio con el proyecto encima. Un **worktree** es un segundo escritorio con otra copia de los archivos, en **otra rama**, que comparte el mismo historial de git. Así, dos sesiones de Claude pueden trabajar al mismo tiempo sin pisarse los archivos.

### Recomendación para ti

| Opción | ¿Te conviene ahora? | Por qué |
|---|---|---|
| Una sesión a la vez | **Sí, por defecto** | Con plan Pro, dos agentes gastan tu límite al doble. Con ~5 h a la semana, el cuello de botella eres tú revisando, no Claude escribiendo. Además, ver una cosa a la vez es como mejor se aprende |
| Worktrees nativos (`claude --worktree`) | **A veces** | Para una tarea chica e independiente mientras la principal avanza. Por ejemplo, el README o los tests de formato mientras se construye el chat. Buenos momentos: S7 y S8 |
| Conductor | **No por ahora** | Es una app de Mac muy buena para manejar 3 a 10 agentes en paralelo, pero brilla con un plan Max. Si te cambias a Max, es la opción natural en S7–S8 |

**Regla de oro:** nunca pongas dos sesiones a tocar los mismos archivos ni la misma migración. Paraleliza solo trabajo independiente.

### Cómo usar un worktree, paso a paso

Requisitos (se cumplen al terminar LUC-2 y LUC-4):

- El repo tiene al menos un commit.
- `.claude/worktrees/` está en el `.gitignore`.
- Existe un archivo `.worktreeinclude` en la raíz con esta línea, para que el worktree reciba tus claves:
  ```text
  .env.local
  ```

Pasos:

1. **Terminal 1:** tu sesión normal, trabajando en el issue principal.
2. **Terminal 2**, en la raíz del proyecto:
   ```bash
   claude --worktree readme
   ```
   Esto crea la carpeta `.claude/worktrees/readme/` en una rama nueva llamada `worktree-readme`, que parte desde `main`.
3. Dentro de esa sesión, lo primero: *"instala las dependencias"* (`npm install`). Es una copia limpia y tarda 1–2 minutos.
4. Si necesitas correr la app en los dos lados, la segunda va en otro puerto: `npm run dev -- -p 3001`.
5. Trabaja normal. Al terminar, Claude abre el PR desde ahí.
6. Sal con `/exit`:
   - Si no quedaron cambios, el worktree se borra solo.
   - Si quedaron, Claude pregunta si conservarlo.
   - Para volver después: `claude --worktree readme --resume`.
7. Para ver todos los worktrees: `git worktree list`. Para borrar uno a mano: `git worktree remove .claude/worktrees/readme`.

### Si algún día usas Conductor

1. Descárgalo desde [conductor.build](https://www.conductor.build) (solo Mac) y conecta tu cuenta de GitHub.
2. Cada *workspace* es un worktree con su propio Claude Code, que usa tu misma suscripción.
3. Configura el comando de preparación de cada workspace: `npm install` y copiar `.env.local`.
4. La app es gratis; revisa en su sitio si los planes pagos (nube, equipos) han cambiado.

---

## 5. El ciclo de un issue, de punta a punta

1. **Partir limpio.** En la terminal de VS Code: `git switch main && git pull`.
2. **Empezar:** `/feature LUC-17`.
3. **Plan:** Claude lee el issue, lo mueve a *In Progress*, crea la rama y te muestra el plan. **Léelo completo.** Si algo no lo entiendes, pregunta antes de aprobar: este es el momento de aprender.
4. **Implementación:** mientras Claude implementa, tú pruebas en `localhost:3000`.
   - Para ver la app en tu celular dentro de la misma red Wi-Fi: `npm run dev -- -H 0.0.0.0` y abre `http://<IP-de-tu-Mac>:3000`.
   - La instalación PWA, el push y la biometría necesitan HTTPS: pruébalos en el preview de Vercel.
5. **Checks:** lint, tipos, tests y `/code-review` (los corre la skill).
6. **PR:** Vercel crea el preview y deja la URL en el PR. Ábrelo en tu celular.
7. **Revisar el diff en GitHub.** Si un archivo no te queda claro: *"explícame este archivo paso a paso"*.
8. **Si el PR trae una migración**, aplícala a producción **antes** de mergear. Nuestras migraciones solo agregan cosas, así que el código que ya está en producción sigue funcionando.
   ```bash
   npx supabase link --project-ref <ref-de-luca-prod>   # te pide la contraseña de la base de prod
   npx supabase db push
   npx supabase link --project-ref <ref-de-luca-dev>    # vuelve a dev
   ```
9. **Mergear** con *Squash and merge*: un commit limpio por funcionalidad. Linear mueve el issue a *Done* y Vercel despliega a producción.
10. **Siguiente issue** del ciclo.

---

## 6. Problemas comunes

| Síntoma | Causa | Solución |
|---|---|---|
| `name can no longer contain capital letters` | La carpeta se llama `LUCA` | Crea el proyecto en la subcarpeta `luca` y súbelo un nivel (LUC-2) |
| `npm error code EALLOWSCRIPTS` | La línea `allow-scripts=…` en `~/.npmrc` | `mv ~/.npmrc ~/.npmrc.respaldo` |
| `command not found: claude` (o `vercel`) en una terminal nueva | Estaban instalados dentro de otra versión de Node | Claude Code nativo (`claude install`) y `npm install -g vercel` en Node 24 |
| `Failed to resolve base branch "HEAD"` al usar `--worktree` | El repo no tiene commits | Haz el primer commit |
| La demo muestra un error de conexión a la base | Supabase pausó el proyecto por 7 días sin actividad | Dashboard de Supabase → *Restore*. Revisa que el cron diario esté corriendo |
| `/mcp` no muestra Linear | No se agregó o no se autenticó | `claude mcp list`. Si aparece, vuelve a autenticar con `/mcp` |
| El iPhone no recibe push | La PWA no está instalada, o se abrió desde Safari en vez del ícono | Instálala con Compartir → "Agregar a inicio", ábrela desde el ícono y activa push con un botón |
| Llegas al límite de Claude Pro | Uso intenso o sesiones en paralelo | Espera a que se renueve tu ventana de uso. Mientras, lee el código o prueba en el celular. Para tareas mecánicas, `/model` te deja elegir un modelo más liviano si tu plan lo ofrece |
| El webhook de WhatsApp no recibe nada | Meta no llega a la URL, o la protección de Vercel la bloquea | Usa la URL de **producción** (pública) y revisa que el token de verificación coincida |
