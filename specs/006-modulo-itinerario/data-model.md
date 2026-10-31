# Data Model: Módulo Itinerario

**Feature**: `006-modulo-itinerario` | **Date**: 2026-10-30

## Actividad: Firestore `events/{EVENT_ID}/activities/{activityId}`

El id lo genera Firestore. Solo escribe el backend; las reglas de Firestore siguen cerradas a los
clientes.

| Campo | Tipo | Regla |
|-------|------|-------|
| `title` | string | De 1 a 80 caracteres tras recortar espacios |
| `date` | string | `AAAA-MM-DD`, uno de los días del evento (FR-004) |
| `startTime` | string | `HH:MM` de 24 horas, hora de Colombia |
| `endTime` | string | Opcional; `HH:MM`. Sin `endsNextDay`, posterior a `startTime` (FR-005) |
| `endsNextDay` | bool | Solo con `endTime`. El fin cae el día siguiente; la duración es como máximo 24 h |
| `place` | string | Opcional; hasta 80 caracteres |
| `description` | string | Opcional; hasta 500 caracteres |
| `updatedAt` | timestamp | Hora del servidor de la última escritura |
| `updatedBy` | string | `guestId` de quien hizo la última escritura |

**Duración** con `endsNextDay`: `(24 h − startTime) + endTime`, que MUST ser ≤ 24 h. Ejemplo: 20:00
a 01:00 del día siguiente = 5 h. 22:00 a 23:00 del día siguiente = 25 h, que se rechaza.

## Días válidos (calculados, no se guardan)

`eventDays(e)` = fechas de calendario de Colombia desde `startsAt` hasta `endsAt`, ambas incluidas.
Para el evento de la V1: `2026-12-23`, `2026-12-24`, `2026-12-25` y `2026-12-26`.

## Orden (FR-002)

`date` → `startTime` → título sin acentos ni mayúsculas → `id`.

## Errores de dominio

| Error | Cuándo | HTTP |
|-------|--------|------|
| `errInvalidActivity` | título vacío o largo, lugar o descripción largos, hora mal formada, `endsNextDay` sin `endTime` | `400 invalid_activity` |
| `errActivityOutOfRange` | `date` no está en los días del evento | `400 activity_out_of_range` |
| `errActivityTimeOrder` | fin ≤ inicio sin `endsNextDay`, o más de 24 h con `endsNextDay` | `400 activity_time_order` |
| `errActivityNotFound` | editar o eliminar un id que no existe | `404 activity_not_found` |
