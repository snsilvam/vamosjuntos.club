# Data Model: Banner de Cuenta Regresiva

**Feature**: `004-banner-cuenta-regresiva` | **Date**: 2026-10-30

La feature no crea colecciones ni escribe en Firestore. Solo lee el evento.

## Lecturas en Firestore (sin cambios)

| Ruta | Lo crea | Uso en esta feature |
|------|---------|---------------------|
| `events/{EVENT_ID}` | el responsable, a mano | `startsAt` y `endsAt` |

Si el documento no existe o le falta un campo obligatorio, `EventStore.Get` devuelve
`errEventNotFound` (regla `Event.valid()` existente) y la API responde `404 event_not_found`.

El nombre del invitado no se pide de nuevo: el frontend ya lo tiene de `GET /api/me`
(`me.guest.name`, estado `confirmed`).

## Countdown (nuevo, backend)

Resultado de la función pura `countdownAt(e Event, now time.Time)` (research R2).

| Campo | JSON | Tipo | Regla |
|-------|------|------|-------|
| `Phase` | `phase` | string | uno de los 7 tramos de la tabla de abajo |
| `DaysLeft` | `daysLeft` | int | días calendario de Colombia entre `now` y `StartsAt`; `0` en `today`, `ongoing` y `finished` |
| `SecondsToNextChange` | `secondsToNextChange` | int, opcional | segundos, redondeados hacia arriba, hasta la próxima medianoche de Colombia, `StartsAt` o `EndsAt`, el que llegue primero; ausente en `finished` |

**Invariantes**:
- `DaysLeft ≥ 0`.
- `DaysLeft = 1` si y solo si `Phase = tomorrow`.
- `SecondsToNextChange` está entre 1 y 86 400 cuando viene.
- `Phase = finished` si y solo si no viene `SecondsToNextChange`.

## Tramos (estado de la cuenta regresiva)

Se evalúan en este orden; gana la primera fila que se cumple.

| `phase` | Condición | Mensaje (spec) | Número grande | Emoji | Fondo (research R5) |
|---------|-----------|----------------|---------------|-------|---------------------|
| `finished` | `now ≥ EndsAt` | "¡Qué buen paseo! Gracias por venir" | no | ❤️ | `violet-700 → rose-700` |
| `ongoing` | `StartsAt ≤ now < EndsAt` | "¡Estamos de paseo!" | no | 🌴 | `emerald-700 → sky-700` |
| `today` | días = 0 | "¡Hoy es el día!" | no | 🎉 | `fuchsia-700 → red-700` |
| `tomorrow` | días = 1 | "¡Mañana nos vamos!" | sí: 1 | 🚌 | `red-700 → fuchsia-700` |
| `lastWeek` | días 2 a 7 | "¡Ya casi! Última semana" | sí | 🔥 (rebota) | `rose-700 → fuchsia-700` |
| `near` | días 8 a 30 | "¡Ve alistando la maleta!" | sí | 🧳 | `orange-700 → rose-700` |
| `far` | días > 30 | "¡Ya empieza la cuenta regresiva!" | sí | 🎒 | `orange-700 → red-700` |

En los tramos con número, el banner dice "{nombre}, falta 1 día para el paseo" o "{nombre},
faltan {N} días para el paseo" (FR-004, FR-008), con `{N}` como el texto más grande de la pantalla
(FR-005).

**Transiciones**: el tramo solo avanza hacia abajo en la tabla (`far` → … → `finished`). Cambia en
una medianoche de Colombia, en `StartsAt` o en `EndsAt`, que son los instantes que marca
`secondsToNextChange`. Durante `ongoing` las medianoches no cambian nada, pero se siguen marcando:
así el valor nunca pasa de 24 h y la regla es una sola.

**Ejemplos** (evento del 2026-12-23 14:00 al 2026-12-26 11:00, hora de Colombia):

| `now` (Colombia) | `phase` | `daysLeft` | `secondsToNextChange` |
|------------------|---------|------------|-----------------------|
| 2026-10-30 10:00 | `far` | 54 | 50 400 (hasta las 00:00 del 31) |
| 2026-11-22 23:59:59 | `far` | 31 | 1 |
| 2026-11-23 00:00:00 | `near` | 30 | 86 400 |
| 2026-12-15 12:00 | `near` | 8 | 43 200 |
| 2026-12-16 00:00 | `lastWeek` | 7 | 86 400 |
| 2026-12-22 08:00 | `tomorrow` | 1 | 57 600 |
| 2026-12-23 00:00 | `today` | 0 | 50 400 (hasta las 14:00) |
| 2026-12-23 14:00 | `ongoing` | 0 | 36 000 (hasta las 00:00 del 24) |
| 2026-12-26 10:59:59 | `ongoing` | 0 | 1 |
| 2026-12-26 11:00 | `finished` | 0 | — |

## Estados del banner (frontend)

| Estado | Se ve |
|--------|-------|
| `loading` | "¡Hola, {nombre}!" y "Contando los días…" |
| `ready` | el tramo con su mensaje, emoji, fondo y, si aplica, el número |
| `unavailable` | "¡Hola, {nombre}!" y "La cuenta regresiva no está disponible por ahora." |

Una recarga que falla (temporizador o regreso a la pestaña) no cambia un banner en `ready`
(research R3).
