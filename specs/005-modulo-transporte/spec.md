# Feature Specification: Módulo Transporte

**Feature Branch**: `005-modulo-transporte`

**Created**: 2026-10-30

**Status**: Draft

**Input**: Derivado de la constitución v1.1.0 ("Dominio de la V1"): "Módulo Transporte: vehículos
y cupos." Lugares de salida: Bogotá y Tocancipá. Destino: Acacías, Meta.

## Clarifications

### Session 2026-10-30

- Q: ¿Quién puede crear, editar y eliminar vehículos, actividades e ítems de compra? → A:
  Cualquier invitado confirmado puede crear, editar y eliminar todo.
- Q: ¿Se puede reservar cupo para otro invitado? → A: Sí (spec 001, clarificación sobre actuar
  en nombre de otro).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ver vehículos y cupos disponibles (Priority: P1)

Un invitado entra a Transporte y ve los vehículos del paseo agrupados por lugar de salida (Bogotá
y Tocancipá): quién maneja, cuántos cupos tiene, cuántos quedan libres y quiénes van en cada uno.
También ve cuántos confirmados aún no tienen transporte.

**Why this priority**: resolver cómo llega cada uno a Acacías es el principal problema logístico.

**Independent Test**: con vehículos y pasajeros registrados, la pantalla muestra cupos libres y
pasajeros correctos para cada vehículo.

**Acceptance Scenarios**:

1. **Given** hay vehículos registrados, **When** un invitado abre Transporte, **Then** los ve
   agrupados por lugar de salida, con conductor, cupos totales, cupos libres y pasajeros.
2. **Given** hay confirmados sin vehículo, **When** se abre Transporte, **Then** se ve
   "N confirmados sin transporte" con sus nombres.
3. **Given** no hay vehículos, **When** se abre Transporte, **Then** se ve un mensaje que invita a
   registrar el primero.

---

### User Story 2 - Registrar un vehículo (Priority: P1)

