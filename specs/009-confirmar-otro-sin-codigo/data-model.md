# Data Model: Confirmar a otro invitado sin su código

**Feature**: `009-confirmar-otro-sin-codigo` | **Date**: 2026-11-06

> Base de Firestore: `vamosjuntos-club-db`. No hay colecciones ni campos nuevos. Las reglas de
> seguridad siguen negando todo acceso desde clientes.

## Sin cambios

- **Invitado**: `events/{EVENT_ID}/users/{guestId}`, con `Nombre` y `código` (001).
- **Vínculo**: `events/{EVENT_ID}/links/{guestId}`, con `uid`, `email` y `linkedAt` (008). Que
  exista el documento significa que el invitado tiene cuenta.
- **Respuesta de asistencia**: `events/{EVENT_ID}/attendance/{guestId}`, con `status`,
  `respondedAt` y `respondedBy`. Al confirmar a otro, `respondedBy` es el `guestId` de quien
  confirma, como antes.

## Invitados confirmables (calculado, no se guarda)

Un invitado del catálogo es confirmable por otro cuando:

- no tiene documento en `attendance` (pendiente), y
- no tiene documento en `links` (sin cuenta).

Se listan en el orden del catálogo (`Catalog.Sorted`, alfabético). La lista solo lleva `id` y
`name`.

## Transición "confirmar a otro" (reemplaza la fila de la 001)

Corre en una transacción que lee `links/{guestId}` y `attendance/{guestId}` (research R2).

| Respuesta actual | ¿Vinculado? | Resultado | HTTP |
|------------------|-------------|-----------|------|
| pendiente | no | `confirmed`, con `respondedAt` y `respondedBy` | `200` |
| pendiente | sí | rechazado | `409 guest_already_linked` |
| `confirmed` o `declined` | da igual | rechazado | `409 already_responded` |

Las demás transiciones de la 001 (confirmar, declinar, cambiar de opinión) no cambian.

## Cambia: intentos de código, `loginAttempts/{uid}`

Mismos campos y misma regla (5 fallos → 15 minutos), pero solo cuentan los códigos de
`POST /api/onboarding/link`. Confirmar a otro ya no lee ni compara códigos (FR-007).

## Deshacer una confirmación por error (organizador)

En la consola, borrar `events/{EVENT_ID}/attendance/{guestId}`. La persona vuelve a pendiente y a
las dos listas.
