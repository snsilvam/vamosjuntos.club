# API Contract: Autenticación con Firebase

**Feature**: `008-autenticacion-firebase` | **Base**: `/api`, en el mismo dominio del sitio, porque
Firebase Hosting reescribe `/api/**` hacia Cloud Run.

Reemplaza [contracts/api.md de la 001](../../001-onboarding-sesion/contracts/api.md). Se mantienen
los tipos `Guest` y `EventSummary` y el cuerpo de error `{"error":{"code"}}`.

## Convenciones

- **Autenticación**: todas las rutas exigen `Authorization: Bearer <ID token de Firebase>`.
  - Sin encabezado, o con un token inválido o vencido → `401 unauthenticated`.
  - Ante un `401`, el frontend renueva el token una vez (`getIdToken(true)`) y reintenta. Si sigue
    en `401`, cierra la sesión del SDK y muestra el inicio de sesión.
- **Caché**: toda respuesta lleva `Cache-Control: no-store`.
- **Sin `X-Device-Id`**: el límite de intentos se cuenta por cuenta.
- **🔒C**: la ruta además exige una cuenta vinculada a un invitado del catálogo y confirmado.
  - Cuenta sin vincular → `403 not_linked`.
  - Invitado fuera del catálogo o no confirmado → `403 not_invited`.

### Tipos

```jsonc
// Me
{
  "status": "unlinked" | "pending" | "declined" | "confirmed",
  "guest": Guest,          // ausente si status = "unlinked"
  "event": EventSummary    // presente si status = "pending" o "declined" (para la pregunta)
}
```

---

## `GET /api/me`

Estado de la cuenta; la app decide a qué pantalla ir (FR-003, FR-007, FR-009).

| Status | Cuerpo |
|--------|--------|
| 200 | `Me` |
| 401 | `unauthenticated` |
| 403 | `not_invited` (vinculada a un invitado que ya no está en el catálogo) |
| 404 | `event_not_found` (solo si `status` sería `pending` o `declined`) |

## `GET /api/onboarding/guests`

Invitados pendientes en orden alfabético (FR-003). Exige sesión, pero no vínculo.

| Status | Cuerpo |
|--------|--------|
| 200 | `{"guests": [Guest, ...]}` |
| 401 | `unauthenticated` |
| 404 | `event_not_found` |

## `POST /api/onboarding/link`

Vincula la cuenta con un invitado (FR-004, FR-005). Cuenta para el límite de intentos.

Request: `{"guestId": "c442Avn…", "code": "K7M2QA"}` desde la lista, o `{"code": "K7M2QA"}`
desde "entrar con mi código".

| Status | Cuerpo |
|--------|--------|
| 200 | `Me` del invitado recién vinculado |
| 400 | `invalid_request`; `invalid_code` (el código no es de ese invitado, o no es de nadie) |
| 401 | `unauthenticated` |
| 404 | `event_not_found`; `guest_not_found` (el `guestId` no existe) |
| 409 | `account_already_linked`; `guest_already_linked` |
| 429 | `too_many_attempts` + `retryAfterSeconds` |

## `POST /api/onboarding/respond`

Respuesta de asistencia del propio invitado vinculado (FR-007, FR-008). Ya no lleva código.

Request: `{"attending": true}`

| Status | Cuerpo |
|--------|--------|
| 200 | `{"status": "confirmed" \| "declined", "guest": Guest}` |
| 400 | `invalid_request` |
| 401 | `unauthenticated` |
| 403 | `not_linked`; `not_invited` |
| 404 | `event_not_found` |
| 409 | `already_confirmed` (intenta declinar estando confirmado) |

## `POST /api/guests/{guestId}/confirmation` 🔒C

Confirma a otro invitado pendiente con su código (FR-013). No crea vínculos. Cuenta para el límite
de intentos. Sin cambios frente a la 001, salvo la autenticación.

Request: `{"code": "X9P4TZ"}`

| Status | Cuerpo |
|--------|--------|
| 200 | `{"status": "confirmed", "guest": Guest}` |
| 400 | `invalid_request`; `invalid_code` |
| 401 / 403 | `unauthenticated` / `not_linked`, `not_invited` |
| 404 | `event_not_found`; `guest_not_found` |
| 409 | `already_responded` |
| 429 | `too_many_attempts` |

## Rutas que desaparecen

`POST /api/onboarding/verify`, `POST /api/session/reentry`, `GET /api/session` y
`DELETE /api/session`. Cerrar sesión es `signOut()` del SDK.

---

## Mensajes de la UI (Principio V)

| Código | Mensaje |
|--------|---------|
| `invalid_code` (desde la lista) | Ese código no coincide con tu nombre. Revísalo e intenta de nuevo. |
| `invalid_code` ("entrar con mi código") | No encontramos ese código. Revísalo e intenta de nuevo. |
| `invalid_code` (confirmar a otro) | Ese código no corresponde a {nombre}. |
| `account_already_linked` | Tu cuenta ya está vinculada a otro invitado. |
| `guest_already_linked` | Esta persona ya entró con otra cuenta. Si eres tú y perdiste esa cuenta, escríbele al organizador. |
| `too_many_attempts` | Demasiados intentos. Espera {n} minutos e inténtalo otra vez. |
| `event_not_found` | El paseo aún no está disponible. Intenta más tarde. |
| `guest_not_found` | No encontramos a esa persona en la lista de invitados. |
| `already_responded` | Esta persona ya respondió la invitación. |
| `already_confirmed` | Ya habías confirmado tu asistencia. |
| `not_invited` | Ya no apareces en la lista de invitados. Escríbele al organizador. |
| `unauthenticated`, `not_linked` | (no se muestra: la app lleva al inicio de sesión o al onboarding) |
| `invalid_request`, red o `5xx` | Algo salió mal. Intenta de nuevo en un momento. |

### Errores del SDK de Firebase (FR-016)

| Código `auth/*` | Mensaje |
|-----------------|---------|
| `invalid-credential`, `wrong-password`, `user-not-found`, `invalid-email` | El correo o la contraseña no son correctos. |
| `email-already-in-use` | Ya existe una cuenta con ese correo. Entra con tu contraseña. |
| `weak-password` | La contraseña debe tener al menos 6 caracteres. |
| `popup-closed-by-user`, `cancelled-popup-request` | (no se muestra: el usuario cerró la ventana) |
| `popup-blocked` | Tu navegador bloqueó la ventana de Google. Permite las ventanas emergentes e intenta otra vez. |
| `account-exists-with-different-credential` | Ese correo ya entró con otro método. Usa el mismo con el que entraste antes. |
| `too-many-requests` | Demasiados intentos. Espera un momento e inténtalo otra vez. |
| `network-request-failed` | No hay conexión. Revisa tu internet e intenta otra vez. |
| Cualquier otro | Algo salió mal. Intenta de nuevo en un momento. |
