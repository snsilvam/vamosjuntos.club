---

description: "Lista de tareas de la feature 003-card-proximo-viaje"
---

# Tasks: Card "Próximo viaje"

**Input**: Design documents from `/specs/003-card-proximo-viaje/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/api.md](contracts/api.md), [quickstart.md](quickstart.md)

**Tests**: la spec no pide TDD. Igual que en la 008, se escriben pruebas de Go junto al código para
la función pura y el handler, con los fakes de `handlers_test.go`. En el frontend, el build con
chequeo de tipos. De punta a punta, el quickstart.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2)
- Include exact file paths in descriptions

## Path Conventions

Las rutas son relativas a `C:\golang\vamosjuntos-app\`.

Reglas que aplican a todas las tareas:

- **Go**: paquete `main` plano, interfaces pequeñas en `handlers.go`, `context.Context` en el I/O y
  errores envueltos.
- **Seguridad**: la respuesta de `/api/event` nunca lleva nombres, ids ni códigos de invitados
  (FR-007, Principio II).
- **Etiquetas**: **(visto bueno)** publica en producción o crea datos de prueba, así que se hace solo
  con aprobación explícita.

---

## Phase 1: Setup (Shared Infrastructure)

- [X] T001 Crear la rama `003-card-proximo-viaje` desde `main` en `vamosjuntos-backend` y en `vamosjuntos-frontend`

---

## Phase 2: Foundational (Blocking Prerequisites)

No hay prerrequisitos compartidos: el evento, el catálogo, la asistencia y `requireGuest` ya
existen (001/008).

---

## Phase 3: User Story 1 - Ver los datos del viaje (Priority: P1) 🎯 MVP

**Goal**: un invitado confirmado ve en la pantalla principal el nombre del paseo, de dónde sale, a
dónde va y cuándo empieza y termina, en hora de Colombia.

**Independent Test**: con el evento creado, la card muestra los datos del evento y las fechas
"miércoles 23 de diciembre de 2026, 2:00 p. m." y "sábado 26 de diciembre de 2026, 11:00 a. m."
también con el dispositivo en otra zona horaria (quickstart, escenarios 1–3 y 6).

- [X] T002 [US1] En `vamosjuntos-backend/handlers.go`:
  - registrar `mux.HandleFunc("GET /api/event", s.requireGuest(s.handleEvent))` en `routes()`;
  - agregar `eventResponse` con `Event EventSummary \`json:"event"\``;
  - `handleEvent(w, r, account Account, me Guest)` usa `s.loadEvent` (que ya responde
    `404 event_not_found` y `500 internal`) y responde `200` con `event.Summary()`.
- [X] T003 [US1] En `vamosjuntos-backend/handlers_test.go`, casos de `GET /api/event`:
  - invitado confirmado → `200` con `name`, `destination`, `departurePlaces`, y `startsAt`/`endsAt`
    en RFC 3339 con `-05:00`;
  - evento ausente (`fakeEvents.err = errEventNotFound`) → `404 event_not_found`;
  - cuenta vinculada pero pendiente → `403 not_invited`;
  - cuenta sin vincular → `403 not_linked`;
  - sin token → `401 unauthenticated`.
- [X] T004 [P] [US1] En `vamosjuntos-frontend/src/format.ts`, que `formatEventDate` use
  `formatToParts` y cambie a un espacio solo el separador literal que sigue al `weekday`. Así
  devuelve "miércoles 23 de diciembre de 2026, 2:00 p. m." en lugar de "miércoles, 23 de …"
  (research R3). Se mantienen `es-CO`, `timeZone: 'America/Bogota'` y `hour12: true`.
- [X] T005 [P] [US1] En `vamosjuntos-frontend/src/api/types.ts`, agregar
  `export interface EventResponse { event: EventSummary }`, con un comentario que apunte a
  `specs/003-card-proximo-viaje/contracts/api.md`.
- [X] T006 [US1] Crear `vamosjuntos-frontend/src/components/TripCard.tsx`:
  - componente funcional que pide `apiFetch<EventResponse>('/api/event')` en un `useEffect` al
    montarse (ignorar la respuesta si el componente ya se desmontó);
  - estados `loading` | `ready` | `unavailable` (data-model.md, "Estados de la card");
  - en `ready` muestra el nombre del evento como título, "Salida desde" con
    `departurePlaces.join(' · ')`, "Destino", "Inicio" y "Fin" con `formatEventDate` y
    `first-letter:uppercase`, con el mismo estilo de tarjeta de `ConfirmPage`
    (`rounded-3xl bg-white p-5 shadow-md ring-2 ring-orange-100`);
  - en `unavailable` (cualquier error) muestra "La información del viaje no está disponible por
    ahora." dentro de la card (FR-006, research R4); un `403` lo sigue manejando el cliente global;
  - sin botón de reintentar.
- [X] T007 [US1] En `vamosjuntos-frontend/src/pages/HomePage.tsx`:
  - mostrar `<TripCard />` debajo del recuadro punteado, que queda como lugar de la cuenta
    regresiva (spec 004, FR-001: el banner es el primer elemento);
  - el texto del recuadro pasa a "Muy pronto verás aquí la cuenta regresiva.";
  - actualizar el comentario de la línea 24 para que mencione solo la 004.

**Checkpoint**: la card muestra los datos del viaje. La US1 se puede publicar sola.

---

## Phase 4: User Story 2 - Ver cómo va la asistencia (Priority: P2)

