# API Contract: Banner de Cuenta Regresiva

**Feature**: `004-banner-cuenta-regresiva` | **Base**: `/api`

Se suma a [contracts/api.md de la 008](../../008-autenticacion-firebase/contracts/api.md), con sus
mismas convenciones: `Authorization: Bearer`, `Cache-Control: no-store`, cuerpo de error
`{"error":{"code"}}` y la marca 🔒C. No cambia ninguna ruta existente y no depende de
`GET /api/event` (003).

## `GET /api/countdown` 🔒C

Días que faltan y tramo de la cuenta regresiva, calculados con el reloj del backend en hora de
Colombia (FR-002, FR-003, FR-007). Detalle de los campos y los tramos en
[data-model.md](../data-model.md).

Response `200`:

```jsonc
{
  "phase": "far",               // far | near | lastWeek | tomorrow | today | ongoing | finished
  "daysLeft": 54,               // días calendario de Colombia hasta el inicio; 0 desde el día del evento
  "secondsToNextChange": 50400, // ausente cuando phase = "finished"
  "secondsToStart": 4635761     // segundos hasta el inicio; ausente desde el inicio (enmienda 2026-10-30)
}
```

| Status | Cuerpo |
|--------|--------|
| 200 | `Countdown` |
| 401 | `unauthenticated` |
| 403 | `not_linked` o `not_invited` |
| 404 | `event_not_found` (el evento no existe o le falta un dato obligatorio) |
| 500 | `internal` |

La respuesta no lleva fechas absolutas ni datos de invitados. El nombre del banner sale de
`GET /api/me`.

## Uso en el frontend

- Se pide al montar el banner, al vencer `secondsToNextChange` y al volver la pestaña a visible
  (research R3).
- Nunca se compara con `Date.now()`: `secondsToNextChange` se usa solo como duración de un
  `setTimeout`, y `secondsToStart` como punto de partida del reloj, que descuenta el tiempo
  transcurrido medido con `performance.now()` (data-model, "Reloj").

## Mensajes en la UI

| Caso | Texto |
|------|-------|
| Cargando | "¡Hola, {nombre}!" · "Contando los días…" |
| `404`, `500` o sin red en la primera carga | "¡Hola, {nombre}!" · "La cuenta regresiva no está disponible por ahora." |
| Falla una recarga posterior | ninguno: el banner conserva lo último que mostró |
| `403` | ninguno en el banner: el manejador global vuelve a pedir `/api/me` (008) |
| `401` | ninguno en el banner: el cliente renueva el token o cierra la sesión (008) |
