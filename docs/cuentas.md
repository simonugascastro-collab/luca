# Cuentas necesarias — LUCA

> 🟢 gratis · 🟡 gratis con un límite que hay que vigilar · 🔴 de pago (nunca la actives sin decidirlo antes)

## Las que ya tienes

| Cuenta | Para qué | Costo | Notas |
|---|---|---|---|
| GitHub | Repo, PRs, CI | 🟢 | Repo **público**: los minutos de Actions son gratis y GitHub bloquea los push con claves |
| Vercel (Hobby) | Deploy, previews, cron | 🟢 | Uso personal no comercial: una demo de portafolio califica. Cron máximo 1 vez al día |
| Supabase (Free) | Base de datos y login | 🟡 | 2 proyectos activos (`luca-dev` y `luca-prod`). **Se pausan tras 7 días sin actividad**: lo evita el cron diario. No reactives "Fintax" o te quedas sin cupo |
| Claude Pro | Claude Code | (ya lo pagas) | Es **distinto** de la API de Claude (ver abajo) |

## Las que tienes que crear

| Cuenta | Para qué | Cuándo | Costo | ¿Pide tarjeta? |
|---|---|---|---|---|
| [Linear](https://linear.app) | Tablero de issues | S1 (LUC-5) | 🟢 Free: 250 issues, 2 equipos | No |
| [Fintoc](https://app.fintoc.com) | Bancos en modo test | S6 (LUC-32) | 🟢 Modo test gratis | No |
| [Meta for Developers](https://developers.facebook.com) + app con WhatsApp | Bot de WhatsApp | S6 (LUC-36) | 🟢 Número de prueba gratis (hasta 5 destinatarios verificados) | No, para el número de prueba |
| [Claude Console](https://platform.claude.com) (API) | IA dentro de LUCA | S5 (LUC-26) | 🔴 **Prepago por uso** | Sí |
| [Sentry](https://sentry.io) | Errores en producción | S8 (LUC-43) | 🟢 Developer: 5.000 errores/mes | No |

### Sobre la API de Claude (la única que cuesta)

- Es un cobro **aparte** de tu suscripción Pro. Se paga por tokens usados, con créditos que cargas antes.
- Con Haiku 5.5 (US$0,10 por millón de tokens de entrada y US$0,50 por millón de salida):
  - una pregunta del chat cuesta ~US$0,001;
  - categorizar 25 movimientos cuesta menos de US$0,001;
  - **todo el proyecto debería costar menos de US$3**.
- **Antes de usarla:**
  1. Carga el mínimo de créditos que permita la consola.
  2. Fija un **límite de gasto mensual de US$5**.
  3. Crea una API key solo para LUCA.
- La app pone además un tope de 30 preguntas de chat por usuario al día, así que un reclutador entusiasta no puede gastarte los créditos.

## Opcionales (solo si sobra tiempo)

| Cuenta | Para qué | Costo |
|---|---|---|
| Cloudflare (Turnstile) | Captcha invisible en "Probar demo" | 🟢 |
| Google Cloud | Botón "Entrar con Google" | 🟢 |
| PostHog | Analítica con grabación de sesiones | 🟢 1 millón de eventos/mes |
| Conductor | Varios agentes en paralelo | 🟢 la app (usa tu plan de Claude); tiene planes de pago para nube y equipos |
| Dominio propio (por ejemplo, `luca.cl`) | URL más profesional | 🔴 pago anual. **No hace falta:** `luca-finanzas.vercel.app` es gratis |

## Antes de crear cuentas: edad mínima

Revisa la edad mínima en los términos de cada servicio. Varios servicios para desarrolladores (por ejemplo, la consola de la API de Claude y algunas plataformas de pagos o datos bancarios) exigen tener **18 años**. Si te falta, lo correcto es que la cuenta (y la tarjeta) la abra un adulto responsable y te dé acceso como miembro.

## Seguridad de las cuentas

- Activa la **verificación en dos pasos** en GitHub, Vercel, Supabase, Meta y la consola de Claude. Una clave filtrada de cualquiera de ellas da acceso a tu demo.
- Guarda las contraseñas de la base de datos de Supabase en un gestor de contraseñas, nunca en un archivo del proyecto.
- Las claves de API van solo en `.env.local` (tu Mac) y en el panel de Vercel.