Un invitado que va a manejar registra su vehículo: un nombre para reconocerlo (por ejemplo, "La
camioneta de Orlando"), el conductor, el lugar de salida, los cupos para pasajeros y, si quiere,
la hora estimada de salida y una nota.

**Why this priority**: sin vehículos no hay cupos que repartir.

**Independent Test**: un invitado registra un vehículo y aparece de inmediato en la lista con
todos sus cupos libres.

**Acceptance Scenarios**:

1. **Given** un invitado confirmado, **When** registra un vehículo con datos
   válidos, **Then** aparece en la lista con todos sus cupos libres.
2. **Given** el formulario, **When** se ingresan 0 cupos o más de 24, **Then** se muestra un
   mensaje amigable y no se guarda.

---

### User Story 3 - Reservar y liberar un cupo (Priority: P2)

Un invitado elige un vehículo con cupo libre y reserva un puesto; si cambia de planes, lo libera.

**Why this priority**: es como cada persona asegura su transporte.

**Independent Test**: un invitado reserva un cupo y el contador de libres baja en uno; al liberar,
vuelve a subir.

**Acceptance Scenarios**:

1. **Given** un vehículo con cupos libres, **When** un invitado reserva, **Then** aparece como
   pasajero y los cupos libres bajan en uno.
2. **Given** el invitado ya tiene cupo en un vehículo, **When** reserva en otro, **Then** el
   sistema lo cambia de vehículo (nunca ocupa dos cupos a la vez).
3. **Given** un vehículo lleno, **When** alguien intenta reservar, **Then** ve "Este vehículo ya
   no tiene cupos." y no se reserva.
4. **Given** un invitado con cupo, **When** lo libera, **Then** deja de aparecer como pasajero.
5. **Given** un invitado confirmado sin transporte, **When** otro invitado le reserva un cupo,
   **Then** aparece como pasajero de ese vehículo.
6. **Given** un pasajero, **When** cualquier invitado confirmado libera su cupo, **Then** el
   pasajero queda sin transporte.

---

### User Story 4 - Editar o eliminar un vehículo (Priority: P3)

Cualquier invitado confirmado cambia los datos de un vehículo o lo elimina si ya no va.

**Why this priority**: los planes cambian, pero es menos frecuente que reservar.

**Independent Test**: al eliminar un vehículo, sus pasajeros pasan a "sin transporte".

**Acceptance Scenarios**:

1. **Given** un vehículo con 3 pasajeros, **When** se intenta bajar a 2 cupos, **Then** el sistema
   lo impide con un mensaje amigable.
2. **Given** un vehículo con pasajeros, **When** se elimina tras confirmar, **Then** sus pasajeros
   quedan sin transporte.

---

### Edge Cases

- Dos invitados intentan reservar el último cupo al mismo tiempo: solo uno lo obtiene; el otro ve
  el mensaje de vehículo lleno.
- El conductor intenta reservarse como pasajero de otro vehículo: el sistema lo impide mientras
  sea conductor de un vehículo.
- Un invitado que no ha confirmado asistencia: no aparece como pasajero ni como conductor posible.
- Cupos totales de todos los vehículos superan el número de invitados: se permite (sobran cupos).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El módulo MUST estar disponible solo para invitados con sesión activa (spec 001).
- **FR-002**: El sistema MUST listar los vehículos agrupados por lugar de salida (Bogotá,
  Tocancipá), mostrando nombre, conductor, cupos totales, cupos libres, pasajeros, hora estimada
  de salida y nota.
- **FR-003**: Un vehículo MUST tener nombre, conductor (un invitado confirmado), lugar de salida
  (Bogotá o Tocancipá) y cupos para pasajeros entre 1 y 24; hora estimada y nota son opcionales.
- **FR-004**: El conductor MUST NOT contar como pasajero ni ocupar cupo.
- **FR-005**: Una persona MUST ocupar como máximo un lugar en todo el módulo (un cupo de pasajero o
  ser conductor de un vehículo).
- **FR-006**: El sistema MUST impedir reservar en un vehículo sin cupos libres, incluso con
  reservas simultáneas.
- **FR-007**: El sistema MUST impedir reducir los cupos de un vehículo por debajo de su número de
  pasajeros.
- **FR-008**: Al eliminar un vehículo, sus pasajeros MUST quedar sin transporte.
- **FR-009**: Cualquier invitado confirmado MUST poder crear, editar y eliminar cualquier
  vehículo. Esta misma regla aplica a Itinerario (006) y Compras (007); no hay roles.
- **FR-010**: El sistema MUST mostrar la cantidad y los nombres de los confirmados sin transporte.
- **FR-011**: Solo invitados confirmados MUST poder ser conductores o pasajeros.
- **FR-012**: Todo error MUST presentarse como un mensaje amigable en español.
- **FR-013**: Cualquier invitado confirmado MUST poder reservar un cupo para sí mismo o para otro
  invitado confirmado sin transporte, y liberar el cupo de cualquier pasajero.

### Key Entities

- **Vehículo**: nombre, conductor, lugar de salida, cupos para pasajeros, hora estimada de salida
  (opcional) y nota (opcional).
- **Reserva de cupo**: la relación entre un invitado y el vehículo en el que va como pasajero.
- **Invitado**: definido en la spec 001.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un invitado reserva un cupo en menos de 30 segundos desde que abre Transporte.
- **SC-002**: En ninguna prueba de reservas simultáneas un vehículo termina con más pasajeros que
  cupos.
- **SC-003**: El organizador identifica quién no tiene transporte en menos de 10 segundos.
- **SC-004**: A la fecha de salida, el 100 % de los confirmados tienen vehículo asignado o figuran
  explícitamente como "sin transporte".

## Assumptions

- La misma asignación de vehículo vale para la ida y el regreso; no se gestionan por separado.
- Los lugares de salida son solo Bogotá y Tocancipá, fijados por la constitución.
- No se gestionan costos, gasolina ni peajes (la constitución prohíbe pagos y extras no pedidos).
- No hay mapas, rutas ni seguimiento en tiempo real.
- Depende de la spec 001 (sesión y estado de confirmación).
