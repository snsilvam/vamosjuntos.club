# Feature Specification: Banner de Cuenta Regresiva

**Feature Branch**: `004-banner-cuenta-regresiva`

**Created**: 2026-10-30

**Status**: Implemented

**Input**: Derivado de la constitución v1.1.0 ("Dominio de la V1"): "Un banner MUY EXPRESIVO que
le diga al usuario cuántos días faltan para el próximo evento al que está invitado."

## Clarifications

### Session 2026-10-30

- Q: El responsable pidió una cuenta regresiva en tiempo real (días, horas, minutos y segundos).
  ¿Cómo convive con el número grande de días? → A: los dos salen del mismo conteo, el tiempo que
  falta para el inicio según el reloj del backend. El número grande son los días completos de ese
  conteo, así que nunca contradice al reloj. Los tramos siguen por días calendario de Colombia.
  En "mañana" y "hoy" no hay número grande: el mensaje del tramo va con el reloj.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ver cuántos días faltan (Priority: P1)

Un invitado con sesión abre la pantalla principal y lo primero que ve es un banner grande,
colorido y animado que lo saluda por su nombre y le dice cuántos días faltan para el paseo, por
ejemplo: "¡Carmencita, faltan 54 días para el paseo!".

**Why this priority**: es la feature emocional de la app; genera expectativa en la familia.

**Independent Test**: con una fecha de referencia conocida, el banner muestra el número correcto
de días y es el elemento más llamativo de la pantalla principal.

**Acceptance Scenarios**:

1. **Given** son las 10:00 a. m. del 2026-10-30 en Colombia, **When** el invitado abre la
   pantalla principal, **Then** el banner muestra "54" como número protagonista junto a su nombre,
   y debajo un reloj que avanza cada segundo desde "54d 04h 00m 00s".
2. **Given** son las 10:00 p. m. del 2026-10-30 en Colombia, **When** se abre el banner, **Then**
   muestra "53" y el reloj "53d 16h 00m 00s": el número y el reloj siempre coinciden.
3. **Given** el banner visible, **When** se compara con el resto de la pantalla, **Then** el
   número de días es el texto más grande y el banner usa colores vivos, emojis o ilustración y una
   animación de entrada.
4. **Given** el dispositivo tiene activada la opción de reducir movimiento, **When** se muestra el
   banner, **Then** aparece sin animaciones, con el mismo contenido.

---

### User Story 2 - El mensaje cambia a medida que se acerca el paseo (Priority: P2)

El tono del banner sube a medida que se acerca la fecha: más intenso en la última semana y
especial el día anterior.

**Why this priority**: mantiene el banner fresco durante los casi dos meses de espera.

**Independent Test**: simulando fechas en cada tramo, el banner muestra el mensaje y la
intensidad de ese tramo.

**Acceptance Scenarios**:

1. **Given** faltan más de 30 días, **When** se abre el banner, **Then** muestra el mensaje de
   tramo "lejano" (por ejemplo, "¡Ya empieza la cuenta regresiva!").
2. **Given** faltan entre 8 y 30 días, **When** se abre, **Then** muestra el mensaje de tramo
   "cerca" (por ejemplo, "¡Ve alistando la maleta!").
3. **Given** faltan entre 2 y 7 días, **When** se abre, **Then** muestra el mensaje de tramo
   "última semana" con la máxima intensidad visual.
4. **Given** falta 1 día, **When** se abre, **Then** muestra "¡Mañana nos vamos!".

---

### User Story 3 - El día del paseo, durante y después (Priority: P3)

El 23 de diciembre el banner celebra que es el día; mientras dura el paseo lo dice; cuando
termina, agradece.

**Why this priority**: evita que el banner muestre números negativos o sin sentido.

**Independent Test**: simulando el 23 de diciembre por la mañana, el 24 de diciembre y el 27 de
diciembre, el banner muestra el mensaje de cada estado.

**Acceptance Scenarios**:

1. **Given** es 2026-12-23 antes de las 2:00 p. m. (Colombia), **When** se abre, **Then** muestra
   "¡Hoy es el día!" como texto grande, sin número de días, y el reloj con las horas que faltan
   para la salida.
