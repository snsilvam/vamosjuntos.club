---

description: "Lista de tareas de la feature 001-onboarding-sesion"
---

# Tasks: Onboarding y Sesión

**Input**: Design documents from `/specs/001-onboarding-sesion/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/api.md](contracts/api.md), [quickstart.md](quickstart.md)

**Tests**: El spec no pide TDD. Sí se incluyen pruebas unitarias de Go para la lógica pura y los
handlers, porque el plan las define (research R10) y la constitución exige aislar la lógica para
que sea testeable. Se escriben junto a la implementación, no antes. No hay pruebas automáticas en
el frontend.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

Las rutas son relativas a `C:\golang\vamosjuntos-app\`, donde viven los tres repos hermanos:

- `vamosjuntos-backend/`: paquete `main` plano de Go, módulo `github.com/snsilvam/vamosjuntos-backend`.
- `vamosjuntos-frontend/`: React + TypeScript + Vite.
- Este repo (`vamosjuntos.club/`) no recibe código.

Reglas que aplican a todas las tareas:

- **Go**: constructores `NewXxx`, `context.Context` como primer parámetro en el I/O, errores
  envueltos con `fmt.Errorf("...: %w", err)`, sin `panic` ni estado global.
- **Interfaces**: pequeñas y definidas en `handlers.go`, que es donde se consumen.
- **Códigos de invitación**: nunca se registran en logs.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Inicializar ambos repos con el stack del plan.

- [X] T001 Inicializar el módulo Go `github.com/snsilvam/vamosjuntos-backend` con `go 1.26` y agregar la dependencia `cloud.google.com/go/firestore` en vamosjuntos-backend/go.mod
- [X] T002 [P] Crear vamosjuntos-backend/.gitignore que excluya binarios compilados y archivos `.env*` (donde se guardan `GUESTS_JSON` y `SESSION_KEY` locales)
- [X] T003 [P] Generar la app con la plantilla `react-ts` de Vite en vamosjuntos-frontend/ (React 19, TypeScript 5), instalar `react-router` v7, borrar los assets y el CSS de demostración y dejar vamosjuntos-frontend/src/main.tsx renderizando `<App />`
- [X] T004 Instalar `tailwindcss` y `@tailwindcss/vite` v4, registrar el plugin en vamosjuntos-frontend/vite.config.ts y dejar `@import "tailwindcss";` como único contenido de vamosjuntos-frontend/src/index.css (depende de T003)
- [X] T005 Configurar en vamosjuntos-frontend/vite.config.ts el proxy de desarrollo `server.proxy['/api'] = 'http://localhost:8080'` (depende de T004)
- [X] T006 [P] Asegurar que vamosjuntos-frontend/.gitignore excluya `node_modules`, `dist` y `.env*` (depende de T003)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Catálogo, evento, reglas de asistencia, límite de intentos, sesión y bases del
frontend, que usan todas las historias.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Backend

- [X] T007 [P] Implementar en vamosjuntos-backend/errors.go `writeJSON(w, status, v)` y `writeError(w, status, code)`, que escribe `{"error":{"code":"<código>"}}`, y `writeTooManyAttempts(w, retryAfter time.Duration)`, que responde `429` con `{"error":{"code":"too_many_attempts","retryAfterSeconds":<n>}}` y el encabezado `Retry-After`
- [X] T008 [P] Implementar en vamosjuntos-backend/config.go el struct `Config` y `loadConfig() (Config, error)`, que lee `GUESTS_JSON`, `SESSION_KEY` (base64 que debe decodificar a 32 bytes), `EVENT_ID`, `GOOGLE_CLOUD_PROJECT` y `PORT` (por defecto `8080`); los errores nombran la variable que falla pero nunca incluyen su valor
- [X] T009 [P] Implementar en vamosjuntos-backend/catalog.go `Guest{ID, Name, code}` y `parseCatalog(raw string) (Catalog, error)` para el formato `[["Nombre","código"], ...]`. Usa el slug "minúsculas, sin tildes ni eñes, espacios → `-`" con un `strings.Replacer` de la biblioteca estándar, y valida verbatim: "Máximo 25 invitados (FR-013)", nombre "no vacío", código "≥ 6 caracteres tras recortar espacios; único sin distinguir mayúsculas y minúsculas (FR-016)" e `id` "único". Agrega los métodos `ByID(id)`, `ByCode(code)`, `Matches(guestID, code) bool` (con `strings.EqualFold(strings.TrimSpace(input), code)`), `Len()` y `Sorted()` (orden alfabético sin distinguir tildes)
- [X] T010 [P] Escribir pruebas de tabla en vamosjuntos-backend/catalog_test.go: "papá de lore"→`papa-de-lore`, "Nicolás López"→`nicolas-lopez`, "El Fercho"→`el-fercho`; rechazo de 26 invitados, de los códigos `"Abc123"` y `" abc123 "` repetidos, de un código de 5 caracteres y de un nombre vacío; `Matches` acepta `" PRUEBA01 "` para `prueba01`
- [X] T011 [P] Implementar en vamosjuntos-backend/event.go el struct `Event` (`name`, `startsAt`, `endsAt`, `destination`, `departurePlaces`, `capacity`, con tags `firestore`) y `eventStore` con `NewEventStore(client, eventID)` y `Get(ctx) (Event, error)`, que lee `events/{EVENT_ID}` y devuelve `errEventNotFound` si el documento no existe, si falta un campo obligatorio o si no se cumple "`endsAt` > `startsAt`"; además `EventSummary` con fechas en RFC 3339 en `America/Bogota` (`-05:00`), `sessionExpiry(e Event) time.Time` (que devuelve `EndsAt + 7*24h`) e `import _ "time/tzdata"`
- [X] T012 [P] Implementar en vamosjuntos-backend/attendance.go la parte pura: el tipo `Status` (`""` = pendiente, `"confirmed"`, `"declined"`), las acciones `actionConfirm`, `actionDecline` y `actionConfirmOther`, y `nextStatus(current Status, a action) (Status, error)` según la tabla de transiciones de data-model.md. Los errores `errAlreadyConfirmed` (confirmado + declinar) y `errAlreadyResponded` (confirmar a otro que no está pendiente) se mapean a 409
- [X] T013 [P] Escribir en vamosjuntos-backend/attendance_test.go una prueba de tabla con las 7 filas de la tabla de transiciones de data-model.md
- [X] T014 Implementar en vamosjuntos-backend/attendance.go `attendanceStore` con `NewAttendanceStore(client, eventID)`, `Statuses(ctx) (map[string]Status, error)`, que lee todos los documentos de `events/{EVENT_ID}/attendance`, y `Respond(ctx, guestID string, a action, respondedBy string) (Status, error)`, que dentro de `client.RunTransaction` lee el documento, aplica `nextStatus` y, solo si el estado cambia, escribe `{status, respondedAt: firestore.ServerTimestamp, respondedBy}` (depende de T012)
- [X] T015 [P] Implementar en vamosjuntos-backend/attempts.go la parte pura: `attemptState{Failures int, BlockedUntil time.Time}`, `blockedFor(s, now) time.Duration` y `registerAttempt(s attemptState, ok bool, now time.Time) attemptState`, con la regla "con 5 fallos seguidos bloquea 15 minutos, y un acierto reinicia el contador"
- [X] T016 [P] Escribir en vamosjuntos-backend/attempts_test.go pruebas para 4 fallos (sin bloqueo), el 5.º fallo (bloqueado 15 min), un acierto que reinicia el contador y el bloqueo vencido
- [X] T017 Implementar en vamosjuntos-backend/attempts.go `attemptStore` con `NewAttemptStore(client)`, `Blocked(ctx, deviceID) (time.Duration, error)` y `Record(ctx, deviceID string, ok bool) error`. Usa una transacción sobre `loginAttempts/{deviceId}` con los campos `failures` y `blockedUntil` y llama a `registerAttempt` (depende de T015)
- [X] T018 [P] Implementar en vamosjuntos-backend/session.go `signToken(key []byte, guestID string, exp time.Time) string` y `verifyToken(key []byte, token string, now time.Time) (string, error)` con el formato `base64url(payload) + "." + base64url(HMAC-SHA256(payload))` y el payload `{"g":"<guestId>","exp":<unix>}`; la firma se compara con `hmac.Equal`. Además `setSessionCookie(w, token, exp)` (`__session`, `HttpOnly; Secure; SameSite=Lax; Path=/`, `Expires=exp`) y `clearSessionCookie(w)` (`Max-Age=0`)
- [X] T019 [P] Escribir en vamosjuntos-backend/session_test.go pruebas de ida y vuelta, payload alterado, llave incorrecta, token vencido y token mal formado
- [X] T020 Crear en vamosjuntos-backend/handlers.go el struct `server` y `NewServer(catalog Catalog, events eventGetter, attendance attendanceRecorder, attempts attemptLimiter, sessionKey []byte)`, con interfaces pequeñas declaradas en el mismo archivo: `eventGetter{Get}`, `attendanceRecorder{Statuses, Respond}` y `attemptLimiter{Blocked, Record}`. Agregar `routes() *http.ServeMux` y los helpers: `decodeJSON` (devuelve `400 invalid_request`), `deviceID(r)` (lee `X-Device-Id`; si falta, `400 missing_device_id`) y `checkCode(ctx, w, deviceID string, ok bool) bool` (consulta el bloqueo y responde `429` si aplica, y registra el intento) (depende de T007, T009, T011, T014, T017)
- [X] T021 Implementar en vamosjuntos-backend/session.go el middleware `(s *server) requireSession(next func(http.ResponseWriter, *http.Request, Guest)) http.HandlerFunc`. Si la cookie falta o es inválida o está vencida, responde `401 unauthenticated`; si el `guestId` ya no está en el catálogo o su estado en `Statuses` no es `confirmed`, responde `403 not_invited` (FR-008). Registrar `GET /api/session`, que devuelve `{"guest":{"id","name"}}`, en `routes()` de vamosjuntos-backend/handlers.go (depende de T018, T020)
- [X] T022 Implementar vamosjuntos-backend/main.go: `loadConfig`, `parseCatalog` y `firestore.NewClient`. Si existe el evento y `Catalog.Len() > event.Capacity`, termina con error (regla "como máximo `capacity` del evento"). Ante cualquier error, `log.Printf` sin valores secretos y `os.Exit(1)`. Arrancar `http.Server` con `ReadHeaderTimeout` y apagarlo ordenadamente con `signal.NotifyContext(SIGINT, SIGTERM)` + `Shutdown` (depende de T008, T020, T021)

### Frontend

- [X] T023 [P] Implementar en vamosjuntos-frontend/src/api/deviceId.ts `getDeviceId(): string`, que lee o guarda `crypto.randomUUID()` en `localStorage['vj.deviceId']`; todo acceso a `localStorage` va en `try/catch`, con un valor en memoria como respaldo
- [X] T024 [P] Implementar en vamosjuntos-frontend/src/api/client.ts `apiFetch<T>(path, { method, body, withDeviceId })` con `credentials: 'same-origin'` y JSON, que lanza `ApiError { status, code, retryAfterSeconds? }` a partir de `{"error":{"code"}}` (los errores de red se reportan con `code: 'network'`), y un registro `onUnauthorized(cb)` que se invoca ante `401`/`403` en rutas protegidas
- [X] T025 [P] Definir en vamosjuntos-frontend/src/api/types.ts las interfaces `Guest { id, name }` y `EventSummary { name, startsAt, endsAt, destination, departurePlaces }` según contracts/api.md
- [X] T026 [P] Implementar en vamosjuntos-frontend/src/messages.ts `friendlyMessage(error: ApiError, context: 'list' | 'reentry' | 'other', vars?)` con exactamente los textos de la tabla "Mensajes de la UI por código de error" de contracts/api.md; cualquier código desconocido devuelve "Algo salió mal. Intenta de nuevo en un momento."
- [X] T027 [P] Implementar en vamosjuntos-frontend/src/format.ts `formatEventDate(iso: string)` con `Intl.DateTimeFormat('es-CO', { timeZone: 'America/Bogota', weekday: 'long', day: 'numeric', month: 'long', year: 'numeric', hour: 'numeric', minute: '2-digit', hour12: true })`
- [X] T028 [P] Implementar en vamosjuntos-frontend/src/session/SessionProvider.tsx un Context y el hook `useSession()`, con `{ status: 'loading' | 'authenticated' | 'anonymous', guest, setGuest(guest), clear(), refresh() }`; `refresh()` llama `GET /api/session` al montar
- [X] T029 [P] Implementar en vamosjuntos-frontend/src/onboarding/OnboardingContext.tsx un Context en memoria con `{ guest, code, event }` para pasar datos entre los pasos del onboarding; el código nunca se guarda en `localStorage` ni en la URL
- [X] T030 Crear vamosjuntos-frontend/src/App.tsx con `BrowserRouter`, `SessionProvider` y `OnboardingProvider`, y las rutas `/`, `/codigo`, `/confirmar`, `/ya-confirme`, `/inicio` y `/confirmar-otro`, cada una con un componente provisional (depende de T028, T029)

**Checkpoint**: `go build ./...`, `go test ./...` y `npm run build` pasan; el backend arranca contra el emulador.

---

## Phase 3: User Story 1 - Primer ingreso y confirmación de asistencia (Priority: P1) 🎯 MVP

**Goal**: Un invitado elige su nombre, valida su código, confirma y llega a la pantalla principal con sesión.

**Independent Test**: Escenarios 1, 2 y 3 de [quickstart.md](quickstart.md). Después de confirmar, el nombre no aparece en la lista desde otra ventana.

- [X] T031 [US1] Implementar `GET /api/onboarding/guests` en vamosjuntos-backend/handlers.go: `events.Get`, que si falla responde `404 event_not_found`; luego `Statuses` y devolver `{"guests":[...]}` con solo los pendientes, tomados de `Catalog.Sorted()`
- [X] T032 [US1] Implementar `POST /api/onboarding/verify` en vamosjuntos-backend/handlers.go según contracts/api.md: `deviceID`, `decodeJSON {guestId, code}`, `404 event_not_found` o `guest_not_found`, y `409 already_responded` si el estado no es pendiente. Luego `checkCode` con `Catalog.Matches`; si el código es incorrecto, `401 invalid_code`; si es correcto, `200 {"event": EventSummary}`
- [X] T033 [US1] Implementar `POST /api/onboarding/respond` con `attending: true` en vamosjuntos-backend/handlers.go: valida de nuevo el código con `checkCode` y llama `Respond(ctx, guestID, actionConfirm, guestID)`, que acepta los estados pendiente y "No asistirá" y es idempotente si ya está confirmado. Firma el token con `exp = sessionExpiry(event)` y responde con `setSessionCookie` y `200 {"status":"confirmed","guest":{...}}`
- [X] T034 [US1] Escribir en vamosjuntos-backend/handlers_test.go pruebas con `httptest` y fakes de `eventGetter`, `attendanceRecorder` y `attemptLimiter`: la lista excluye confirmados y "No asistirá"; `verify` con código incorrecto responde `401` y registra el fallo; el 6.º intento responde `429`; `respond` confirmado incluye `Set-Cookie: __session`; un evento ausente responde `404` (depende de T031–T033)
- [X] T035 [P] [US1] Implementar vamosjuntos-frontend/src/pages/GuestListPage.tsx: pide la lista y muestra un botón grande por nombre, que guarda el invitado en `OnboardingContext` y lleva a `/codigo`. Incluye el enlace "Ya confirmé, entrar con mi código" a `/ya-confirme`, un mensaje cuando la lista está vacía y `friendlyMessage` para `event_not_found`. Estilos Tailwind pensados primero para celular
- [X] T036 [P] [US1] Implementar vamosjuntos-frontend/src/pages/CodePage.tsx: campo de código y llamada a `POST /api/onboarding/verify` con `X-Device-Id`. En `200`, guarda `code` y `event` en `OnboardingContext` y va a `/confirmar`; en error muestra `friendlyMessage(e, 'list')`, con los minutos de bloqueo cuando hay `429`
- [X] T037 [P] [US1] Implementar vamosjuntos-frontend/src/pages/ConfirmPage.tsx: muestra nombre, destino, lugares de salida y fechas con `formatEventDate`, y la pregunta "¿Confirmas tu asistencia?". "Sí" llama `respond` con `attending: true`, luego `setGuest` y navega a `/inicio`. Si no hay datos en `OnboardingContext`, redirige a `/`
- [X] T038 [P] [US1] Implementar vamosjuntos-frontend/src/pages/HomePage.tsx como esqueleto con "¡Hola, {guest.name}!" y espacio reservado para la card (spec 003) y el banner (spec 004)
- [X] T039 [US1] Conectar las páginas de T035–T038 en vamosjuntos-frontend/src/App.tsx, reemplazando las provisionales de `/`, `/codigo`, `/confirmar` e `/inicio`

**Checkpoint**: El flujo de primer ingreso funciona de punta a punta en local.

---

## Phase 4: User Story 2 - Acceso protegido por sesión (Priority: P1)

**Goal**: Ninguna pantalla u operación interna es accesible sin la sesión de un invitado confirmado.

**Independent Test**: Escenarios 4, 5 y 12 de [quickstart.md](quickstart.md).

- [X] T040 [US2] Agregar a vamosjuntos-backend/handlers_test.go pruebas para `GET /api/session`: sin cookie responde `401 unauthenticated`; con firma alterada, `401`; con invitado fuera del catálogo, `403 not_invited`; con invitado en estado "No asistirá", `403`; con invitado confirmado, `200`
- [X] T041 [P] [US2] Implementar vamosjuntos-frontend/src/session/RequireSession.tsx: con `status === 'loading'` muestra un indicador de carga; con `anonymous`, `<Navigate to="/" replace />`; si no, muestra los hijos
- [X] T042 [US2] En vamosjuntos-frontend/src/App.tsx, envolver `/inicio` y `/confirmar-otro` con `RequireSession` y registrar `onUnauthorized(() => { clear(); navigate('/') })` para que cualquier `401`/`403` en rutas protegidas devuelva al onboarding (depende de T041)
- [X] T043 [US2] En vamosjuntos-frontend/src/App.tsx, redirigir `/` a `/inicio` cuando la sesión ya está autenticada, para que un confirmado no vuelva a ver la lista

**Checkpoint**: US1 y US2 juntas forman el MVP desplegable.

---

## Phase 5: User Story 3 - Volver a entrar con el código (Priority: P2)

**Goal**: Un confirmado recupera la sesión desde cualquier dispositivo solo con su código, y un "No asistirá" puede cambiar de opinión.

**Independent Test**: Escenarios 8 y 9 de [quickstart.md](quickstart.md).

- [X] T044 [US3] Implementar `POST /api/session/reentry` en vamosjuntos-backend/handlers.go según contracts/api.md: `deviceID`, `decodeJSON {code}` y `Catalog.ByCode` con `checkCode`. Si el código no existe, `401 invalid_code`; si el invitado está confirmado, cookie y `200 {"status":"confirmed","guest"}`; si está en "No asistirá", `200 {"status":"declined","guest","event"}` sin cookie; si está pendiente, `409 pending_use_list`
- [X] T045 [US3] Agregar a vamosjuntos-backend/handlers_test.go pruebas de reentry para los cuatro casos (confirmado, "No asistirá", pendiente y desconocido), y una prueba de que un código desconocido cuenta para el bloqueo (depende de T044)
- [X] T046 [P] [US3] Implementar vamosjuntos-frontend/src/pages/ReentryPage.tsx: campo de código y llamada a `/api/session/reentry`. Si la respuesta es `confirmed`, `setGuest` y va a `/inicio`; si es `declined`, guarda `{guest, code, event}` en `OnboardingContext` y va a `/confirmar` (ConfirmPage vuelve a preguntar); en error, `friendlyMessage(e, 'reentry')`
- [X] T047 [US3] Conectar `ReentryPage` a `/ya-confirme` en vamosjuntos-frontend/src/App.tsx (depende de T046)

---

## Phase 6: User Story 4 - Confirmar a otro invitado (Priority: P2)

**Goal**: Un confirmado confirma a otro invitado pendiente con el código de esa persona, sin cambiar su propia sesión.

**Independent Test**: Escenario 6 de [quickstart.md](quickstart.md).

- [X] T048 [US4] Implementar `POST /api/guests/{guestId}/confirmation` en vamosjuntos-backend/handlers.go, envuelto en `requireSession`. Si el invitado no existe, `404 guest_not_found`; con `checkCode` y un código incorrecto, `400 invalid_code` (no `401`, ver research R7); llama `Respond(ctx, guestId, actionConfirmOther, sessionGuest.ID)`, y si `errAlreadyResponded`, `409 already_responded`; si todo sale bien, `200 {"status":"confirmed","guest"}`, sin `Set-Cookie`
- [X] T049 [US4] Agregar a vamosjuntos-backend/handlers_test.go pruebas de que un código incorrecto responde `400` y no `401`, de que la respuesta no trae `Set-Cookie`, de que `respondedBy` es el invitado de la sesión y de que un invitado ya confirmado responde `409` (depende de T048)
- [X] T050 [P] [US4] Implementar vamosjuntos-frontend/src/pages/ConfirmOtherPage.tsx: lista de pendientes (`GET /api/onboarding/guests`), selección de una persona, campo para su código y envío. Si sale bien, muestra "¡{nombre} quedó confirmado!" y refresca la lista; en error, `friendlyMessage(e, 'other', { nombre })`
- [X] T051 [US4] Conectar `ConfirmOtherPage` a `/confirmar-otro` en vamosjuntos-frontend/src/App.tsx y agregar en vamosjuntos-frontend/src/pages/HomePage.tsx el enlace "Confirmar a otro invitado" (depende de T050)

---

## Phase 7: User Story 5 - Decir que no asistirá (Priority: P3)

**Goal**: Quien responde "No" queda como "No asistirá", sale de la lista y no obtiene sesión.

**Independent Test**: Escenario 7 de [quickstart.md](quickstart.md).

- [X] T052 [US5] Completar `POST /api/onboarding/respond` en vamosjuntos-backend/handlers.go para `attending: false`: llama `Respond(ctx, guestID, actionDecline, guestID)` y responde `200 {"status":"declined"}` sin cookie; si `errAlreadyConfirmed`, `409 already_confirmed`
- [X] T053 [US5] Agregar a vamosjuntos-backend/handlers_test.go pruebas de que declinar no trae `Set-Cookie`, de que un confirmado que declina recibe `409` y de que el invitado ya no aparece en `GET /api/onboarding/guests` (depende de T052)
- [X] T054 [US5] Agregar a vamosjuntos-frontend/src/pages/ConfirmPage.tsx el botón "No": llama `respond` con `attending: false`, muestra "¡Te vamos a extrañar! Si cambias de opinión, entra con tu código." y un botón para volver a `/`

---

## Phase 8: User Story 6 - Cerrar sesión (Priority: P3)

**Goal**: Un invitado termina su sesión en un dispositivo compartido.

**Independent Test**: Escenario 11 de [quickstart.md](quickstart.md).

- [X] T055 [US6] Implementar `DELETE /api/session` en vamosjuntos-backend/handlers.go: `clearSessionCookie` y `204`, aunque no haya sesión
- [X] T056 [US6] Agregar a vamosjuntos-backend/handlers_test.go una prueba de que `DELETE /api/session` responde `204` con `Set-Cookie` de `__session` y `Max-Age=0` (depende de T055)
- [X] T057 [US6] Agregar a vamosjuntos-frontend/src/pages/HomePage.tsx el botón "Cerrar sesión": `DELETE /api/session`, `clear()` y navegar a `/`

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: Compuertas de calidad y validación final.

- [X] T058 Revisar todos los `log.*` de vamosjuntos-backend/*.go para confirmar que ningún código de invitación, ni `GUESTS_JSON` ni `SESSION_KEY` aparece en logs ni en respuestas
- [X] T059 [P] Revisar en un ancho de 360 px cada página de vamosjuntos-frontend/src/pages/, con nombres largos ("Nicolás López", "papá de lore"), botones táctiles de al menos 44 px de alto y sin desbordes horizontales
- [X] T060 [P] Reemplazar vamosjuntos-backend/readme con un README.md breve: variables de entorno requeridas, cómo correr en local y un enlace a specs/001-onboarding-sesion/quickstart.md del repo vamosjuntos.club
- [X] T061 Ejecutar en vamosjuntos-backend `gofmt -l .` (sin salida), `go vet ./...`, `golangci-lint run` y `go test ./...`, y corregir todo hallazgo
- [X] T062 Ejecutar `npm run build` en vamosjuntos-frontend sin errores de tipos
- [X] T063 Ejecutar los 14 escenarios de specs/001-onboarding-sesion/quickstart.md y marcar cualquier desviación como bug

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias.
- **Foundational (Phase 2)**: depende de Setup y BLOQUEA todas las historias.
- **US1 (Phase 3)**: depende de Foundational.
- **US2 (Phase 4)**: depende de Foundational. El backend ya quedó en T021; la guarda del frontend necesita `/inicio`, que crea T038 (US1).
- **US3 (Phase 5)**: depende de Foundational. El caso "No asistirá" reutiliza `ConfirmPage` (T037) y el `respond` de T033; para probarlo de punta a punta necesita US5 (T052), que crea ese estado.
- **US4 (Phase 6)**: depende de US2, porque usa `requireSession` y la ruta protegida.
- **US5 (Phase 7)**: depende de US1 (amplía `respond` y `ConfirmPage`).
- **US6 (Phase 8)**: depende de US1 (`HomePage`).
- **Polish (Phase 9)**: depende de las historias que se quieran entregar.

### Within Each User Story

- Backend: handler → pruebas del handler.
- Frontend: página → conexión en `App.tsx`.
- `handlers.go` y `handlers_test.go` son archivos compartidos, así que sus tareas son secuenciales.

### Parallel Opportunities

- Setup: T002, T003 y T006 en paralelo con T001.
- Foundational: T007–T013, T015, T016, T018, T019 y T023–T029 son archivos distintos sin dependencias entre sí.
- En cada historia, las páginas del frontend `[P]` avanzan en paralelo con el handler del backend.

---

## Parallel Example: Foundational

```bash
# Lógica pura del backend, en paralelo:
Task: "T009 parseCatalog en vamosjuntos-backend/catalog.go"
Task: "T012 nextStatus en vamosjuntos-backend/attendance.go"
Task: "T015 registerAttempt en vamosjuntos-backend/attempts.go"
Task: "T018 signToken/verifyToken en vamosjuntos-backend/session.go"

