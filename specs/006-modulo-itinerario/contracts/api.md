# API Contract: Módulo Itinerario

**Feature**: `006-modulo-itinerario` | **Base**: `/api`

Convenciones de la [spec 008](../../008-autenticacion-firebase/contracts/api.md):
- `Authorization: Bearer <ID token>` en cada llamada;
- errores con la forma `{"error":{"code"}}`;
- `Cache-Control: no-store`.

**Todas las rutas son 🔒C**: exigen una cuenta vinculada y confirmada. Si no la hay, responden
`401 unauthenticated`, `403 not_linked` o `403 not_invited`.

## Tipos

```jsonc
// Activity
{
  "id": "aB3…",
  "title": "Piscina",
  "date": "2026-12-24",
  "startTime": "10:00",
  "endTime": "12:00",          // opcional
  "endsNextDay": false,        // ausente si es false
  "place": "Hotel",            // opcional
  "description": "Llevar toalla" // opcional
}

// ActivityInput
{ "title": "...", "date": "2026-12-24", "startTime": "10:00", "endTime": "12:00",
  "endsNextDay": false, "place": "...", "description": "..." }

// Itinerary
{ "days": ["2026-12-23", "2026-12-24", "2026-12-25", "2026-12-26"], "activities": [Activity] }
```

## `GET /api/itinerary`

| Status | Cuerpo |
|--------|--------|
| 200 | `Itinerary`; `activities` en el orden de [data-model.md](../data-model.md) |
| 404 | `event_not_found` |

## `POST /api/itinerary/activities`

Agrega una actividad (US2). Request: `ActivityInput`.

| Status | Cuerpo |
|--------|--------|
| 201 | `Itinerary` |
| 400 | `invalid_request`, `invalid_activity`, `activity_out_of_range`, `activity_time_order` |

## `PUT /api/itinerary/activities/{id}`

Edita una actividad (US3). Request: `ActivityInput`.

| Status | Cuerpo |
|--------|--------|
| 200 | `Itinerary` |
| 400 | `invalid_request`, `invalid_activity`, `activity_out_of_range`, `activity_time_order` |
| 404 | `activity_not_found` |

## `DELETE /api/itinerary/activities/{id}`

Elimina una actividad (US3). La confirmación previa (FR-008) es de la UI.

| Status | Cuerpo |
|--------|--------|
| 204 | — |
| 404 | `activity_not_found` |

## Mensajes en la UI (FR-009)

| Código | Mensaje |
|--------|---------|
| `activity_time_order` | "La hora de fin debe ser después de la hora de inicio." |
| `activity_out_of_range` | "La actividad debe estar dentro de los días del paseo." |
| `invalid_activity` | "Revisa los datos de la actividad: el título es obligatorio y los textos no pueden ser tan largos." |
| `activity_not_found` | "Esa actividad ya no existe. Actualizamos el itinerario." |
| Itinerario vacío | "El itinerario aún no tiene actividades." |
| Error al cargar | "No pudimos cargar el itinerario. Intenta de nuevo en un momento." |
