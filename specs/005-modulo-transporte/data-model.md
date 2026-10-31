# Data Model: Módulo Transporte

**Feature**: `005-modulo-transporte` | **Date**: 2026-10-30

## Vehículo: Firestore `events/{EVENT_ID}/vehicles/{vehicleId}`

El id lo genera Firestore. Solo escribe el backend; las reglas de Firestore siguen cerradas a los
clientes.

| Campo | Tipo | Regla |
|-------|------|-------|
| `name` | string | De 1 a 60 caracteres tras recortar espacios |
| `driverId` | string | `guestId` de un confirmado que no conduce otro vehículo ni va de pasajero |
| `departure` | string | `bogota` o `tocancipa` |
| `seats` | number | Entero de 1 a 24; nunca menor que `len(passengers)` |
| `departureTime` | string | Opcional; `HH:MM` de 24 horas, en hora de Colombia |
| `note` | string | Opcional; hasta 200 caracteres |
| `passengers` | array\<string\> | `guestId` de confirmados. Ninguno aparece en otro vehículo ni es conductor. Largo ≤ `seats` |
| `updatedAt` | timestamp | Hora del servidor de la última escritura |
| `updatedBy` | string | `guestId` de quien hizo la última escritura |

**Invariantes de toda la colección** (los garantiza `applyTransport` dentro de una transacción
que lee todos los vehículos):

- Cada `guestId` aparece como mucho una vez entre todos los `driverId` y `passengers` (FR-005).
- `len(passengers) ≤ seats` en cada vehículo (FR-006 y FR-007).

## Vistas calculadas (no se guardan)

- **Cupos libres**: `seats − len(passengers)`.
- **Confirmados sin transporte**: los confirmados del catálogo que no son conductores ni pasajeros
  (FR-010).

## Transiciones de un invitado confirmado

```text
sin transporte ──reservar en V──► pasajero de V ──reservar en W──► pasajero de W (cambio)
      ▲                                │
      └───────── liberar / eliminar V ─┘
sin transporte ──registrar vehículo como conductor──► conductor (no puede reservar)
conductor ──eliminar su vehículo o cambiar de conductor──► sin transporte
```