2. **Given** es entre el 2026-12-23 2:00 p. m. y el 2026-12-26 11:00 a. m., **When** se abre,
   **Then** muestra "¡Estamos de paseo!".
3. **Given** es después del 2026-12-26 11:00 a. m., **When** se abre, **Then** muestra "¡Qué buen
   paseo! Gracias por venir".

---

### Edge Cases

- La hora del dispositivo está mal configurada: el conteo usa la hora oficial de Colombia y no la
  del dispositivo.
- La app queda abierta al pasar la medianoche de Colombia: el tramo se actualiza sin recargar.
- La app queda abierta a las 2:00 p. m.: el reloj completa otro día y el número grande baja en 1
  sin recargar.
- El invitado está en otro país: los días se cuentan en hora de Colombia.
- El nombre del invitado es largo (por ejemplo, "Nicolás López"): el banner lo muestra completo
  sin romper el diseño en un celular.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El banner MUST mostrarse en la pantalla principal solo a invitados con sesión
  activa (spec 001), como primer elemento visible.
- **FR-002**: El tramo MUST calcularse con la diferencia en días calendario entre la fecha actual
  y la fecha de inicio del evento, ambas en hora de Colombia. El número grande MUST ser los días
  completos (de 24 h) del tiempo que falta para el inicio, el mismo que muestra el reloj
  (FR-011).
- **FR-003**: La fecha actual MUST tomarse de una referencia confiable de hora de Colombia, no del
  reloj del dispositivo.
- **FR-004**: El banner MUST incluir el nombre del invitado con sesión.
- **FR-005**: El número de días MUST ser el elemento tipográfico más grande de la pantalla
  principal.
- **FR-006**: El banner MUST ser visualmente expresivo: colores vivos, emojis o ilustración y una
  animación de entrada; si el dispositivo pide reducir movimiento, MUST mostrarse sin animación.
- **FR-007**: El mensaje y la intensidad visual MUST cambiar por tramos: más de 30 días, 8 a 30,
  2 a 7, 1 día ("¡Mañana nos vamos!"), día del evento antes del inicio ("¡Hoy es el día!"),
  durante el evento ("¡Estamos de paseo!") y después del fin ("¡Qué buen paseo! Gracias por
  venir").
- **FR-008**: El banner MUST usar singular y plural correctos ("falta 1 día" / "faltan 2 días").
- **FR-009**: Si la app sigue abierta al cambiar el día en Colombia, el banner MUST actualizarse
  sin que el invitado recargue.
- **FR-010**: El texto del banner MUST tener contraste suficiente para leerse a pleno sol en un
  celular.
- **FR-011**: Antes del inicio del evento, el banner MUST mostrar un reloj con los días, horas,
  minutos y segundos que faltan, que avanza cada segundo. Se calcula a partir del tiempo restante
  que entrega el backend y del tiempo transcurrido en el dispositivo, nunca de su fecha (FR-003).
  En los tramos "mañana" y "hoy" el banner no muestra el número grande: muestra el mensaje del
  tramo con el reloj.

### Key Entities

- **Evento**: definido en la spec 001; aporta fecha y hora de inicio y fin.
- **Estado de la cuenta regresiva**: tramo actual (lejano, cerca, última semana, mañana, hoy,
  en curso, finalizado) y días restantes.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un invitado puede decir cuántos días faltan en menos de 3 segundos de abrir la app.
- **SC-002**: El tramo y el número de días son correctos en el 100 % de los momentos probados,
  incluidas las transiciones de medianoche y de las 2:00 p. m. en Colombia y desde dispositivos en
  otras zonas horarias. El número grande y el reloj nunca muestran días distintos.
- **SC-003**: Cada uno de los 7 tramos muestra su mensaje correspondiente al simular una fecha
  dentro de él.
- **SC-004**: Al preguntarles, la mayoría de los invitados describe el banner como "llamativo" o
  "emocionante".

## Assumptions

- Hay un solo evento en la V1; el "próximo evento" siempre es el "Paseo Familiar Diciembre 2026".
- Los textos de cada tramo son ejemplos; el texto final puede ajustarse en el diseño mientras se
  respeten los tramos.
- El banner no envía notificaciones ni recordatorios; solo se ve dentro de la app.
- Depende de la spec 001 (sesión y nombre del invitado).
