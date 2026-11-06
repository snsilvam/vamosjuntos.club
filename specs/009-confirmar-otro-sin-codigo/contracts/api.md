# API Contract: Confirmar a otro invitado sin su código

**Feature**: `009-confirmar-otro-sin-codigo` | **Base**: `/api`

Cambia la ruta de confirmar a otro de [contracts/api.md de la 008](../../008-autenticacion-firebase/contracts/api.md)
y agrega su lista. Se mantienen las convenciones de la 008: `Authorization: Bearer <ID token>`,
`Cache-Control: no-store`, cuerpo de error `{"error":{"code"}}`, tipo `Guest` y la marca **🔒C**
(cuenta vinculada a un invitado del catálogo y confirmado; si no, `403 not_linked` o
`403 not_invited`).

---

## `GET /api/guests/confirmable` 🔒C (nueva)

Invitados que quien llama puede confirmar: pendientes y sin cuenta vinculada, en orden alfabético
(FR-002). Sin códigos ni datos de cuentas.

| Status | Cuerpo |
|--------|--------|
| 200 | `{"guests": [Guest, ...]}` (puede ser `[]`) |
| 401 | `unauthenticated` |
| 403 | `not_linked`; `not_invited` |
| 404 | `event_not_found` |

## `POST /api/guests/{guestId}/confirmation` 🔒C (cambia)

Confirma a otro invitado sin su código (FR-001, FR-004, FR-005). No crea vínculos ni cuenta para
el límite de intentos.

Request: sin cuerpo. Si llega uno (por ejemplo `{"code": "..."}` de la versión anterior de la
app), se ignora (research R4).

| Status | Cuerpo |
|--------|--------|
| 200 | `{"status": "confirmed", "guest": Guest}` |
| 401 / 403 | `unauthenticated` / `not_linked`, `not_invited` |
| 404 | `event_not_found`; `guest_not_found` |
| 409 | `already_responded` (ya respondió); `guest_already_linked` (tiene cuenta vinculada) |

**Sale** frente a la 008: `400 invalid_request`, `400 invalid_code` y `429 too_many_attempts`.

---

## Mensajes de la UI (Principio V)

Solo en la pantalla "Confirmar a otro invitado":

| Caso | Mensaje |
|------|---------|
| Pregunta previa | "¿Confirmar a {nombre}?" + "Quedará en la lista de confirmados del paseo. Si te equivocas, solo el organizador puede deshacerlo." |
| `200` | ¡{nombre} quedó confirmado! |
| `guest_already_linked` | {nombre} ya entró con su propia cuenta y responde desde ahí. |
| `already_responded` | Esta persona ya respondió la invitación. (sin cambios) |
| `guest_not_found` | No encontramos a esa persona en la lista de invitados. (sin cambios) |
| Lista vacía | No hay invitados pendientes sin cuenta. Quien ya entró con su cuenta responde desde ahí. |
| `event_not_found`, `invalid_request`, red o `5xx` | Los de la 008, sin cambios. |

Ante `404 guest_not_found` y ante cualquier `409`, la pantalla recarga la lista. Sale el mensaje
"Ese código no corresponde a {nombre}.".
