# Data Model: Card "Próximo viaje"

**Feature**: `003-card-proximo-viaje` | **Date**: 2026-10-30

La feature no crea colecciones ni escribe en Firestore. Solo lee lo que ya existe.

## Lecturas en Firestore (sin cambios)

| Ruta | Lo crea | Uso en esta feature |
|------|---------|---------------------|
| `events/{EVENT_ID}` | el responsable, a mano | nombre, lugares de salida, destino, inicio y fin |
| `events/{EVENT_ID}/users/{guestId}` | el responsable, a mano | tamaño del catálogo (`invited`) y qué respuestas cuentan |
| `events/{EVENT_ID}/attendance/{guestId}` | la app (001/008) | `confirmed` y `declined` |

## EventSummary (sin cambios)

Ya existe en `event.go` (001/008). Fechas en RFC 3339 con `-05:00`.

| Campo | Tipo | Origen |
|-------|------|--------|
| `name` | string | `events/{EVENT_ID}.name` |
| `startsAt` | string (RFC 3339) | `startsAt`, en hora de Colombia |
| `endsAt` | string (RFC 3339) | `endsAt`, en hora de Colombia |
| `destination` | string | `destination` |
| `departurePlaces` | string[] | `departurePlaces` |

Si el documento no existe o le falta un campo obligatorio, `EventStore.Get` devuelve
`errEventNotFound` (regla `Event.valid()` existente) y la API responde `404 event_not_found`.

## AttendanceSummary (nuevo)

Solo conteos (FR-007: sin nombres ni datos de otros invitados).

| Campo | Tipo | Regla |
|-------|------|-------|
| `invited` | int | `catalog.Len()` |
| `confirmed` | int | invitados del catálogo con estado `confirmed` |
| `declined` | int | invitados del catálogo con estado `declined` ("No asistirá") |
| `pending` | int | `invited − confirmed − declined` |

**Invariantes**:
- `confirmed + declined + pending = invited`.
- Cada campo es ≥ 0.
- Un documento de `attendance` cuyo `guestId` no está en el catálogo no cuenta (research R2).

**Ejemplo** (US2, escenario 1): catálogo de 22, con 5 `confirmed` y 2 `declined` →
`{invited: 22, confirmed: 5, declined: 2, pending: 15}`.

## Estados de la card (frontend)

| Estado | Se ve |
|--------|-------|
| `loading` | la card con un texto de carga |
| `ready` | datos del viaje y la línea de conteos |
| `unavailable` | "La información del viaje no está disponible por ahora." |
