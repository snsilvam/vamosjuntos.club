# Tasks: Confirmar a otro invitado sin su código

**Input**: Design documents from `/specs/009-confirmar-otro-sin-codigo/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/api.md, quickstart.md

**Tests**: se incluyen pruebas de Go para la regla pura y los handlers, como en la 005 a la 008. El
Principio IV exige que la lógica de negocio sea testeable sin Firestore.

**Organization**: por historia de usuario. Código en ramas `009-confirmar-otro-sin-codigo` de
`vamosjuntos-backend` y `vamosjuntos-frontend`.

## Format: `[ID] [P?] [Story] Description`

---

## Phase 1: Setup

- [X] T001 Crear la rama `009-confirmar-otro-sin-codigo` desde `main` en `vamosjuntos-backend` y en `vamosjuntos-frontend`

---

## Phase 2: Foundational

**Purpose**: la regla pura y los accesos a Firestore que usan las dos historias.

- [X] T002 En vamosjuntos-backend/links.go:
  - función `linksCollection(client, eventID)` con la ruta `events/{EVENT_ID}/links`, usada por `LinkStore.collection`;
  - `LinkStore.Linked(ctx) (map[string]bool, error)`: los `guestId` con vínculo, leyendo solo los ids (`Select()` vacío).
- [X] T003 En vamosjuntos-backend/attendance.go:
  - regla pura `confirmOther(current Status, linked bool) (Status, error)` según la tabla de data-model.md: primero `nextStatus(current, actionConfirmOther)` (`errAlreadyResponded`), luego `linked` → `errGuestLinked`;
  - `AttendanceStore.ConfirmOther(ctx, guestID, by string) error`: transacción que lee `links/{guestId}` (vía `linksCollection`) y `attendance/{guestId}`, aplica `confirmOther` y guarda `status`, `respondedAt` y `respondedBy`.
- [X] T004 [P] Agregar a vamosjuntos-backend/attendance_test.go la tabla de `confirmOther`: pendiente sin vínculo → `confirmed`; pendiente con vínculo → `errGuestLinked`; confirmado o "No asistirá", con o sin vínculo → `errAlreadyResponded`
- [X] T005 En vamosjuntos-backend/handlers.go, agregar `ConfirmOther` a `attendanceRecorder` y `Linked` a `linkStore`; en vamosjuntos-backend/handlers_test.go, implementarlos en `fakeAttendance` (con acceso a `fakeLinks`, usando `confirmOther`) y en `fakeLinks`

**Checkpoint**: la regla pura pasa sus pruebas y el servidor compila con los métodos nuevos.

---

## Phase 3: User Story 1 - Confirmar a otro invitado eligiendo su nombre (Priority: P1) 🎯 MVP

**Goal**: confirmar a un pendiente sin cuenta sin escribir su código, con pregunta previa.

**Independent Test**: escenarios 1 a 4 y 9 del quickstart.

- [X] T006 [US1] Agregar `GET /api/guests/confirmable` con `requireGuest` en vamosjuntos-backend/handlers.go: evento (`404 event_not_found`), catálogo, respuestas y `Linked`; devuelve `{"guests": [...]}` con los pendientes sin vínculo en el orden de `Catalog.Sorted`
- [X] T007 [US1] Cambiar `handleConfirmOther` en vamosjuntos-backend/handlers.go: sin leer el cuerpo ni llamar a `checkCode`; `404 guest_not_found` si no está en el catálogo; `attendance.ConfirmOther(ctx, target.ID, me.ID)`; `409 already_responded` con `errAlreadyResponded`; `200 {"status":"confirmed","guest"}`
- [X] T008 [US1] En vamosjuntos-backend/handlers_test.go:
  - reescribir `TestConfirmOther`: confirma sin cuerpo, `respondedBy` es quien confirma, no crea vínculos, no registra intentos, `guest_not_found` y `already_responded` al repetir;
  - prueba de que un cuerpo `{"code": "cualquiera"}` se ignora (research R4);
  - `TestConfirmableGuests`: solo pendientes sin vínculo, en orden, sin códigos; `401` sin token y `403 not_linked` sin vínculo;
  - quitar el cuerpo con código de `TestProtectedRoutes` y `TestEventCountsChangeAfterConfirmingOther`.
- [X] T009 [US1] Reescribir vamosjuntos-frontend/src/pages/ConfirmOtherPage.tsx:
  - lista desde `GET /api/guests/confirmable`; subtítulo sin mención al código;
  - al tocar un nombre, "¿Confirmar a {nombre}?" con "Quedará en la lista de confirmados del paseo. Si te equivocas, solo el organizador puede deshacerlo.", "Sí, confirmar a {nombre}" y "Cancelar";
  - `POST` sin cuerpo; en éxito "¡{nombre} quedó confirmado!" y recarga;
  - lista vacía: "No hay invitados pendientes sin cuenta. Quien ya entró con su cuenta responde desde ahí.".
- [X] T010 [P] [US1] En vamosjuntos-frontend/src/messages.ts, quitar el caso de `invalid_code` para el contexto `other` ("Ese código no corresponde a {nombre}.")

**Checkpoint**: se confirma a otro con 3 toques y sin código (MVP).

---

## Phase 4: User Story 2 - Quien ya tiene cuenta responde por sí mismo (Priority: P1)

**Goal**: nadie confirma a un pendiente con cuenta vinculada, ni siquiera en una carrera.

**Independent Test**: escenarios 5 a 8 del quickstart.

- [X] T011 [US2] En `handleConfirmOther` de vamosjuntos-backend/handlers.go, responder `409 guest_already_linked` con `errGuestLinked`, y en vamosjuntos-backend/handlers_test.go probar que un pendiente vinculado no aparece en `GET /api/guests/confirmable` y que confirmarlo responde `409 guest_already_linked` sin cambiar su respuesta
- [X] T012 [US2] En vamosjuntos-frontend/src/messages.ts, `guest_already_linked` en el contexto `other` → "{nombre} ya entró con su propia cuenta y responde desde ahí."; en vamosjuntos-frontend/src/pages/ConfirmOtherPage.tsx, recargar la lista ante `404` y `409`

**Checkpoint**: las dos historias funcionan y las pruebas de Go cubren la carrera con la regla pura.

---

## Phase 5: Polish & Cross-Cutting Concerns

- [X] T013 Pasar las compuertas: backend (`gofmt -l .` sin salida, `go vet ./...`, `golangci-lint run`, `go test ./...`) y frontend (`npm run lint`, `npm run build`)
- [X] T014 Marcar el reemplazo parcial en specs/008-autenticacion-firebase/spec.md (US6, FR-013 y FR-014) y agregar la 009 al README.md
- [ ] T015 (visto bueno) Integrar en `main`, publicar backend y luego frontend y recorrer el quickstart en `vamosjuntos.club` con cuentas de prueba, borrando solo lo creado
- [X] T016 Registrar aquí las notas de implementación y hacer commit de specs/009-confirmar-otro-sin-codigo

---

## Dependencies & Execution Order

- **Setup → Foundational**: T001 → T002 → T003 → T005. T004 es paralela a T005.
- **Historias**: US1 → US2. Las dos tocan `handleConfirmOther` y `ConfirmOtherPage.tsx`; US2 agrega
  el caso del vínculo sobre lo de US1.
- **Polish**: al final. T015 publica el backend antes que el frontend (research R4).

## Parallel Example

```bash
Task: "tabla de confirmOther (T004)"
Task: "messages.ts (T010) mientras se hace ConfirmOtherPage.tsx (T009)"
```

## Implementation Strategy

- **MVP**: Setup, Foundational y US1. Con eso ya no se pide el código.
- **US2 antes de publicar**: sin ella, quitar el código permitiría confirmar a quien ya usa la app.
  Las dos se publican juntas.

## Implementation Notes (2026-11-06)

- **Backend**, commit `40231e2` en la rama `009-confirmar-otro-sin-codigo`, integrado en `main` local:
  - `links.go`: `linksCollection` (la ruta que comparten los dos almacenes) y `LinkStore.Linked`,
    que lee solo los ids;
  - `attendance.go`: regla pura `confirmOther`, `AttendanceStore.ConfirmOther` (transacción con el
    vínculo y la respuesta) y `statusIn`, el helper que ahora comparte con `Respond` para leer la
    respuesta dentro de la transacción;
  - `handlers.go`: `GET /api/guests/confirmable` y `handleConfirmOther` sin cuerpo, sin código y
    sin intentos;
  - pruebas: `TestConfirmOtherRule` (tabla), `TestConfirmOther`, `TestConfirmOtherIgnoresBody`,
    `TestConfirmableGuests`, `TestConfirmOtherRejectsGuestWithAccount` y `TestProtectedRoutes` sobre
    las dos rutas.
- **Frontend**, commit `11ef2db` en la rama `009-confirmar-otro-sin-codigo` (desde `main` `8a9051a`), integrado en `main` local:
  - `ConfirmOtherPage.tsx`: lista de `/api/guests/confirmable`, pregunta previa en la misma
    pantalla, `POST` sin cuerpo y recarga ante `404`/`409`;
  - `messages.ts`: `guest_already_linked` en el contexto `other`; sale el `invalid_code` de "otro".
- **Desviaciones**: ninguna respecto al contrato.
- **Validación**:
  - compuertas en verde: `gofmt` (sin cambios al normalizar CRLF), `go vet`, `golangci-lint`
    (0 issues), `go test`, `npm run lint` y `npm run build`;
  - falta la UI a 360 px y el quickstart en producción (T015).
- **Publicación (T015)**: falta subir `main` a `origin` y luego `git push origin main:production`,
  primero el backend y después el frontend (research R4).
