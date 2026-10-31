# Research: Módulo Itinerario

**Feature**: `006-modulo-itinerario` | **Date**: 2026-10-30

No hay puntos `NEEDS CLARIFICATION`. La spec quedó aclarada el 2026-10-30 ("Termina al día
siguiente" y sesión de la 008).

## R1. Día y horas como texto en hora de Colombia

- **Decision**: la actividad guarda `date` (`AAAA-MM-DD`), `startTime` y `endTime` (`HH:MM` de 24
  horas) como texto, en hora de Colombia, y `endsNextDay` como booleano.
- **Rationale**:
  - Es lo que el invitado escribe y lee: no hay conversión que pueda correr la hora en un celular
    configurado en otra zona horaria (FR-006).
  - El orden cronológico es el orden del texto (`date`, luego `startTime`).
  - Colombia no tiene horario de verano, así que no hay horas ambiguas.
- **Alternatives considered**: `Timestamp` de Firestore para inicio y fin. Obliga a convertir en
  cada lectura y escritura, y un error de zona horaria en el frontend movería actividades de día.

## R2. Días válidos tomados del evento

- **Decision**: los días válidos son las fechas de calendario de Colombia entre `startsAt` y
  `endsAt` del evento, ambas incluidas (hoy, del 23 al 26 de diciembre). La función pura
  `eventDays(e Event)` las calcula y la API las devuelve en `days`.
- **Rationale**: FR-004 y la spec ("el evento delimita los días válidos"). El frontend usa `days`
  para el selector de día, así que no repite fechas fijas.
- **Alternatives considered**: fijar las cuatro fechas en el código; si el responsable corrige el
  evento, habría que publicar de nuevo.

## R3. Sin transacción sobre la colección

- **Decision**:
  - crear es `Doc(nuevoId).Create`;
  - editar es una transacción corta sobre ese documento: lee, devuelve `404` si no existe y
    escribe;
  - eliminar es `Delete` con la precondición `Exists`.
- **Rationale**: a diferencia de la 005, no hay reglas entre actividades. Se permiten en paralelo
  (US1-2) y no hay cupos. Si dos personas editan la misma actividad a la vez, gana la última, como
  en cualquier lista compartida; la spec acepta la regla de permisos abierta.
- **Alternatives considered**: la transacción sobre toda la colección de la 005. Aquí no protege
  ninguna regla y solo agrega reintentos.

## R4. Validación pura con códigos por regla

- **Decision**: `validateActivity(in ActivityInput, days []string) (ActivityInput, error)` recorta
  espacios y devuelve:
  - `errInvalidActivity`: título vacío o de más de 80 caracteres, lugar de más de 80, descripción
    de más de 500, horas mal formadas, o `endsNextDay` sin hora de fin;
  - `errActivityOutOfRange`: `date` no está en `days` (FR-004);
  - `errActivityTimeOrder`: sin `endsNextDay`, el fin no es posterior al inicio; con
    `endsNextDay`, la duración pasa de 24 horas (FR-005).
- **Rationale**: los dos mensajes de la spec ("La hora de fin debe ser después de la hora de
  inicio." y "La actividad debe estar dentro de los días del paseo.") necesitan códigos propios.
  El frontend valida lo mismo antes de enviar, pero el backend es quien decide (Principio II).

## R5. Escrituras responden con el itinerario completo

- **Decision**: `POST` responde `201` y `PUT` responde `200`, los dos con el mismo cuerpo de
  `GET /api/itinerary`. `DELETE` responde `204`.
- **Rationale**: es la convención que quedó en la 005 (desviación registrada allí): la pantalla
  se actualiza sin otra llamada.

## R6. Orden y agrupación

- **Decision**: el backend ordena por `date`, `startTime`, título sin acentos (`foldAccents`) e
  `id` (FR-002). El frontend agrupa por `date` y muestra solo los días con actividades, con el
  encabezado "Miércoles 23 de diciembre". El nombre del día se calcula con
  `Intl.DateTimeFormat('es-CO', { timeZone: 'UTC' })` sobre `AAAA-MM-DDT12:00:00Z`, así que no
  depende de la zona del dispositivo.
- **Rationale**: una actividad que cruza la medianoche se lista en su día de inicio, con "hasta la
  1:00 a. m. del día siguiente" (caso borde de la spec).

## R7. Pantalla

- **Decision**: igual que Transporte:
  - página `/itinerario` dentro del bloque `confirmed` de `App.tsx`;
  - `ActivityForm` para crear y editar;
  - eliminar con un paso de confirmación en la misma tarjeta ("¿Eliminar esta actividad?") (FR-008);
  - horas con `input type="time"` y el día con un selector de `days`.
- **Rationale**: reutiliza `apiFetch`, los estilos y los mensajes de la 005, y deja el mismo patrón
  para la 007.
