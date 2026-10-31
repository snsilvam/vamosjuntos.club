# API Contract: Card "Próximo viaje"

**Feature**: `003-card-proximo-viaje` | **Base**: `/api`

Se suma a [contracts/api.md de la 008](../../008-autenticacion-firebase/contracts/api.md), con sus
mismas convenciones: `Authorization: Bearer`, `Cache-Control: no-store`, cuerpo de error
`{"error":{"code"}}` y la marca 🔒C. No cambia ninguna ruta existente.

## `GET /api/event` 🔒C

Datos del viaje y conteos de asistencia para la card de la pantalla principal (FR-002, FR-004).

Response `200`:

```jsonc
{
  "event": {                     // EventSummary (008)
    "name": "Paseo Familiar Diciembre 2026",
    "startsAt": "2026-12-23T14:00:00-05:00",
    "endsAt": "2026-12-26T11:00:00-05:00",
    "destination": "Acacías, Meta",
    "departurePlaces": ["Bogotá", "Tocancipá"]
  },
  "attendance": {                // AttendanceSummary
    "invited": 22,
    "confirmed": 5,
    "declined": 2,
    "pending": 15
  }
}
```

| Status | Cuerpo |
|--------|--------|
| 200 | `{"event": EventSummary, "attendance": AttendanceSummary}` |
| 401 | `unauthenticated` |
| 403 | `not_linked` o `not_invited` |
| 404 | `event_not_found` (el evento no existe o le falta un dato obligatorio) |
| 500 | `internal` |

La respuesta no lleva nombres de invitados, ids ni códigos (FR-007, Principio II).

## Mensajes en la UI

| Caso | Texto |
|------|-------|
| `404`, `500` o sin red | "La información del viaje no está disponible por ahora." |
| `403` | ninguno en la card: el manejador global vuelve a pedir `/api/me` (008) |
| `401` | ninguno en la card: el cliente renueva el token o cierra la sesión (008) |

Línea de conteos (research R5): `"{invited} invitados · {confirmed} confirmados · {declined} no
asistirán · {pending} pendientes"`, en singular cuando el número es 1.
