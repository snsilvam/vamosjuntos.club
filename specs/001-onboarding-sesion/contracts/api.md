# API Contract: Onboarding y Sesión

**Feature**: `001-onboarding-sesion` | **Base**: `/api` (mismo dominio que el sitio; Firebase
Hosting reescribe `/api/**` hacia Cloud Run)

## Convenciones

- Cuerpos en JSON (`Content-Type: application/json`), con fechas en RFC 3339 y offset `-05:00`.
- **Errores**: `{"error":{"code":"<código>"}}`. Para `429` se agrega
  `"retryAfterSeconds": <n>` y también el encabezado `Retry-After`. El frontend traduce cada
  `code` a un mensaje en español (ver tabla al final); el backend nunca envía textos técnicos.
- **Caché**: toda respuesta de `/api` lleva `Cache-Control: no-store`.
- **`X-Device-Id`**: UUID que genera el frontend. Es obligatorio en toda operación que recibe un
  código; si falta, la respuesta es `400 missing_device_id`.
- **Sesión**: cookie `__session` (`HttpOnly; Secure; SameSite=Lax; Path=/`). Las rutas marcadas
  🔒 la exigen: sin ella responden `401 unauthenticated`, y con la sesión de alguien que ya no
  está invitado o confirmado responden `403 not_invited`.

### Tipos compartidos

```jsonc
// Guest
{ "id": "papa-de-lore", "name": "papá de lore" }

// EventSummary
{
  "name": "Paseo Familiar Diciembre 2026",
  "startsAt": "2026-12-23T14:00:00-05:00",
  "endsAt": "2026-12-26T11:00:00-05:00",
  "destination": "Acacías, Meta, Colombia",
  "departurePlaces": ["Bogotá", "Tocancipá"]
}
```

---

## `GET /api/onboarding/guests`

Lista de invitados pendientes, en orden alfabético sin distinguir tildes (FR-002).

| Status | Cuerpo |
|--------|--------|
| 200 | `{"guests": [Guest, ...]}` (puede ser vacía) |
| 404 | `event_not_found` |

## `POST /api/onboarding/verify`

Valida el código del invitado elegido de la lista (FR-003, FR-005). No cambia estados.

Request: `{"guestId": "carmencita", "code": "k7m2qa"}` + `X-Device-Id`

| Status | Cuerpo |
|--------|--------|
| 200 | `{"event": EventSummary}` |
| 400 | `invalid_request`, `missing_device_id` |
| 401 | `invalid_code` (cuenta para el bloqueo) |
| 404 | `event_not_found`, `guest_not_found` |
| 409 | `already_responded` (el invitado ya no está pendiente) |
| 429 | `too_many_attempts` |

## `POST /api/onboarding/respond`

Registra la respuesta (FR-006, FR-007). Vuelve a validar el código, porque el servidor no guarda
estado entre pasos. Se acepta si el invitado está pendiente o en "No asistirá" (el segundo caso
viene del reingreso).

Request: `{"guestId": "carmencita", "code": "k7m2qa", "attending": true}` + `X-Device-Id`

| Status | Cuerpo |
|--------|--------|
| 200 (`attending: true`) | `{"status": "confirmed", "guest": Guest}` + `Set-Cookie: __session=...` |
| 200 (`attending: false`) | `{"status": "declined"}` (sin cookie) |
| 400 | `invalid_request`, `missing_device_id` |
| 401 | `invalid_code` |
| 404 | `event_not_found`, `guest_not_found` |
| 409 | `already_confirmed` (intenta declinar estando confirmado) |
| 429 | `too_many_attempts` |

Confirmar a alguien que ya está confirmado devuelve `200` con una nueva cookie: es idempotente y
cubre el caso de dos dispositivos.

## `POST /api/session/reentry`

Reingreso solo con código (FR-011).

Request: `{"code": "k7m2qa"}` + `X-Device-Id`

| Status | Cuerpo |
|--------|--------|
| 200 | `{"status": "confirmed", "guest": Guest}` + `Set-Cookie: __session=...` |
| 200 | `{"status": "declined", "guest": Guest, "event": EventSummary}`: el frontend vuelve a preguntar y llama a `/api/onboarding/respond` |
| 400 | `invalid_request`, `missing_device_id` |
| 401 | `invalid_code` (no pertenece a ningún invitado) |
| 404 | `event_not_found` |
| 409 | `pending_use_list` (el invitado aún no ha respondido) |
| 429 | `too_many_attempts` |

## `GET /api/session` 🔒

Quién soy. La guarda de rutas del frontend lo usa al cargar.

| Status | Cuerpo |
|--------|--------|
| 200 | `{"guest": Guest}` |
| 401 | `unauthenticated` |
| 403 | `not_invited` |

## `DELETE /api/session`

Cierra la sesión (FR-012). Funciona aunque no haya sesión.

| Status | Cuerpo |
|--------|--------|
| 204 | sin cuerpo + `Set-Cookie: __session=; Max-Age=0` |

## `POST /api/guests/{guestId}/confirmation` 🔒

Confirma a otro invitado pendiente con su código (FR-014). No cambia la sesión actual y registra
`respondedBy` igual al invitado de la sesión.

Request: `{"code": "x9p4tz"}` + `X-Device-Id`

| Status | Cuerpo |
|--------|--------|
| 200 | `{"status": "confirmed", "guest": Guest}` |
| 400 | `invalid_request`, `missing_device_id`, `invalid_code` (cuenta para el bloqueo; es `400` y no `401` para no cerrar la sesión de quien llama) |
| 401 / 403 | `unauthenticated` / `not_invited` |
| 404 | `event_not_found`, `guest_not_found` |
| 409 | `already_responded` |
| 429 | `too_many_attempts` |

---

## Mensajes de la UI por código de error (Principio V)

| `code` | Mensaje |
|--------|---------|
| `invalid_code` (desde la lista) | Ese código no coincide con tu nombre. Revísalo e intenta de nuevo. |
| `invalid_code` (reingreso) | No encontramos ese código. Revísalo e intenta de nuevo. |
| `invalid_code` (confirmar a otro) | Ese código no corresponde a {nombre}. |
| `too_many_attempts` | Demasiados intentos. Espera {n} minutos e inténtalo otra vez. |
| `event_not_found` | El paseo aún no está disponible. Intenta más tarde. |
| `guest_not_found` | No encontramos a esa persona en la lista de invitados. |
| `already_responded` | Esta persona ya respondió la invitación. |
| `already_confirmed` | Ya habías confirmado tu asistencia. |
| `pending_use_list` | Aún no has confirmado. Busca tu nombre en la lista. |
| `unauthenticated`, `not_invited` | (no se muestra: la app vuelve al onboarding) |
| `invalid_request`, `missing_device_id`, red o `5xx` | Algo salió mal. Intenta de nuevo en un momento. |
