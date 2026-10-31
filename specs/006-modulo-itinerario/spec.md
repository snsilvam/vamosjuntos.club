# Feature Specification: Módulo Itinerario

**Feature Branch**: `006-modulo-itinerario`

**Created**: 2026-10-30

**Status**: Implemented

**Input**: Derivado de la constitución v1.1.0 ("Dominio de la V1"): "Módulo Itinerario:
actividades y horarios en el transcurso del evento." Evento del 23 de diciembre de 2026, 2:00 p. m.,
al 26 de diciembre de 2026, 11:00 a. m., hora Colombia.

## Clarifications

### Session 2026-10-30

- Q: FR-005 permite que la hora de fin caiga en el día siguiente, pero US2-2 rechaza un fin anterior
  o igual al inicio. ¿Cómo se distingue? → A: con una casilla explícita "Termina al día siguiente".
  Sin marcarla, el fin MUST ser posterior al inicio (US2-2). Marcada, el fin cae el día siguiente y
  la actividad dura como máximo 24 horas.
- Q: La spec cita la sesión de la spec 001. ¿Sigue vigente? → A: no; desde la constitución v3.0.0
  la sesión es la de Firebase Authentication (spec 008). El módulo es para invitados con cuenta
  vinculada y asistencia confirmada, como Transporte (spec 005).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consultar el itinerario (Priority: P1)

Un invitado entra a Itinerario y ve las actividades del paseo agrupadas por día (23 al 26 de
diciembre) y ordenadas por hora: qué se hace, a qué hora, dónde y cualquier detalle útil.

**Why this priority**: es lo que la familia consulta durante el paseo para saber qué sigue.

**Independent Test**: con actividades registradas, la pantalla las muestra por día y en orden
cronológico.

**Acceptance Scenarios**:

1. **Given** hay actividades en varios días, **When** un invitado abre Itinerario, **Then** las ve
   agrupadas por día ("Miércoles 23 de diciembre", etc.) y ordenadas por hora de inicio.
2. **Given** dos actividades a la misma hora, **When** se muestran, **Then** ambas aparecen (se
   permiten actividades en paralelo).
3. **Given** no hay actividades, **When** se abre Itinerario, **Then** se ve un mensaje que indica
   que el itinerario aún no tiene actividades.

---

### User Story 2 - Agregar una actividad (Priority: P2)

Un invitado confirmado agrega una actividad: título, día, hora de inicio y, de forma opcional,
hora de fin, lugar y descripción.

**Why this priority**: sin actividades no hay itinerario, pero consultarlo es más frecuente que
crearlo.

**Independent Test**: una actividad nueva aparece en el día y la posición correcta.

**Acceptance Scenarios**:

1. **Given** un invitado confirmado, **When** agrega "Piscina" el 24 de diciembre de
   10:00 a. m. a 12:00 p. m., **Then** aparece en el 24 de diciembre en su posición por hora.
2. **Given** el formulario, **When** la hora de fin es anterior o igual a la de inicio, **Then**
   se muestra "La hora de fin debe ser después de la hora de inicio." y no se guarda.
3. **Given** el formulario, **When** se elige un día fuera del 23 al 26 de diciembre de 2026,
   **Then** se muestra "La actividad debe estar dentro de los días del paseo." y no se guarda.

---

### User Story 3 - Editar o eliminar una actividad (Priority: P3)

Cualquier invitado confirmado corrige o elimina una actividad cuando cambian los planes.

**Why this priority**: los planes cambian, pero con menos frecuencia.

**Independent Test**: al editar la hora de una actividad, cambia de posición; al eliminarla, ya no
aparece.

**Acceptance Scenarios**:

1. **Given** una actividad existente, **When** se cambia su hora, **Then** se reordena en su día.
2. **Given** una actividad existente, **When** se elimina tras confirmar, **Then** desaparece del
   itinerario.

---

### Edge Cases

- Un invitado borra por error una actividad que creó otro: se permite (regla de permisos
  abierta); la confirmación previa de FR-008 es la única protección.
- Una actividad cruza la medianoche (por ejemplo, la cena de Nochebuena de 8:00 p. m. a 1:00 a. m.):
  se muestra en el día de inicio con su hora de fin del día siguiente.
- Un invitado consulta desde otra zona horaria: todas las horas se muestran en hora de Colombia.
- Títulos o descripciones largas: se muestran completos sin romper el diseño en un celular.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El módulo MUST estar disponible solo para invitados con sesión activa, cuenta
  vinculada y asistencia confirmada (spec 008).
- **FR-002**: El sistema MUST mostrar las actividades agrupadas por día y ordenadas por hora de
  inicio; en empate, por título.
- **FR-003**: Una actividad MUST tener título, día y hora de inicio; hora de fin, lugar y
  descripción son opcionales.
- **FR-004**: El día de la actividad MUST estar entre el 23 y el 26 de diciembre de 2026 (hora
  Colombia).
- **FR-005**: Si hay hora de fin, MUST ser posterior a la de inicio. Puede caer en el día
  siguiente solo si se marca "Termina al día siguiente", y en ese caso la actividad dura como
  máximo 24 horas.
- **FR-006**: Todas las horas MUST mostrarse y capturarse en hora de Colombia.
- **FR-007**: Cualquier invitado confirmado MUST poder crear, editar y eliminar cualquier
  actividad (misma regla de la spec 005, FR-009).
- **FR-008**: Eliminar una actividad MUST pedir confirmación.
- **FR-009**: Todo error MUST presentarse como un mensaje amigable en español.

### Key Entities

- **Actividad**: título, día, hora de inicio, hora de fin (opcional), "termina al día siguiente"
  (solo con hora de fin), lugar (opcional) y descripción (opcional).
- **Evento**: definido en las specs 001 y 008; delimita los días válidos.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Durante el paseo, un invitado encuentra la siguiente actividad en menos de 15
  segundos.
- **SC-002**: El 100 % de las actividades aparecen en el día y el orden correctos.
- **SC-003**: Un invitado con permiso agrega una actividad en menos de 1 minuto.

## Assumptions

- Se permiten actividades en todo el día calendario del 23 al 26 de diciembre (no solo entre
  2:00 p. m. del 23 y 11:00 a. m. del 26), para incluir la salida desde Bogotá/Tocancipá y el
  regreso.
- No hay recordatorios ni notificaciones de actividades.
- No se marca la actividad "en curso" ni se resalta la siguiente; solo se lista en orden.
- No hay inscripción de asistentes por actividad.
- Depende de la spec 008 (sesión) y de la regla de permisos de la spec 005.
