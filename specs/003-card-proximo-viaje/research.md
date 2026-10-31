# Research: Card "Próximo viaje"

**Feature**: `003-card-proximo-viaje` | **Date**: 2026-10-30

No quedan puntos `NEEDS CLARIFICATION`. La spec se escribió antes de la 008: donde dice "sesión
activa (spec 001)" se lee como la sesión de la 008, es decir, una cuenta de Firebase vinculada a un
invitado del catálogo y confirmado. Es la única cuenta que llega a la pantalla principal.

## R1. Un endpoint nuevo, `GET /api/event`

- **Decision**: ruta nueva `GET /api/event` con `requireGuest` (🔒C). Responde el evento y los
  conteos de asistencia en una sola llamada.
- **Rationale**:
  - La pantalla principal solo existe para invitados confirmados, que es justo lo que exige
    `requireGuest`.
  - `/api/me` decide a qué pantalla va la app y se llama en cada arranque. Meterle los conteos lo
    haría leer más de lo que necesita para esa decisión.
- **Alternatives considered**:
  - **Agregar `event` y `attendance` a `/api/me` cuando `status = "confirmed"`**: mezcla "a dónde
    voy" con "qué muestro", y cambia un contrato que la 008 acaba de cerrar.
  - **Leer Firestore desde el cliente**: lo prohíbe el Principio II (reglas cerradas a clientes).

## R2. Conteos con una función pura

- **Decision**: `summarizeAttendance(catalog Catalog, statuses map[string]Status) AttendanceSummary`.
  - `invited` es `catalog.Len()` (FR-004: tamaño del catálogo).
  - `confirmed` y `declined` cuentan solo invitados que están en el catálogo. Si el organizador
    retira a alguien que ya había respondido, su respuesta deja de contar.
  - `pending = invited − confirmed − declined`. Nunca es negativo, porque los tres salen del mismo
    catálogo.
- **Rationale**: Principio IV (lógica sin I/O, testeable sin Firestore). Con los conteos de
  `attendance` sin filtrar, un invitado retirado del catálogo haría que `pending` no cuadre o quede
  negativo.
- **Alternatives considered**: **contar con agregaciones de Firestore (`count()`)**: no puede
  filtrar por el catálogo, y para 25 documentos no ahorra nada.

## R3. Fechas: misma fuente, formato ajustado a la spec

- **Decision**:
  - El backend sigue mandando `startsAt` y `endsAt` en RFC 3339 con `-05:00` (`EventSummary`
    existente).
  - El frontend formatea con `Intl.DateTimeFormat('es-CO', { timeZone: 'America/Bogota' })`, como
    ya hace `formatEventDate`.
  - `formatEventDate` se ajusta para quitar la coma después del día de la semana. `Intl` produce
    "miércoles, 23 de diciembre de 2026, 2:00 p. m." y la spec pide "miércoles 23 de diciembre de
    2026, 2:00 p. m." (US1, escenario 2). Se usa `formatToParts` y se cambia solo el separador que
    sigue al día de la semana.
- **Rationale**: la zona horaria fija en `Intl` cumple FR-003 y SC-003 sin importar la del
  dispositivo. La pantalla de confirmación (001) usa la misma función, así que las dos quedan
  iguales.
- **Alternatives considered**:
  - **Mandar el texto ya formateado desde Go**: Go no trae nombres de días ni meses en español;
    habría que mantener tablas propias.
  - **Una librería de fechas (date-fns, dayjs)**: es una dependencia para un solo formato que
    `Intl` ya resuelve.

## R4. Errores dentro de la card

- **Decision**: cualquier fallo al cargar (`404 event_not_found`, `500`, sin red) se muestra
  dentro de la card como "La información del viaje no está disponible por ahora." (FR-006). Un
  `403` sigue yendo al manejador global, que vuelve a pedir `/api/me` (comportamiento de la 008).
- **Rationale**: la spec da un solo texto para este caso, y el resto de la pantalla (cerrar sesión,
  confirmar a otro) debe seguir funcionando aunque la card falle.
- **Alternatives considered**: **un botón "Reintentar"**: no está en la spec (Principio VI). Basta
  con recargar (FR-005).

## R5. Singular y plural en los conteos

- **Decision**: "1 invitado / 2 invitados", "1 confirmado / 2 confirmados", "1 no asistirá / 2 no
  asistirán", "1 pendiente / 2 pendientes". El cero va en plural ("0 confirmados").
- **Rationale**: la spec da el ejemplo en plural y el caso "0 confirmados". El singular es solo
  gramática correcta, no una función nueva.

## R6. La asistencia se lee dos veces por request

- **Decision**: se acepta. `requireGuest` lee `attendance` para saber si el invitado está
  confirmado, y el handler vuelve a leerla para los conteos.
- **Rationale**: son 25 documentos como máximo, en una pantalla que se abre pocas veces al día.
- **Alternatives considered**: **pasar los estados desde el middleware**: cambia la firma de
  `requireGuest` para todos los handlers y les entrega datos que no usan.