# Bases del frontend, en paralelo:
Task: "T023 deviceId en vamosjuntos-frontend/src/api/deviceId.ts"
Task: "T024 apiFetch en vamosjuntos-frontend/src/api/client.ts"
Task: "T026 friendlyMessage en vamosjuntos-frontend/src/messages.ts"
```

## Parallel Example: User Story 1

```bash
# Mientras se implementan T031–T033 en handlers.go:
Task: "T035 GuestListPage en vamosjuntos-frontend/src/pages/GuestListPage.tsx"
Task: "T036 CodePage en vamosjuntos-frontend/src/pages/CodePage.tsx"
Task: "T037 ConfirmPage en vamosjuntos-frontend/src/pages/ConfirmPage.tsx"
Task: "T038 HomePage en vamosjuntos-frontend/src/pages/HomePage.tsx"
```

---

## Phase 10: Enmienda v2.0.0 — catálogo en Firestore (2026-10-30)

**Purpose**: el responsable pidió que la lista de invitados salga de Firestore
(`events/{EVENT_ID}/users`, campos `Nombre` y `código`) y no de `GUESTS_JSON`. Diseño en
[research.md R12](research.md) y [data-model.md](data-model.md).

- [X] T064 Enmendar la constitución a la v2.0.0 (Principios I y II, "Configuración y secretos" y "Dominio de la V1") y alinear spec.md, plan.md, research.md (R12), data-model.md y quickstart.md
- [X] T065 Reemplazar `parseCatalog` por `newCatalog([]Guest)` en vamosjuntos-backend/catalog.go, con las mismas reglas (1 a 25 invitados, nombre no vacío, código de 6 o más caracteres y único sin distinguir mayúsculas), y quitar `slug`: el `guestId` es el id del documento
- [X] T066 Agregar `GuestStore.Catalog(ctx)` en vamosjuntos-backend/catalog.go: lee `events/{EVENT_ID}/users` (`guestDoc{Nombre, código}`, ignora otros campos) y valida con `newCatalog`; los errores nombran invitados, nunca códigos
- [X] T067 Leer el catálogo en cada request mediante la interfaz `guestCatalog` y `loadCatalog` (500 `internal_error` si falla) en vamosjuntos-backend/handlers.go y en `requireSession` de vamosjuntos-backend/session.go
- [X] T068 Validar el catálogo de Firestore al arrancar en vamosjuntos-backend/main.go (sale con error si es inválido) y quitar `GUESTS_JSON` de vamosjuntos-backend/config.go
- [X] T069 Ajustar las pruebas: `newCatalog` en vamosjuntos-backend/catalog_test.go; `fakeGuests`, `TestCatalogChangesApplyRightAway` y `TestCatalogUnavailable` en vamosjuntos-backend/handlers_test.go
- [X] T070 Crear en `vamosjuntos-club-db` los 21 invitados que faltaban, con la forma del ejemplo del responsable (id automático, `Nombre`, `código`). Los códigos son aleatorios de 6 caracteres sin 0/O/1/I/L, generados sin imprimirse
- [X] T071 Verificar contra el Firestore real: el backend arranca y `GET /api/onboarding/guests` devuelve los 22 pendientes en orden, sin códigos; las reglas de Firestore niegan el acceso sin credenciales (403)
- [X] T072 Actualizar vamosjuntos-backend/README.md y quitar `GUESTS_JSON` del workflow del backend (spec 002)

---

## Implementation Strategy

### MVP First (US1 + US2)

1. Phase 1: Setup.
2. Phase 2: Foundational.
3. Phase 3: US1 (primer ingreso).
4. Phase 4: US2 (protección). Sin ella no se cumple el Principio II, así que no se publica US1 sola.
5. **STOP and VALIDATE**: escenarios 1 a 5 y 12 del quickstart.
6. Publicar con el pipeline de la spec 002.

### Incremental Delivery

1. MVP (US1 + US2).
2. US3 (reingreso), antes de que alguien cambie de celular.
3. US4 (confirmar a otros), cuando las familias empiecen a confirmar a los niños.
4. US5 y US6.
5. Polish y la validación completa del quickstart.

---

## Implementation Notes (2026-10-30)

Diferencias entre lo implementado y el texto original de las tareas, sin cambio funcional:

- T028/T029: cada Context se separó en dos archivos, `useXxx.ts` (contexto + hook) y
  `XxxProvider.tsx` (componente), para cumplir la regla `only-export-components` de oxlint:
  `src/session/useSession.ts` + `src/session/SessionProvider.tsx`, y
  `src/onboarding/useOnboarding.ts` + `src/onboarding/OnboardingProvider.tsx`.
- Se agregaron `src/components/Screen.tsx` (layout común) y `src/components/ui.tsx` (estilos de
  botones, `Alert` y `Loading`) para no repetir clases de Tailwind en cada página.
- T011: la zona horaria es `time.FixedZone("COT", -5h)` en lugar de `time/tzdata` +
  `LoadLocation`. Colombia no tiene horario de verano, así que el resultado es el mismo sin
  depender de la base de zonas horarias.
- T008: `GOOGLE_CLOUD_PROJECT` es opcional; en Cloud Run el proyecto se detecta solo
  (`firestore.DetectProjectID`).
- T003: la plantilla de Vite instaló TypeScript 7, Vite 8 y oxlint como linter (`npm run lint`).
- T008/T022: nueva variable opcional `FIRESTORE_DATABASE` (por defecto `(default)`), porque la base
  del proyecto `vamosjuntos-club` se llama `vamosjuntos-club-db`; se usa
  `firestore.NewClientWithDatabase`.
- T020: todas las respuestas de `/api` llevan `Cache-Control: no-store` (middleware `noStore`),
  porque dependen de la sesión y pasan por la CDN de Firebase Hosting.
- T050: `ConfirmOtherPage` quita de inmediato al invitado confirmado de la lista; antes se veía
  como pendiente mientras se recargaba.

### Validación T059/T063 (2026-10-30)

- T063 contra Firestore real (`vamosjuntos-club` / `vamosjuntos-club-db`) con un evento de prueba
  aparte (`events/prueba-onboarding-001`): 21/21 chequeos de API, más los escenarios 12 (`403`
  al salir del catálogo) y 14 (`404` sin evento). Incluye concurrencia: dos confirmaciones
  simultáneas del mismo invitado → una `200` y otra `409`. Al terminar se borraron todos los
  documentos de prueba; la base quedó vacía.
- T059 en Chrome headless con un viewport de 360 px: 21/21 chequeos (flujos completos, sin
  scroll horizontal, botones ≥ 44 px, fecha en hora Colombia).

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each task or logical group, in the corresponding repo (backend or frontend)
- Stop at any checkpoint to validate story independently
- No crear Dockerfile, workflows ni `firebase.json`: pertenecen a la spec 002
