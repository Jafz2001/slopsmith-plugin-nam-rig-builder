# Seguridad: cómo Rig Builder accede a tone3000 (v1.1.0)

**Resumen corto (para pegar en Discord):**

> Desde la v1.1.0, Rig Builder **ya no te pide pegar ninguna clave**. Te
> conectas con un **login** ("Connect with tone3000"), igual que "Iniciar
> sesión con Google": se abre tone3000 en tu navegador, apruebas una vez, y el
> plugin recibe un **permiso temporal y revocable** (token). Ese token se
> guarda **solo en tu máquina** (archivo con permisos solo-dueño) y solo se
> comunica con `tone3000.com`. No hay telemetría ni claves sensibles que
> manejar. Puedes revocar el acceso desde tu cuenta tone3000 cuando quieras.

---

## Cómo funciona la autenticación de tone3000

tone3000 ofrece dos credenciales:

| Clave | Formato | ¿Segura en el dispositivo del usuario? |
|---|---|---|
| **Publishable key** (`client_id`) | `t3k_pub_…` | ✅ Sí — *"Safe to include in client-side code… browser environments."* Solo identifica la app en OAuth; sola no descarga nada. |
| **Secret key** | `t3k_cs_…` | ❌ No — *"Treat it like a database password — never embed it in… any environment that reaches a user's device."* |

**Rig Builder usa el flujo recomendado: OAuth 2.0 + PKCE con la publishable
key.** El usuario nunca maneja un secret. (El campo para pegar un secret se
quitó de la interfaz en la v1.1.0; el backend aún lo entiende para usos
avanzados server-to-server, pero no se muestra.)

## El flujo de login (paso a paso)

1. Pulsas **Settings → Connect with tone3000**.
2. Se abre tone3000 en tu **navegador del sistema**; inicias sesión y apruebas.
3. tone3000 redirige de vuelta al servidor local del plugin
   (`http://127.0.0.1:…`), que canjea el código por un **access token** +
   **refresh token**.
4. El plugin usa esos tokens para buscar/descargar tonos, y los **refresca
   solo** cuando expiran.

## Por qué es seguro (medidas concretas)

- **El código de autorización nunca sale de tu máquina:** el `redirect_uri`
  está restringido por código a `127.0.0.1`/`localhost` — aunque alguien
  manipulara el parámetro, no puede desviarlo a otro servidor.
- **PKCE (S256):** si un proceso local interceptara el código, no sirve sin el
  `code_verifier`, que nunca sale del backend del plugin.
- **Protección CSRF:** se valida un `state` aleatorio en el callback.
- **Tokens en reposo:** se guardan en `rig_builder_settings.json` con permisos
  **solo-dueño (0600)** — otros usuarios del sistema no pueden leerlos.
- **Solo lectura/descarga:** el plugin solo hace GET (buscar, listar,
  descargar). No modifica ni borra nada de tu cuenta.
- **Revocable:** quitas el acceso desde tu cuenta tone3000 cuando quieras, sin
  afectar nada más.
- **Sin telemetría:** el único host externo en todo el código es
  `www.tone3000.com`.

## Verificación (para testers técnicos)

El código no está ofuscado. Lo puedes confirmar tú mismo:

```bash
# Único host externo en todo el código:
grep -rEno "https?://[a-zA-Z0-9._/-]+" routes.py tone3000_client.py screen.js \
  | sed -E 's#(https?://[^/]+).*#\1#' | sort | uniq -c
# → solo aparece https://www.tone3000.com
```

- Flujo OAuth + refresco de tokens: `tone3000_client.py`
  (`build_authorize_url`, `exchange_code`, `_refresh_token`, `_bearer`).
- Rutas del login + restricción loopback: `routes.py`
  (`_safe_loopback_redirect`, `/oauth/start`, `/oauth/callback`).
- Permisos 0600 del archivo de settings: `routes.py` (`_write_settings_file`).

## Nota para producción

`DEFAULT_PUBLISHABLE_KEY` en `tone3000_client.py` es hoy la publishable key
personal del desarrollador (placeholder). Antes de una distribución amplia
conviene **registrar una app OAuth dedicada "Rig Builder"** en tone3000 y
**registrar el `redirect_uri` loopback** para esa app.