**Goal**: en la misma card, el invitado ve cuántos están invitados, cuántos confirmaron, cuántos no
asistirán y cuántos faltan.

**Independent Test**: con 22 invitados, 5 confirmados y 2 "No asistirá", la card dice "22 invitados ·
5 confirmados · 2 no asistirán · 15 pendientes"; al confirmar a otro y recargar, confirmados +1 y
pendientes −1 (quickstart, escenarios 4, 5 y 7).

- [X] T008 [P] [US2] En `vamosjuntos-backend/attendance.go`, agregar:
  - `AttendanceSummary` con `Invited`, `Confirmed`, `Declined` y `Pending` (`int`), con JSON
    `invited`, `confirmed`, `declined` y `pending`;
  - la función pura `summarizeAttendance(catalog Catalog, statuses map[string]Status) AttendanceSummary`,
    con las reglas de data-model.md:
    - "`invited` = `catalog.Len()`";
    - "invitados del catálogo con estado `confirmed`" y "con estado `declined`";
    - "`pending` = `invited − confirmed − declined`";
    - "Un documento de `attendance` cuyo `guestId` no está en el catálogo no cuenta". Recorrer
      `catalog.Sorted()` y consultar `statuses[g.ID]`.
- [X] T009 [P] [US2] En `vamosjuntos-backend/attendance_test.go`, prueba de tabla de
  `summarizeAttendance`:
  - 22 invitados, 5 `confirmed` y 2 `declined` → `{22, 5, 2, 15}`;
  - sin respuestas → `{n, 0, 0, n}`;
  - un estado `confirmed` de un id que no está en el catálogo no cuenta;
  - en todos los casos, `confirmed + declined + pending == invited`.
- [X] T010 [US2] En `vamosjuntos-backend/handlers.go`:
  - agregar `Attendance AttendanceSummary \`json:"attendance"\`` a `eventResponse`;
  - en `handleEvent`, después de `loadEvent`, usar `s.loadCatalog` y `s.attendance.Statuses`
    (un error va a `writeInternalError`) y llenar `Attendance` con `summarizeAttendance`.
    Research R6 acepta que la asistencia se lea dos veces.
- [X] T011 [US2] En `vamosjuntos-backend/handlers_test.go`, extender los casos de `GET /api/event`:
  - el `200` trae `attendance` con los conteos esperados para los estados del fake;
  - el cuerpo no contiene el nombre ni el id de ningún invitado del catálogo de prueba (FR-007).
- [X] T012 [P] [US2] En `vamosjuntos-frontend/src/api/types.ts`, agregar
  `export interface AttendanceSummary { invited: number; confirmed: number; declined: number; pending: number }`
  y el campo `attendance: AttendanceSummary` en `EventResponse`.
- [X] T013 [P] [US2] En `vamosjuntos-frontend/src/format.ts`, agregar
  `formatAttendance(a: AttendanceSummary): string`, que devuelve
  "{invited} invitados · {confirmed} confirmados · {declined} no asistirán · {pending} pendientes".
  El número 1 va en singular ("1 invitado", "1 confirmado", "1 no asistirá", "1 pendiente") y el 0
  en plural (research R5).
- [X] T014 [US2] En `vamosjuntos-frontend/src/components/TripCard.tsx`, mostrar en `ready` la línea
  `formatAttendance(data.attendance)` debajo de las fechas. Debe verse completa en 360 px de ancho,
  pudiendo partirse en varias líneas.

**Checkpoint**: la card completa (US1 + US2).

---

## Phase 5: Polish & Cross-Cutting Concerns

- [X] T015 En `vamosjuntos-backend`, correr `gofmt -l .` (sin salida), `go vet ./...`,
  `golangci-lint run` y `go test ./...`; deben pasar sin errores (Principio IV, compuertas del
  pipeline).
- [X] T016 [P] En `vamosjuntos-frontend`, correr `npm run lint` y `npm run build` (con el chequeo de
  tipos); deben pasar sin errores.
- [ ] T017 (visto bueno) Con la 008 ya en producción (T047 de su `tasks.md`):
  - integrar `003-card-proximo-viaje` en `main` en los dos repos;
  - publicar con `git push origin main:production`, primero el backend y luego el frontend;
  - recorrer los escenarios 1–9 de [quickstart.md](quickstart.md) y deshacer el escenario 5.

---

## Dependencies & Execution Order

- **Setup (T001)** antes de todo.
- **US1 (T002–T007)**: T003 depende de T002; T006 depende de T004 y T005; T007 depende de T006.
- **US2 (T008–T014)** depende de US1 (amplía el mismo endpoint y la misma card):
  - T010 depende de T002 y T008;
  - T011 depende de T010;
  - T014 depende de T006, T012 y T013.
- **Polish (T015–T017)** después de las historias que se vayan a publicar. T017 además depende de
  la 008 en producción.

## Parallel Example

```text
# US1: el backend y la base del frontend a la vez
T002 (handlers.go)  |  T004 (format.ts)  |  T005 (types.ts)

# US2: la función pura, su prueba y el frontend a la vez
T008 (attendance.go)  |  T009 (attendance_test.go)  |  T012 (types.ts)  |  T013 (format.ts)
```

## Implementation Strategy

1. **MVP**: T001–T007. La card con los datos del viaje ya responde la pregunta más frecuente (US1).
2. **Incremento**: T008–T014 agregan los conteos.
3. **Cierre**: T015–T016 en local. T017 cuando la 008 esté publicada, porque la card necesita su
   sesión.
