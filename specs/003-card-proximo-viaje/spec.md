# Feature Specification: Card "Próximo viaje"

**Feature Branch**: `003-card-proximo-viaje`

**Created**: 2026-10-30

**Status**: Draft

**Input**: Derivado de la constitución v1.1.0 ("Dominio de la V1"): "Card que muestra información
del próximo evento: nombre, lugar de salida, lugar de llegada, fecha de inicio y fecha de fin.
Cantidad de invitados del evento, confirmados y pendientes de confirmar."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ver los datos del viaje (Priority: P1)

Un invitado con sesión abre la pantalla principal y ve una card con el nombre del paseo, de dónde
sale (Bogotá y Tocancipá), a dónde va (Acacías, Meta) y cuándo empieza y termina, en hora de
Colombia.

**Why this priority**: es la información que más preguntan los invitados.

**Independent Test**: con el evento creado, un invitado con sesión ve todos los datos del viaje
en la pantalla principal y coinciden con el registro del evento.

**Acceptance Scenarios**:

1. **Given** un invitado con sesión, **When** abre la pantalla principal, **Then** ve el nombre
   "Paseo Familiar Diciembre 2026", los lugares de salida "Bogotá" y "Tocancipá" y el destino
   "Acacías, Meta".
2. **Given** un invitado con sesión, **When** ve la card, **Then** el inicio se muestra como
   "miércoles 23 de diciembre de 2026, 2:00 p. m." y el fin como "sábado 26 de diciembre de 2026,
   11:00 a. m.", ambos en hora de Colombia, sin importar la zona horaria del dispositivo.

---

### User Story 2 - Ver cómo va la asistencia (Priority: P2)

En la misma card, el invitado ve cuántas personas están invitadas, cuántas confirmaron, cuántas
no asistirán y cuántas faltan por responder.

**Why this priority**: ayuda a la familia a saber cuántos van y a presionar a los que faltan.

**Independent Test**: después de que un invitado confirma, al recargar, los conteos cambian en
uno (confirmados +1, pendientes −1).

**Acceptance Scenarios**:

1. **Given** 22 invitados en el catálogo, 5 confirmados y 2 "No asistirá", **When** se abre la
   card, **Then** se ve "22 invitados · 5 confirmados · 2 no asistirán · 15 pendientes".
2. **Given** un invitado acaba de confirmar en otro dispositivo, **When** se recarga la pantalla
   principal, **Then** los conteos reflejan esa confirmación.

---

### Edge Cases

- El registro del evento no existe o le falta un dato obligatorio: la card muestra "La
  información del viaje no está disponible por ahora." en lugar de datos parciales.
- El evento ya terminó: la card sigue mostrando los mismos datos (en la V1 no hay otro evento).
- El organizador agrega invitados al catálogo: el total de invitados cambia en la siguiente carga.
- Nadie ha confirmado: la card muestra "0 confirmados" sin errores.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: La card MUST mostrarse solo a invitados con sesión activa (spec 001).
- **FR-002**: La card MUST mostrar nombre del evento, lugares de salida, destino, fecha y hora de
  inicio y fecha y hora de fin, tomados del registro del evento.
- **FR-003**: Fechas y horas MUST mostrarse en hora de Colombia, en español, con día de la semana,
  día, mes, año y hora en formato de 12 horas.
- **FR-004**: La card MUST mostrar el total de invitados del evento (tamaño del catálogo), el
  número de confirmados, el número de "No asistirá" y el número de pendientes, donde
  pendientes = invitados − confirmados − "No asistirá" (spec 001, FR-007).
- **FR-005**: Los conteos MUST coincidir con los estados de asistencia registrados al momento de
  cargar la pantalla; se actualizan al recargar.
- **FR-006**: Si los datos del evento no están disponibles, la card MUST mostrar un mensaje
  amigable en vez de datos incompletos o errores técnicos.
- **FR-007**: La card MUST NOT mostrar los nombres de quiénes confirmaron ni datos de otros
  invitados; solo conteos.

### Key Entities

- **Evento**: definido en la spec 001; aporta nombre, lugares de salida, destino, inicio y fin.
- **Resumen de asistencia**: total de invitados, confirmados, "No asistirá" y pendientes,
  calculado a partir de los estados de asistencia de los invitados.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un invitado encuentra la fecha de salida y el destino en menos de 10 segundos de
  abrir la pantalla principal.
- **SC-002**: Los conteos de asistencia coinciden con los registros en el 100 % de las
  verificaciones tras recargar.
- **SC-003**: Las fechas se muestran idénticas en dispositivos configurados con zonas horarias
  distintas.

## Assumptions

- Hay un solo evento en la V1, así que "próximo evento" siempre es el "Paseo Familiar Diciembre
  2026".
- Los conteos no se actualizan en tiempo real; basta con que estén correctos al cargar.
- La card no lista nombres de asistentes; si se quisiera, sería una feature nueva.
- Depende de la spec 001 (sesión y estados de asistencia).
