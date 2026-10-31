---

description: "Lista de tareas de la feature 005-modulo-transporte"
---

# Tasks: Módulo Transporte

**Input**: Design documents from `/specs/005-modulo-transporte/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/api.md](contracts/api.md), [quickstart.md](quickstart.md)

**Tests**: no hay TDD. Se escriben pruebas de Go para la regla pura (tabla de casos) y para los
handlers con un fake, igual que en la 001 y la 008.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)

## Path Conventions

El código va en **worktrees** aparte, porque otras sesiones usan los working trees principales:

- backend: `C:\golang\vamosjuntos-app\vamosjuntos-backend-005\`
- frontend: `C:\golang\vamosjuntos-app\vamosjuntos-frontend-005\`

En `vamosjuntos.club` solo se hace `git add` de `specs/005-modulo-transporte`.

---

## Phase 1: Setup

- [X] T001 Crear los worktrees de la rama `005-modulo-transporte` desde `main`:
  - `git -C vamosjuntos-backend worktree add ../vamosjuntos-backend-005 -b 005-modulo-transporte main`;
  - lo mismo para el frontend en `../vamosjuntos-frontend-005`;
  - en el del frontend, correr `npm ci` y copiar `.env.local` desde vamosjuntos-frontend/.

---

## Phase 2: Foundational

- [X] T002 Crear vamosjuntos-backend-005/transport.go:
  - **Tipos**: `Vehicle{ID, Name, DriverID, Departure, Seats, DepartureTime, Note, Passengers}` y `VehicleInput`.
  - **Constantes**: `departureBogota = "bogota"` y `departureTocancipa = "tocancipa"`.
  - **Errores de dominio**: `errInvalidVehicle`, `errDriverUnavailable`, `errSeatsBelowPassengers`, `errVehicleFull`, `errGuestIsDriver`, `errGuestNotConfirmed`, `errVehicleNotFound` y `errPassengerNotFound`.
  - **`validateVehicleInput`**:
    - `name` de 1 a 60 caracteres tras recortar espacios;
    - `departure` `bogota` o `tocancipa`;
    - `seats` de 1 a 24;
    - `departureTime` vacío o `HH:MM` de 24 horas;
    - `note` de hasta 200 caracteres.
- [X] T003 Agregar a vamosjuntos-backend-005/transport.go la regla pura `applyTransport(vehicles []Vehicle, cmd transportCmd, confirmed map[string]bool) (transportResult, error)` (research R3), con un comando por acción:
  - **create**: el conductor debe ser confirmado y no conducir ni ir de pasajero → si no, `errDriverUnavailable`.
  - **update**: igual que create, aceptando el mismo conductor del vehículo; además, `seats ≥ len(passengers)` → si no, `errSeatsBelowPassengers`.
  - **delete**: borra el vehículo.
  - **reserve**:
    - el invitado debe ser confirmado → si no, `errGuestNotConfirmed`;
    - si conduce → `errGuestIsDriver`;
    - si ya va en ese vehículo → no hace nada;
    - si el vehículo está lleno → `errVehicleFull`;
    - si va en otro vehículo, se le quita de ese (cambio de vehículo).
  - **release**: si no va en el vehículo → `errPassengerNotFound`.
  - **Resultado**: los vehículos a escribir y el id a borrar.
- [X] T004 [P] Crear vamosjuntos-backend-005/transport_test.go: tabla de casos de `validateVehicleInput` y `applyTransport` con todos los errores de T003, el cambio de vehículo, la no-op y el conductor que sigue siendo el mismo al editar
- [X] T005 Agregar a vamosjuntos-backend-005/transport.go el `TransportStore` sobre `events/{EVENT_ID}/vehicles`:
  - `List(ctx)`;
  - `Apply(ctx, cmd, confirmed, by)`: transacción que lee todos los vehículos con `tx.Documents`, llama a `applyTransport` y escribe con `updatedAt = ServerTimestamp` y `updatedBy`, o borra;
  - IDs nuevos con `NewDoc`.
- [X] T006 Agregar a vamosjuntos-backend-005/errors.go los códigos `invalid_vehicle`, `driver_unavailable`, `seats_below_passengers`, `vehicle_full`, `guest_is_driver`, `guest_not_confirmed`, `vehicle_not_found` y `passenger_not_found`
- [X] T007 Conectar el almacén en el backend:
  - en vamosjuntos-backend-005/handlers.go, la interfaz `transportStore` y el campo en `server`;
  - un helper `confirmedGuests(ctx)` → catálogo + asistencia confirmada;
  - un mapeo de los errores de dominio a HTTP;
  - en vamosjuntos-backend-005/main.go, `NewTransportStore`.
- [X] T008 [P] Agregar los tipos `Vehicle`, `TransportResponse` y `VehicleInput` a vamosjuntos-frontend-005/src/api/types.ts y los mensajes de contracts/api.md a vamosjuntos-frontend-005/src/messages.ts (contexto `transport`)

---

## Phase 3: User Story 1 - Ver vehículos y cupos (Priority: P1) 🎯 MVP

**Goal**: ver los vehículos por lugar de salida, con cupos y pasajeros, y quién no tiene transporte.
**Independent Test**: escenario 1 del quickstart, y ver datos sembrados.

- [X] T009 [US1] Agregar `GET /api/transport` (detrás de `requireGuest`) en vamosjuntos-backend-005/handlers.go:
  - `vehicles` resuelve conductor y pasajeros a `Guest`, omite a los que ya no son confirmados y calcula `free`;
  - ordena por `departure` y luego por `name`;
  - incluye `withoutTransport` y `confirmed`, en orden alfabético.
- [X] T010 [US1] Probar `GET /api/transport` en vamosjuntos-backend-005/handlers_test.go con un `fakeTransport` que reutiliza `applyTransport`
- [X] T011 [US1] Crear vamosjuntos-frontend-005/src/pages/TransportPage.tsx:
  - grupos "Sale de Bogotá" y "Sale de Tocancipá";
  - tarjetas con nombre, conductor, hora, nota, "N de M cupos libres" y pasajeros;
  - resumen "N confirmados sin transporte" con los nombres;
  - mensaje para registrar el primer vehículo cuando no hay ninguno;
  - botones de 44 px o más.
- [X] T012 [US1] Agregar la ruta `/transporte` en el bloque `confirmed` de vamosjuntos-frontend-005/src/App.tsx y el enlace "Transporte" en vamosjuntos-frontend-005/src/pages/HomePage.tsx, debajo de "Confirmar a otro invitado"

---

## Phase 4: User Story 2 - Registrar un vehículo (Priority: P1)

**Goal**: registrar un vehículo válido, que aparece con todos sus cupos libres.
**Independent Test**: escenarios 2 y 3 del quickstart.

- [X] T013 [US2] Agregar `POST /api/transport/vehicles` (201 `Vehicle`) en vamosjuntos-backend-005/handlers.go y probar `invalid_vehicle` y `driver_unavailable` en vamosjuntos-backend-005/handlers_test.go
- [X] T014 [US2] Crear vamosjuntos-frontend-005/src/pages/VehicleForm.tsx:
  - nombre;
  - conductor, con un selector de confirmados;
  - salida (Bogotá o Tocancipá);
  - cupos (1 a 24);
  - hora (`type="time"`, opcional);
  - nota (opcional, hasta 200).

  Sirve para crear y editar, y muestra los errores con messages.ts. Conectarlo en TransportPage con el botón "Registrar vehículo".

---

## Phase 5: User Story 3 - Reservar y liberar un cupo (Priority: P2)

**Goal**: reservar para uno mismo o para otro, cambiar de vehículo y liberar.
**Independent Test**: escenarios 4 a 9 y 12 del quickstart.

- [X] T015 [US3] Agregar `POST /api/transport/vehicles/{id}/passengers` y `DELETE /api/transport/vehicles/{id}/passengers/{guestId}` en vamosjuntos-backend-005/handlers.go, y probar reservar, cambiar de vehículo, `vehicle_full`, `guest_is_driver`, `guest_not_confirmed`, liberar y `passenger_not_found` en vamosjuntos-backend-005/handlers_test.go
- [X] T016 [US3] En vamosjuntos-frontend-005/src/pages/TransportPage.tsx:
  - "Reservar mi cupo" o "Cambiarme aquí", que no aparece si el usuario conduce o ya va en ese vehículo;
  - "Reservar para…", con un selector de los confirmados sin transporte;
  - "Liberar" en cada pasajero;
  - volver a pedir los datos tras cada acción.

---

## Phase 6: User Story 4 - Editar o eliminar un vehículo (Priority: P3)

**Goal**: editar respetando los pasajeros, y eliminar dejando a los pasajeros sin transporte.
**Independent Test**: escenarios 10 y 11 del quickstart.

- [X] T017 [US4] Agregar `PUT` y `DELETE /api/transport/vehicles/{id}` en vamosjuntos-backend-005/handlers.go, y probar `seats_below_passengers`, `vehicle_not_found` y que eliminar deja a los pasajeros sin transporte en vamosjuntos-backend-005/handlers_test.go
- [X] T018 [US4] En vamosjuntos-frontend-005/src/pages/TransportPage.tsx, botón "Editar", que abre VehicleForm con los datos, y "Eliminar" con un paso de confirmación en la misma tarjeta ("¿Eliminar? Sus pasajeros quedarán sin transporte")

---

## Phase 7: Polish

- [X] T019 Pasar las compuertas en los worktrees: backend (`gofmt` sin CR, `vet`, `golangci-lint`, `test`) y frontend (`build`, `lint`)
- [X] T020 Validar en local contra Firebase y Firestore reales los escenarios del quickstart, con cuentas de prueba y Chrome a 360 px, incluido el escenario 12 (reservas simultáneas). Luego borrar los vehículos, cuentas, vínculos y respuestas de prueba
- [X] T021 Hacer commit en las ramas `005-modulo-transporte` de los worktrees y en vamosjuntos.club (solo `specs/005-modulo-transporte`), y registrar aquí las notas de implementación
- [X] T022 (visto bueno, coordinado con las sesiones de la 003 y la 004) Integrar en `main` con rebase previo, publicar en `production` y probar en `vamosjuntos.club`

---

## Dependencies & Execution Order

- **Setup → Foundational**: T001 → T002 a T008. T004 y T008 son paralelas al resto.
- **Historias**: US1 → US2 → US3 → US4. US1 da la pantalla que usan las demás.
- **Polish**: al final. T022 espera a que las otras sesiones integren o no tengan conflictos.

## Parallel Example

```bash
Task: "transport_test.go (T004)"
Task: "tipos y mensajes del frontend (T008)"
```

## Implementation Strategy

- **MVP**: US1 + US2, es decir, ver y registrar vehículos.
- **Luego**: US3 (reservas), que da el valor principal, y US4 (editar o eliminar).

## Implementation Notes (2026-10-30)

- **Backend** (worktree `vamosjuntos-backend-005`, rama `005-modulo-transporte`, con rebase sobre
  la 003):
  - `transport.go`: regla pura `applyTransport` y `TransportStore` con transacción sobre todos los
    vehículos;
  - `transport_handlers.go`: los handlers van en un archivo aparte para reducir conflictos con
    otras sesiones;
  - pruebas en `transport_test.go` y `transport_handlers_test.go`;
  - en `handlers.go` solo cambiaron el struct, `NewServer` y las rutas. Hubo un conflicto con
    `GET /api/event` de la 003 y se resolvió dejando las dos rutas.
- **Frontend** (worktree `vamosjuntos-frontend-005`):
  - `TransportPage.tsx` y `VehicleForm.tsx`;
  - `PUT` y `DELETE` en `apiFetch`;
  - ruta `/transporte` y botón "Transporte: vehículos y cupos" en la pantalla principal, debajo de
    la card de la 003.
- **Desviación del contrato**: crear, editar y reservar responden con el estado completo
  (`TransportResponse`) en lugar de un solo `Vehicle`, para que la pantalla no haga otra llamada.
  Eliminar y liberar responden `204`.
- **Validación** contra Firebase y Firestore reales:
  - API: 14/14, incluida la carrera por el último cupo (`200` y `409`);
  - UI a 360 px: 12/12 (pantalla principal, vacío, formulario, registrar, reservar para otro,
    liberar y eliminar con confirmación);
  - compuertas en verde: Go con 0 issues y build y lint del frontend.
- **No automatizado**: escribir la hora en `input type="time"` desde CDP no fija el valor. La API
  sí guarda y devuelve `06:30`.
- **Limpieza**: 0 vehículos y 0 cuentas de prueba. Quedan la cuenta real del responsable
  (`men***@gmail.com`, vinculada a Sergio) y la confirmación de Sergio.

## Publicación (2026-10-30)

- **Publicado**:
  - backend `96271fa` (revisión `vamosjuntos-backend-00005-nw8`) y frontend `4c3270b`, por
    pipeline y en verde;
  - `GET /api/transport` responde `401` sin token;
  - el bundle publicado trae "Transporte: vehículos y cupos".
- **Worktrees** `vamosjuntos-backend-005` y `vamosjuntos-frontend-005` quitados; la rama ya está
  en `main`.
- **Pendiente (T022)**: probar el módulo en `vamosjuntos.club` con sesión iniciada. La sesión que
  publicó se cortó justo después del deploy.

### Validación en producción T022 (2026-10-30)

- **API, con 5 cuentas de prueba**: escenarios 2 a 13 en verde, incluida la carrera por el último
  cupo (`200`/`409`). Los vehículos de prueba se llamaron "[PRUEBA] …" y solo se borraron sus ids;
  el vehículo real quedó intacto.
- **No aplica en producción**: el escenario 1 ("sin vehículos"), porque ya hay un vehículo real.
  Quedó cubierto por T020 en local.
- **UI a 360 px**: la pantalla abre con los vehículos y "sin transporte", sin scroll horizontal.
