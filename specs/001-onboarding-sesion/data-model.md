# Data Model: Onboarding y Sesión

**Feature**: `001-onboarding-sesion` | **Date**: 2026-10-30

> Base de Firestore: `FIRESTORE_DATABASE` (en `vamosjuntos-club` es `vamosjuntos-club-db`).

## Evento — Firestore `events/{EVENT_ID}`

Lo crea el organizador a mano en la consola de Firestore. `EVENT_ID` llega por variable de
entorno (sugerido: `paseo-familiar-diciembre-2026`). La app solo lo lee.

| Campo | Tipo | Ejemplo | Regla |
|-------|------|---------|-------|
| `name` | string | "Paseo Familiar Diciembre 2026" | obligatorio |
| `startsAt` | timestamp | 2026-12-23 19:00:00 UTC (2:00 p. m. COT) | obligatorio |
| `endsAt` | timestamp | 2026-12-26 16:00:00 UTC (11:00 a. m. COT) | obligatorio, > `startsAt` |
| `destination` | string | "Acacías, Meta, Colombia" | obligatorio |
| `departurePlaces` | array\<string\> | ["Bogotá", "Tocancipá"] | obligatorio, ≥ 1 |
| `capacity` | number | 25 | obligatorio, ≤ 25 |

Si el documento no existe o le falta un campo obligatorio, las operaciones de onboarding
responden `404 event_not_found` (FR-004).

## Invitado — Firestore `events/{EVENT_ID}/users/{guestId}`

> Enmienda de la constitución v2.0.0: antes venía de la variable de entorno `GUESTS_JSON`.

Lo crea y edita el organizador a mano en la consola de Firestore, un documento por invitado. La
app solo lo lee.

| Campo / atributo | Tipo | Regla |
|------------------|------|-------|
| id del documento | string | Es el `guestId` de la app (sesión, `attendance`, rutas). Firestore lo genera; no cambia aunque se edite el nombre |
| `Nombre` | string | No vacío tras recortar espacios. Es el nombre visible en la lista |
| `código` | string | Al menos 6 caracteres tras recortar espacios; único sin distinguir mayúsculas y minúsculas (FR-016) |

Cualquier otro campo del documento (p. ej. `confirmado`) se ignora: la asistencia vive en
`attendance`.

**Lectura**: el backend lee la subcolección completa (≤ 25 documentos) en cada request que usa
el catálogo, para que los cambios del organizador apliquen de inmediato (FR-001). Luego valida:

- Al menos 1 invitado y como máximo 25 (FR-013).
- Ningún nombre vacío, ningún código corto y ningún código repetido (FR-016).

**Si la validación falla**: al arrancar, el proceso termina con error y Cloud Run no publica la
revisión. Con el servicio ya en marcha, la operación responde `500 internal_error` y el motivo
queda en el log. Además, al arrancar el catálogo debe caber en `capacity` del evento, si existe.

**Comparación de códigos** (FR-003): `strings.EqualFold(strings.TrimSpace(input), code)`.

**Protección de los códigos**:
- Las reglas de Firestore de `vamosjuntos-club-db` niegan todo acceso desde clientes
  (`allow read, write: if false`); solo el backend los lee, con su cuenta de servicio.
- Nunca salen en respuestas ni en logs; los mensajes de error nombran invitados, no códigos.

## Respuesta de asistencia — Firestore `events/{EVENT_ID}/attendance/{guestId}`

| Campo | Tipo | Regla |
|-------|------|-------|
| `status` | string | `"confirmed"` o `"declined"` |
| `respondedAt` | timestamp | hora del servidor al escribir |
| `respondedBy` | string | `guestId` de quien registró la respuesta (el mismo invitado u otro, FR-014) |

Si el documento no existe, el invitado está **pendiente**.

### Transiciones de estado

Las decide una función pura, `nextStatus(current, action)`; la transacción de Firestore solo lee,
aplica esa función y escribe.

```text
              confirmar (propio, FR-006 / por otro, FR-014)
  pendiente ───────────────────────────────────────────────► confirmado
      │                                                         ▲
      │ no asistirá (FR-007)                                    │ confirmar vía reingreso
      ▼                                                         │ (FR-011)
  no asistirá ──────────────────────────────────────────────────┘
```

| Estado actual | Acción | Resultado | HTTP |
|---------------|--------|-----------|------|
| pendiente | confirmar | confirmado | 200 |
| pendiente | declinar | no asistirá | 200 |
| no asistirá | confirmar (vía reingreso) | confirmado | 200 |
| no asistirá | declinar | sin cambio | 200 |
| confirmado | confirmar | sin cambio (idempotente) | 200 |
| confirmado | declinar | rechazado | 409 `already_confirmed` |
| no pendiente | confirmar a otro (FR-014) | rechazado | 409 `already_responded` |

En la V1 no se puede pasar de confirmado a no asistirá; no está en el spec.

## Intentos de código — Firestore `loginAttempts/{deviceId}`

| Campo | Tipo | Regla |
|-------|------|-------|
| `failures` | number | fallos seguidos; vuelve a 0 con un acierto |
| `blockedUntil` | timestamp \| null | ahora + 15 min al llegar a 5 fallos (FR-010) |

La regla también es una función pura, `registerAttempt(state, ok, now)`, que devuelve el nuevo
estado y si el dispositivo queda bloqueado.

## Sesión — cookie `__session` (no se guarda en Firestore)

| Atributo | Regla |
|----------|-------|
| `g` | `guestId` del invitado confirmado |
| `exp` | `event.endsAt + 7 días` (FR-009), en segundos Unix |
| firma | HMAC-SHA256 con `SESSION_KEY` |

**Verificación en cada request protegido** (FR-008):

1. Si la firma es inválida o el token está vencido: `401 unauthenticated`.
2. Si el `guestId` ya no está en el catálogo o su estado no es `confirmed`:
   `403 not_invited`.

## Resumen de asistencia (lo usa la spec 003)

Se calcula al momento, sin guardarlo:
`invitados = len(catálogo)`, `confirmados` y `noAsistirán` se cuentan en `attendance`, y
`pendientes = invitados − confirmados − noAsistirán`.
