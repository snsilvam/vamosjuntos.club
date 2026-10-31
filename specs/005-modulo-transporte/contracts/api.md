# API Contract: Módulo Transporte

**Feature**: `005-modulo-transporte` | **Base**: `/api`

Convenciones de la [spec 008](../../008-autenticacion-firebase/contracts/api.md):
- `Authorization: Bearer <ID token>` en cada llamada;
- errores con la forma `{"error":{"code"}}`;
- `Cache-Control: no-store`.

**Todas las rutas son 🔒C**: exigen una cuenta vinculada y confirmada. Si no la hay, responden
`401 unauthenticated`, `403 not_linked` o `403 not_invited`.

## Tipos

```jsonc
// Vehicle
{
  "id": "aB3…",
  "name": "La camioneta de Orlando",
  "driver": Guest,
  "departure": "bogota" | "tocancipa",
  "seats": 6,
  "free": 4,
  "passengers": [Guest, ...],
  "departureTime": "06:30",   // opcional
  "note": "Salimos del parque" // opcional
}

// VehicleInput
{ "name": "...", "driverId": "guestId", "departure": "bogota", "seats": 6,
  "departureTime": "06:30", "note": "..." }
```

## `GET /api/transport`

| Status | Cuerpo |
|--------|--------|
| 200 | `{"vehicles": [Vehicle], "withoutTransport": [Guest], "confirmed": [Guest]}` |

- `vehicles` se ordena por `departure` y luego por `name`.
- `withoutTransport` y `confirmed` van en orden alfabético.

## `POST /api/transport/vehicles`

> **Nota (2026-10-30)**: crear, editar y reservar responden con el estado completo del módulo, no
> con un solo `Vehicle`, para que la pantalla no haga otra llamada (desviación registrada en
> tasks.md). Eliminar y liberar responden `204`.

Registra un vehículo (FR-003). Request: `VehicleInput`.

| Status | Cuerpo |
|--------|--------|
| 201 | El mismo cuerpo de `GET /api/transport` (ver nota) |
| 400 | `invalid_request`, `invalid_vehicle` (nombre, salida, cupos, hora o nota fuera de las reglas) |
| 409 | `driver_unavailable` (el conductor no es confirmado, ya conduce o va como pasajero) |

## `PUT /api/transport/vehicles/{id}`

Edita un vehículo (FR-007, FR-009). Request: `VehicleInput`.

| Status | Cuerpo |
|--------|--------|
| 200 | El mismo cuerpo de `GET /api/transport` |
| 400 | `invalid_request`, `invalid_vehicle` |
| 404 | `vehicle_not_found` |
| 409 | `driver_unavailable`, `seats_below_passengers` |

## `DELETE /api/transport/vehicles/{id}`

Elimina el vehículo; sus pasajeros quedan sin transporte (FR-008).

| Status | Cuerpo |
|--------|--------|
| 204 | Sin cuerpo |
| 404 | `vehicle_not_found` |

## `POST /api/transport/vehicles/{id}/passengers`

Reserva un cupo para `guestId` (FR-013). Si ya iba en otro vehículo, lo cambia de vehículo.
Request: `{"guestId": "…"}`.

| Status | Cuerpo |
|--------|--------|
| 200 | El mismo cuerpo de `GET /api/transport` |
| 400 | `invalid_request` |
| 404 | `vehicle_not_found` |
| 409 | `vehicle_full`; `guest_is_driver`; `guest_not_confirmed` |

## `DELETE /api/transport/vehicles/{id}/passengers/{guestId}`

Libera el cupo (FR-013).

| Status | Cuerpo |
|--------|--------|
| 204 | Sin cuerpo |
| 404 | `vehicle_not_found`, `passenger_not_found` |

---

## Mensajes de la UI

| Código | Mensaje |
|--------|---------|
| `invalid_vehicle` | Revisa los datos: nombre, lugar de salida y entre 1 y 24 cupos. |
| `driver_unavailable` | Esa persona ya tiene lugar en otro vehículo o no ha confirmado. |
| `seats_below_passengers` | No puedes dejar menos cupos que pasajeros. Libera algún cupo primero. |
| `vehicle_full` | Este vehículo ya no tiene cupos. |
| `guest_is_driver` | Esa persona va manejando su propio vehículo. |
| `guest_not_confirmed` | Solo se puede reservar para invitados que confirmaron. |
| `vehicle_not_found` | Ese vehículo ya no existe. Actualiza la página. |
| `passenger_not_found` | Esa persona ya no va en este vehículo. |
