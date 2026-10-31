# Research: Módulo Transporte

**Feature**: `005-modulo-transporte` | **Date**: 2026-10-30

No hay puntos `NEEDS CLARIFICATION`; la spec ya trae sus clarificaciones (cualquiera edita todo y
se puede reservar para otro).

## R1. Pasajeros dentro del documento del vehículo

- **Decision**: `events/{EVENT_ID}/vehicles/{vehicleId}` guarda `passengers: [guestId]`. No hay
  una colección aparte de reservas.
- **Rationale**: un vehículo con sus pasajeros se lee y se escribe de una vez. Contar cupos libres
  es `seats - len(passengers)`. Eliminar el vehículo deja a sus pasajeros sin transporte sin pasos
  extra (FR-008).
- **Alternatives considered**: una colección `rides/{guestId}`, con el `guestId` como id, da la
  unicidad del pasajero gratis. Pero no cubre la del conductor, cuenta cupos con consultas y
  obliga a borrar en cascada al eliminar un vehículo.

## R2. Concurrencia: transacción sobre todos los vehículos y regla pura

- **Decision**: cada escritura (crear, editar, eliminar, reservar, liberar) corre en una
  transacción.
  1. Lee todos los documentos de `vehicles`.
  2. Llama a la regla pura `applyTransport(vehicles, cmd, confirmed)`, que devuelve qué escribir
     o borrar, o un error de dominio.
  3. Escribe.
- **Rationale**:
  - Al leer todos los vehículos, cualquier escritura concurrente en cualquiera de ellos invalida
    la transacción, y Firestore la reintenta con los datos nuevos.
  - Así se cumplen FR-005 (una persona en un solo lugar) y FR-006 (nunca más pasajeros que
    cupos), incluido "el último cupo al mismo tiempo" (SC-002).
  - Con unos 10 vehículos, leerlos todos cuesta lo mismo que leer uno.
  - Toda la lógica queda en una función pura, fácil de probar con una tabla de casos.
- **Alternatives considered**: transacciones que solo leen el vehículo tocado. No ven que la
  persona ya va en otro vehículo, así que FR-005 fallaría con reservas simultáneas.

## R3. Reglas de dominio (`applyTransport`)

- **Datos del vehículo**:
  - `name`: 1 a 60 caracteres tras recortar espacios.
  - `departure`: `bogota` o `tocancipa`.
  - `seats`: entero de 1 a 24.
  - `departureTime`: opcional, `HH:MM` de 24 horas, el día de salida en hora de Colombia.
  - `note`: opcional, hasta 200 caracteres.
- **Conductor**: un confirmado (FR-003, FR-011) que no conduce otro vehículo ni va como pasajero en
  ninguno (FR-005). En una edición, puede seguir siendo el mismo.
- **Editar**: `seats` no puede quedar por debajo de los pasajeros actuales (FR-007).
- **Reservar `{guestId}` en V**:
  1. el invitado debe ser confirmado;
  2. si conduce algún vehículo → `guest_is_driver`;
  3. si ya va en V → no hace nada;
  4. si V está lleno → `vehicle_full`;
  5. si va en otro vehículo, se le quita de ese y se agrega a V: es un cambio de vehículo, no un
     doble cupo.
- **Liberar `{guestId}` de V**: si no va en V → `passenger_not_found`.
- **Eliminar V**: se borra el documento y sus pasajeros quedan sin transporte.
- **Al listar**: pasajeros o conductores que ya no son confirmados del catálogo (porque el
  organizador los quitó) se omiten. Esos registros no se borran solos.

## R4. API y quién puede hacer qué

- **Decision**: todas las rutas exigen `requireGuest`.
  - Cualquier confirmado puede crear, editar y eliminar cualquier vehículo (FR-009).
  - También puede reservar o liberar para sí mismo o para cualquier otro confirmado (FR-013).
  - `GET /api/transport` devuelve en una sola respuesta los vehículos con nombres ya resueltos,
    los confirmados sin transporte y la lista de confirmados (para elegir conductor o pasajero).
- **Rationale**: una sola lectura alimenta toda la pantalla, y el frontend no necesita otro
  endpoint para los nombres.

## R5. Frontend

- **Decision**: la página `/transporte`, enlazada desde la principal, tiene:
  - los grupos "Sale de Bogotá" y "Sale de Tocancipá", con tarjetas de vehículo;
  - un resumen "N confirmados sin transporte" con los nombres;
  - en cada tarjeta, "Reservar mi cupo" (o "Cambiarme aquí") y "Reservar para…", con un selector
    de los confirmados sin transporte;
  - en cada pasajero, el botón "Liberar";
  - los botones "Editar" y "Eliminar", este último con un paso de confirmación en la misma
    tarjeta.

  El formulario sirve para crear y editar. La hora usa `input type="time"`.

  Tras cada acción se vuelve a pedir `GET /api/transport`, porque la verdad está en el servidor.
- **Rationale**: es una sola pantalla con todo a la vista, pensada para el celular (SC-003: ver
  quién no tiene transporte en menos de 10 segundos).
