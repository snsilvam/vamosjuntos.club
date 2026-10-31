# Tasks: Módulo Itinerario

**Input**: Design documents from `/specs/006-modulo-itinerario/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/api.md, quickstart.md

**Tests**: se incluyen pruebas de Go para la regla pura y los handlers, como en la 005. El
Principio IV exige que la lógica de negocio sea testeable sin Firestore.

**Organization**: por historia de usuario. Código en ramas `006-modulo-itinerario` de
`vamosjuntos-backend` y `vamosjuntos-frontend`.

## Format: `[ID] [P?] [Story] Description`

---

## Phase 1: Setup

- [X] T001 Crear la rama `006-modulo-itinerario` desde `main` en `vamosjuntos-backend` y en `vamosjuntos-frontend`

---

## Phase 2: Foundational

**Purpose**: modelo, regla pura, almacén y tipos que usan todas las historias.

- [X] T002 Crear vamosjuntos-backend/itinerary.go con:
  - `Activity` (campos de Firestore `title`, `date`, `startTime`, `endTime`, `endsNextDay`, `place` y `description`, e `ID` con `firestore:"-"`) y `ActivityInput` con las mismas etiquetas JSON del contrato;
  - errores `errInvalidActivity`, `errActivityOutOfRange`, `errActivityTimeOrder` y `errActivityNotFound`;
  - `eventDays(e Event) []string`: fechas de Colombia `AAAA-MM-DD` desde `StartsAt` hasta `EndsAt`, ambas incluidas;
  - `validateActivity(in ActivityInput, days []string) (ActivityInput, error)` con las reglas de data-model.md: "De 1 a 80 caracteres tras recortar espacios" (título), lugar "hasta 80 caracteres", descripción "hasta 500 caracteres", `HH:MM` de 24 horas, `endsNextDay` "Solo con `endTime`", fin posterior al inicio sin `endsNextDay` y "la duración es como máximo 24 h" con él;
  - `sortActivities`: `date` → `startTime` → `foldAccents(title)` → `ID`.
- [X] T003 [P] Crear vamosjuntos-backend/itinerary_test.go con tablas para `eventDays` (evento de la V1 → 23 a 26 de diciembre), `validateActivity` (cada error de data-model.md, 20:00–01:00 con `endsNextDay` válido, 22:00–23:00 con `endsNextDay` rechazado, recorte de espacios) y `sortActivities` (empate de hora resuelto por título sin acentos)
- [X] T004 Agregar a vamosjuntos-backend/itinerary.go `ItineraryStore` con `NewItineraryStore(client, eventID)` sobre `events/{EVENT_ID}/activities`:
  - `List`;
  - `Create(ctx, in, by)` con id nuevo;
  - `Update(ctx, id, in, by)` en una transacción que devuelve `errActivityNotFound` si no existe;
  - `Delete(ctx, id)` con la precondición `Exists`, que traduce `NotFound` a `errActivityNotFound`.
  Las escrituras guardan `updatedAt` (`ServerTimestamp`) y `updatedBy` (research R3).
- [X] T005 Agregar a vamosjuntos-backend/errors.go los códigos `invalid_activity`, `activity_out_of_range`, `activity_time_order` y `activity_not_found`
- [X] T006 En vamosjuntos-backend/handlers.go:
  - interfaz `itineraryStore` (`List`, `Create`, `Update`, `Delete`) y campo `itinerary` en `server`;
  - parámetro en `NewServer`;
  - y en vamosjuntos-backend/main.go, `NewItineraryStore`;
  - ajustar las llamadas a `NewServer` en las pruebas existentes.
- [X] T007 [P] Agregar a vamosjuntos-frontend/src/api/types.ts `Activity`, `ActivityInput` e `Itinerary`, y a vamosjuntos-frontend/src/messages.ts los mensajes de contracts/api.md ("Mensajes en la UI")

**Checkpoint**: la regla pura pasa sus pruebas y el servidor compila con el almacén nuevo.

---

## Phase 3: User Story 1 - Consultar el itinerario (Priority: P1) 🎯 MVP

**Goal**: ver las actividades agrupadas por día y en orden de hora.

**Independent Test**: escenarios 1, 3, 4, 13 y 14 del quickstart.

- [X] T008 [US1] Crear vamosjuntos-backend/itinerary_handlers.go con `GET /api/itinerary` (`{"days", "activities"}` ordenadas, `404 event_not_found` si no hay evento) y registrar la ruta al final de `routes()` en vamosjuntos-backend/handlers.go con `requireGuest`
- [X] T009 [P] [US1] Crear vamosjuntos-backend/itinerary_handlers_test.go con `fakeItinerary`: lista ordenada, `days` del evento, `401` sin token y `403` sin vincular
- [X] T010 [US1] Crear vamosjuntos-frontend/src/pages/ItineraryPage.tsx:
  - agrupa por `date` y muestra solo los días con actividades, con encabezados como "Miércoles 23 de diciembre" (`Intl.DateTimeFormat('es-CO', { timeZone: 'UTC' })` sobre `AAAA-MM-DDT12:00:00Z`);
  - cada actividad con hora de inicio y fin en formato "10:00 a. m.", lugar y descripción;
  - "hasta la 1:00 a. m. del día siguiente" cuando `endsNextDay`;
  - estado vacío "El itinerario aún no tiene actividades." y error amigable al cargar;
  - botón "Volver".
- [X] T011 [US1] Agregar la ruta `/itinerario` en el bloque `confirmed` de vamosjuntos-frontend/src/App.tsx y el botón "Itinerario: actividades y horarios" en vamosjuntos-frontend/src/pages/HomePage.tsx, debajo del de Transporte

**Checkpoint**: Itinerario se ve, vacío o con actividades (MVP).

---

## Phase 4: User Story 2 - Agregar una actividad (Priority: P2)

**Goal**: un confirmado agrega una actividad con validación de día y horas.

**Independent Test**: escenarios 2, 5, 6, 7, 8 y 12 del quickstart.

- [X] T012 [US2] Agregar `POST /api/itinerary/activities` en vamosjuntos-backend/itinerary_handlers.go (valida con `validateActivity` y los días del evento, responde `201` con el itinerario) y `writeItineraryError` (`400` y `404` según data-model.md), con pruebas en vamosjuntos-backend/itinerary_handlers_test.go para `201`, `invalid_activity`, `activity_out_of_range` y `activity_time_order`
- [X] T013 [US2] Crear vamosjuntos-frontend/src/pages/ActivityForm.tsx:
  - título, día (selector con `days`), hora de inicio, hora de fin opcional, casilla "Termina al día siguiente" (solo con hora de fin), lugar y descripción;
  - valida antes de enviar las mismas reglas y muestra los mensajes de la spec;
  - en ItineraryPage.tsx, botón "Agregar actividad" que lo abre y, al guardar, muestra el itinerario que devuelve la API.

---

## Phase 5: User Story 3 - Editar o eliminar una actividad (Priority: P3)

**Goal**: corregir o eliminar cualquier actividad.

**Independent Test**: escenarios 9, 10 y 11 del quickstart.

- [X] T014 [US3] Agregar `PUT` y `DELETE /api/itinerary/activities/{activityID}` en vamosjuntos-backend/itinerary_handlers.go y sus rutas en handlers.go, con pruebas en vamosjuntos-backend/itinerary_handlers_test.go para editar (reordena), eliminar (`204`) y `404 activity_not_found`
- [X] T015 [US3] En vamosjuntos-frontend/src/pages/ItineraryPage.tsx:
  - "Editar" abre ActivityForm con los datos;
  - "Eliminar" pide confirmación en la misma tarjeta ("¿Eliminar esta actividad?");
  - ante `activity_not_found` recarga la lista.

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T016 Pasar las compuertas: backend (`gofmt -l .` sin salida, `go vet ./...`, `golangci-lint run`, `go test ./...`) y frontend (`npm run lint`, `npm run build`)
- [X] T017 Validar en local los escenarios del quickstart. La API con pruebas de Go y la UI a 360 px con `/api/*` simulado, para no tocar datos reales
- [X] T018 (visto bueno) Integrar en `main`, publicar backend y luego frontend (`git push origin main:production`) y recorrer el quickstart en `vamosjuntos.club` con cuentas de prueba sobre invitados pendientes, borrando solo lo creado
- [X] T019 Registrar aquí las notas de implementación y hacer commit de specs/006-modulo-itinerario

---

## Dependencies & Execution Order

- **Setup → Foundational**: T001 → T002 a T007. T003 y T007 son paralelas al resto.
- **Historias**: US1 → US2 → US3. US1 da la pantalla que usan las demás; en el backend, US2 y US3
  agregan rutas al mismo archivo.
- **Polish**: al final. T018 publica el backend antes que el frontend, porque el frontend nuevo
  llama rutas nuevas.

## Parallel Example

```bash
Task: "itinerary_test.go (T003)"
Task: "tipos y mensajes del frontend (T007)"
Task: "itinerary_handlers_test.go (T009) mientras se hace ItineraryPage.tsx (T010)"
```

## Implementation Strategy

- **MVP**: Setup, Foundational y US1. La familia ya puede consultar un itinerario que alguien
  cargue.
- **Luego**: US2 (agregar), que es como se llena el itinerario, y US3 (editar o eliminar).
- La 007 sigue el mismo patrón; conviene integrar la 006 en `main` antes de empezar la 007 para no
  cruzar cambios en `handlers.go`, `App.tsx` y `HomePage.tsx`.

## Implementation Notes (2026-10-30)

- **Backend** `c7f4ac1` (revisión publicada por pipeline):
  - `itinerary.go`: `Activity`, `eventDays`, `validateActivity` (pura), `sortActivities` con
    `cmp.Or` e `ItineraryStore`;
  - `itinerary_handlers.go`: la interfaz `itineraryStore` y los 4 handlers;
  - pruebas en `itinerary_test.go` e `itinerary_handlers_test.go`;
  - en `handlers.go` solo cambian el struct, `NewServer` y 4 rutas.
- **Frontend** `add683c`:
  - `ItineraryPage.tsx` y `ActivityForm.tsx`;
  - ruta `/itinerario` y botón en la pantalla principal;
  - `formatClock` se movió de `TransportPage.tsx` a `format.ts` para compartirla, y se agregó
    `formatDayHeading`.
- **Desviaciones**: ninguna respecto al contrato. La aclaración "Termina al día siguiente" se
  registró en spec.md antes del plan.
- **Validación**:
  - compuertas en verde (`gofmt` sin CR, `vet`, `golangci-lint` con 0 issues, `test`, `lint`,
    `build`);
  - UI a 360 px con `/api/*` simulado: 16/16 en local y 16/16 en `vamosjuntos.club`;
  - API en producción con 2 cuentas de prueba sobre invitados pendientes: 15/15 (escenarios 2 a
    12 y 14).
- **Limpieza**: 4 actividades y 3 cuentas de prueba borradas. `activities` quedó en 0 y `links` y
  `attendance` en 4, como antes.
