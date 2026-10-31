---

description: "Lista de tareas de la feature 008-autenticacion-firebase"
---

# Tasks: Autenticación con Firebase

**Input**: Design documents from `/specs/008-autenticacion-firebase/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/api.md](contracts/api.md), [quickstart.md](quickstart.md)

**Tests**: la spec no pide TDD. Se escriben pruebas de Go junto al código para la lógica pura y los
handlers, con fakes de `tokenVerifier` y `linkStore` (research R9), igual que en la 001. En el
frontend, el build con chequeo de tipos. De punta a punta, el quickstart.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

Las rutas son relativas a `C:\golang\vamosjuntos-app\`. Referencia de kaizen:
`C:\golang\kaizen-app\front-kaizensnsilvam.com\src\auth\` y
`C:\golang\kaizen-app\kaizensnsilvam-backend\internal\infrastructure\firebase\auth.go`.

Reglas que aplican a todas las tareas:

- **Go**: paquete `main` plano, constructores `NewXxx`, interfaces pequeñas en `handlers.go`,
  `context.Context` en el I/O y errores envueltos.
- **Seguridad**: los códigos nunca salen del backend ni se escriben en logs. Los tokens nunca se
  registran en logs.
- **Etiquetas**:
  - **(responsable)**: requiere la consola de Firebase o GitHub.
  - **(visto bueno)**: publica en producción o crea datos de prueba, así que se hace solo con
    aprobación explícita.

---

## Phase 1: Setup (Shared Infrastructure)

- [X] T001 Crear la rama `008-autenticacion-firebase` desde `main` en vamosjuntos-backend y en vamosjuntos-frontend
- [X] T002 (responsable) Configurar Firebase Authentication según quickstart §1:
  1. registrar la app web `vamosjuntos-web`;
  2. habilitar los proveedores Google y Correo/contraseña;
  3. agregar `vamosjuntos.club` a los dominios autorizados;
  4. crear en `snsilvam/vamosjuntos-frontend` las **variables** de Actions `VITE_FIREBASE_API_KEY`, `VITE_FIREBASE_AUTH_DOMAIN`, `VITE_FIREBASE_PROJECT_ID` y `VITE_FIREBASE_APP_ID`.
- [X] T003 [P] Agregar el Admin SDK `firebase.google.com/go/v4@v4.21.0` en vamosjuntos-backend/go.mod
- [X] T004 [P] Instalar `firebase@^12.17.0` en vamosjuntos-frontend/package.json, crear vamosjuntos-frontend/.env.example con los cuatro `VITE_FIREBASE_*` (sin valores) y crear vamosjuntos-frontend/.env.local (ignorado) con los valores de T002

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: autenticación por token, vínculo y estado de la cuenta, que usan todas las historias.
Además, validar el riesgo R4 antes de construir las pantallas.

### Backend

- [X] T005 Quitar `SESSION_KEY` y hacer obligatoria `GOOGLE_CLOUD_PROJECT` en vamosjuntos-backend/config.go, porque el `aud` del token se compara con ella (research R3)
- [X] T006 Borrar vamosjuntos-backend/session.go y vamosjuntos-backend/session_test.go (HMAC y cookie `__session`). Mover lo que siga sirviendo, por ejemplo `sessionExpiry` si algo lo usa, o borrarlo si ya nadie lo usa
- [X] T007 Crear vamosjuntos-backend/auth.go:
  - `Account{UID, Email}`;
  - la interfaz `tokenVerifier { Verify(ctx, raw string) (Account, error) }`;
  - `NewFirebaseVerifier(ctx, projectID)`, que usa `firebase.NewApp` + `app.Auth` + `VerifyIDToken`, como en kaizen;
  - el middleware `requireAccount`: lee `Authorization: Bearer`; sin token o con token inválido responde `401 unauthenticated`, y nunca registra el token en logs.
- [X] T008 Crear vamosjuntos-backend/links.go:
  - los errores `errAccountLinked` y `errGuestLinked`;
  - `LinkStore` sobre `events/{EVENT_ID}/links/{guestId}` con `{uid, email, linkedAt}`;
  - `GuestOf(ctx, uid) (string, bool, error)`, con una consulta `where uid ==` limitada a 1;
  - `Link(ctx, uid, email, guestID)`: una transacción que consulta por `uid` (→ `errAccountLinked`) y luego hace `tx.Create` (si ya existe → `errGuestLinked`).
- [X] T009 Cambiar vamosjuntos-backend/attempts.go para que el id del documento de `loginAttempts` sea el `uid` y no el dispositivo; la regla `registerAttempt` no cambia
- [X] T010 Agregar a vamosjuntos-backend/errors.go los códigos `not_linked`, `account_already_linked` y `guest_already_linked`
- [X] T011 En vamosjuntos-backend/handlers.go:
  - definir la interfaz `linkStore` y quitar `sessionKey`, `deviceID` y `validDeviceID`;
  - `NewServer` recibe `tokenVerifier` y `linkStore`;
  - agregar la función `meFor(ctx, account)`, que calcula `Me{status, guest, event}` según data-model.md, "Estado de la cuenta";
  - agregar `GET /api/me` detrás de `requireAccount`.
- [X] T012 Agregar el middleware `requireGuest` (🔒C) en vamosjuntos-backend/auth.go: cuenta sin vínculo → `403 not_linked`; invitado fuera del catálogo o no confirmado → `403 not_invited`; si no, pasa el `Guest` al handler
- [X] T013 En vamosjuntos-backend/main.go, crear el verificador con `NewFirebaseVerifier(ctx, cfg.ProjectID)` y el `LinkStore`, y conectarlos en `NewServer`
- [X] T014 Actualizar vamosjuntos-backend/handlers_test.go con fakes de `tokenVerifier` y de `linkStore`, quitar las pruebas de `verify`, `reentry`, `session` y `logout`, y agregar `TestMe`: 401 sin token, `unlinked`, `pending` con evento, `declined` con evento, `confirmed` y 403 si el invitado salió del catálogo
- [X] T015 En vamosjuntos-backend/.github/workflows/backend-deploy.yml, quitar el paso "Validate secrets" y `SESSION_KEY`, y agregar `GOOGLE_CLOUD_PROJECT=$PROJECT_ID` a `--set-env-vars`, como en kaizen
- [X] T016 Pasar las compuertas en vamosjuntos-backend: `gofmt -l .`, `go vet ./...`, `golangci-lint run` y `go test ./...`

### Validación del riesgo R4

- [X] T017 (visto bueno) Validar que Hosting reenvía `Authorization` (research R4):
  1. publicar el backend (`main` → `production`);
  2. obtener un ID token de una cuenta de correo de prueba con la API REST de Identity Toolkit (`accounts:signUp` con la `apiKey` pública);
  3. llamar `https://vamosjuntos-club.web.app/api/me` con `Authorization: Bearer <token>`.
  - `200 {"status":"unlinked"}` confirma el reenvío.
  - Si responde `401`, aplicar el plan B de R4: `requireAccount` lee también la cookie `__session`, y el frontend la escribe en `onIdTokenChanged`. Documentarlo en research R4.
  - Al terminar, borrar la cuenta de prueba.

### Frontend

- [X] T018 [P] Crear vamosjuntos-frontend/src/auth/firebase.ts, el único archivo que importa el SDK, igual que el de kaizen:
  - `initializeApp` con los `VITE_FIREBASE_*`, fallando con un mensaje claro si falta alguno;
  - `watchAuth`, `signInWithGoogle` (popup), `signInWithEmail`, `signUpWithEmail`, `resetPassword`, `signOut` y `getIdToken(forceRefresh?)`.
- [X] T019 [P] Crear vamosjuntos-frontend/src/auth/AuthProvider.tsx y vamosjuntos-frontend/src/auth/useAuth.ts: contexto con `user` (`null` | `{uid, email}`), `loading` y las acciones de T018
- [X] T020 [P] Cambiar vamosjuntos-frontend/src/api/client.ts:
  - agregar `Authorization: Bearer <getIdToken()>` a cada request;
  - ante un `401`, reintentar una vez con `getIdToken(true)`; si vuelve a fallar, llamar `onUnauthenticated`;
  - un `403 not_linked` o `not_invited` llama `onForbidden`;
  - quitar `withDeviceId` y borrar vamosjuntos-frontend/src/api/deviceId.ts.
- [X] T021 [P] Actualizar vamosjuntos-frontend/src/api/types.ts con `Me` (`status` y `guest`/`event` opcionales) y quitar los tipos de `verify`, `reentry` y `session`
- [X] T022 [P] Actualizar vamosjuntos-frontend/src/messages.ts con los mensajes de contracts/api.md: `account_already_linked`, `guest_already_linked`, `not_invited`, la variante de `invalid_code` para "entrar con mi código" y la tabla de errores `auth/*` del SDK
- [X] T023 Reemplazar vamosjuntos-frontend/src/session/* por un `MeProvider` que, con un usuario de Firebase, llama `GET /api/me` y expone `status`, `guest`, `event` y `refresh()`; sin usuario, el estado es `signedOut` (depende de T019 y T020)

**Checkpoint**: el backend verifica tokens, `/api/me` responde según el vínculo, R4 está validado y
el frontend tiene SDK, cliente y estado listos.

---

## Phase 3: User Story 1 - Primer ingreso: iniciar sesión, vincularse y confirmar (Priority: P1) 🎯 MVP

**Goal**: una cuenta nueva inicia sesión, elige su nombre, escribe su código (vínculo) y responde.

**Independent Test**: escenarios 1, 3, 4, 5 y 6 del quickstart.

- [X] T024 [US1] Poner `GET /api/onboarding/guests` detrás de `requireAccount` en vamosjuntos-backend/handlers.go (exige sesión, no vínculo)
- [X] T025 [US1] Agregar `POST /api/onboarding/link {guestId, code}` en vamosjuntos-backend/handlers.go:
  - pasa por `checkCode` del límite de intentos, con el `uid`;
  - responde `400 invalid_code`, `404 guest_not_found`, `409` (`errAccountLinked` o `errGuestLinked`) o `429`;
  - si todo está bien, vincula y responde `Me` (contracts/api.md).
- [X] T026 [US1] Cambiar `POST /api/onboarding/respond {attending}` en vamosjuntos-backend/handlers.go: usa el invitado vinculado (`403 not_linked` sin vínculo), no pide código y conserva `409 already_confirmed`
- [X] T027 [US1] Pruebas en vamosjuntos-backend/handlers_test.go:
  - lista con y sin token;
  - vincular con código correcto (espacios y mayúsculas), incorrecto, `guest_not_found`, los dos `409` y el bloqueo tras 5 fallos por `uid`;
  - `respond` confirmar y declinar sin código.
- [X] T028 [P] [US1] Crear vamosjuntos-frontend/src/pages/SignInPage.tsx: botón "Entrar con Google", formulario de correo con las pestañas "Entrar" y "Crear cuenta", enlace "¿Olvidaste tu contraseña?" (envía el correo de restablecimiento) y mensajes de messages.ts; botones de 44 px o más
- [X] T029 [US1] Ajustar vamosjuntos-frontend/src/pages/GuestListPage.tsx (exige sesión, enlace a "Ya me confirmaron, entrar con mi código") y vamosjuntos-frontend/src/pages/CodePage.tsx, que ahora llama `POST /api/onboarding/link` con `guestId` y luego `refresh()` de `MeProvider`
- [X] T030 [US1] Ajustar vamosjuntos-frontend/src/pages/ConfirmPage.tsx: toma el evento de `Me`, llama `respond` sin código y muestra la despedida si declina
- [X] T031 [US1] Reescribir las rutas en vamosjuntos-frontend/src/App.tsx según `status`:
  - `signedOut` → `SignInPage`;
  - `unlinked` → lista o código;
  - `pending` o `declined` → `ConfirmPage`;
  - `confirmed` → `/inicio`.
  - Agregar `AuthProvider` y `MeProvider` en `App`, y quitar `OnboardingProvider` si ya no se usa.

**Checkpoint**: el primer ingreso funciona de punta a punta en local.

---

## Phase 4: User Story 2 - Volver a entrar solo con la cuenta (Priority: P1)

**Goal**: una cuenta vinculada y confirmada entra directo, sin código, y la sesión persiste.

**Independent Test**: escenarios 7 y 8 del quickstart.

- [X] T032 [US2] Verificar en vamosjuntos-frontend/src/App.tsx que, al cargar con un usuario de Firebase restaurado y `status = confirmed`, la app va directo a `/inicio` sin pasar por la lista, y que mientras `AuthProvider` o `MeProvider` cargan se ve `Loading` y no la pantalla de inicio de sesión
- [X] T033 [US2] Mostrar en vamosjuntos-frontend/src/pages/HomePage.tsx el saludo con el nombre del invitado vinculado (`Me.guest.name`)

**Checkpoint**: reingreso sin código en otro navegador.

---

## Phase 5: User Story 3 - Acceso protegido (Priority: P1)

**Goal**: nada interno funciona sin cuenta vinculada y confirmada.

**Independent Test**: escenarios 1, 2 y 16 del quickstart.

- [X] T034 [US3] Poner `requireAccount` + `requireGuest` en todas las rutas 🔒C de vamosjuntos-backend/handlers.go y probar en vamosjuntos-backend/handlers_test.go los casos 401, `403 not_linked` y `403 not_invited`
- [X] T035 [US3] Reemplazar `RequireSession` por `RequireGuest` en vamosjuntos-frontend/src/session/: sin usuario → inicio de sesión; sin vínculo → onboarding. En `App`, conectar `onUnauthenticated` (`signOut` + inicio de sesión) y `onForbidden` (`refresh` de `MeProvider`; `not_invited` muestra su mensaje)

**Checkpoint**: las rutas internas y la API rechazan a quien no corresponde.

---

## Phase 6: User Story 4 - Vincularse cuando ya no aparezco en la lista (Priority: P2)

**Goal**: quien otro confirmó se vincula con solo su código.

**Independent Test**: escenario 12 del quickstart.

- [X] T036 [US4] Permitir `POST /api/onboarding/link {code}` sin `guestId` en vamosjuntos-backend/handlers.go: busca el invitado con `catalog.ByCode`, con el mismo límite de intentos y los mismos `409`. Probarlo en vamosjuntos-backend/handlers_test.go para un invitado confirmado, uno pendiente y uno declinado
- [X] T037 [US4] Reemplazar vamosjuntos-frontend/src/pages/ReentryPage.tsx por vamosjuntos-frontend/src/pages/LinkByCodePage.tsx ("Ya me confirmaron, entrar con mi código"), que llama `link` sin `guestId` y luego `refresh()`; actualizar su ruta en vamosjuntos-frontend/src/App.tsx

---

## Phase 7: User Story 5 - Cambiar de opinión (Priority: P2)

**Goal**: un "No asistirá" vinculado puede confirmar al volver.

**Independent Test**: escenario 13 del quickstart.

- [X] T038 [US5] Probar en vamosjuntos-backend/handlers_test.go que `GET /api/me` devuelve `declined` con el evento y que `respond {attending:true}` lo pasa a `confirmed`; en vamosjuntos-frontend/src/pages/ConfirmPage.tsx, mostrar "¿Cambiaste de opinión?" cuando `status = declined`

---

## Phase 8: User Story 6 - Confirmar a otro invitado (Priority: P2)

**Goal**: un confirmado confirma a otro con su código, sin vínculos.

**Independent Test**: escenario 11 del quickstart.

- [X] T039 [US6] Ajustar `POST /api/guests/{guestId}/confirmation` en vamosjuntos-backend/handlers.go para que use el `Guest` de `requireGuest` y el `uid` en `checkCode`, sin `X-Device-Id`, y adaptar `TestConfirmOther` en vamosjuntos-backend/handlers_test.go (y comprobar que no se crea ningún vínculo)
- [X] T040 [US6] Quitar `withDeviceId` de vamosjuntos-frontend/src/pages/ConfirmOtherPage.tsx y comprobar que sigue funcionando con el nuevo cliente

---

## Phase 9: User Story 7 - Cerrar sesión (Priority: P3)

**Goal**: cerrar sesión desde la app.

**Independent Test**: escenario 17 del quickstart.

- [X] T041 [US7] En vamosjuntos-frontend/src/pages/HomePage.tsx, "Cerrar sesión" llama `signOut()` del SDK, limpia `MeProvider` y lleva al inicio de sesión; quitar cualquier llamada a `DELETE /api/session`

---

## Phase 10: Polish & Cross-Cutting Concerns

- [X] T042 [P] Pasar `VITE_FIREBASE_*` desde `vars.*` al paso "Typecheck and build" de vamosjuntos-frontend/.github/workflows/front-deploy.yml, como en kaizen
- [X] T043 [P] Actualizar vamosjuntos-backend/README.md (sin `SESSION_KEY`, con `GOOGLE_CLOUD_PROJECT` obligatoria, autenticación por ID token y colección `links`) y vamosjuntos-frontend/README.md (configuración de Firebase, `.env.local` y variables de GitHub)
- [X] T044 [P] Actualizar los documentos de la 002 (plan, research R6, data-model, contracts, quickstart y tareas): el backend ya no tiene secrets y el frontend recibe `vars.VITE_FIREBASE_*`
- [X] T045 Pasar las compuertas: backend (`gofmt`, `vet`, `golangci-lint`, `test`) y frontend (`npm run build`, `npm run lint`)
- [X] T046 Probar en local los escenarios 1 a 19 del quickstart: backend local contra el Firestore y el Firebase Auth reales, y Vite con `.env.local`. Cuentas de correo de prueba y la cuenta Google del responsable
- [X] T047 (visto bueno) Publicar backend y frontend (`main` → `production`, uno después del otro) y repetir en producción los escenarios 1, 3, 5, 6, 7 y 20 (este cuando el dominio esté conectado)
- [X] T048 Limpieza del quickstart:
  - borrar las cuentas de prueba en Authentication y sus documentos en `links`, `attendance` y `loginAttempts`;
  - restaurar cualquier invitado quitado.
- [ ] T049 (responsable) Borrar el secret `SESSION_KEY` de `snsilvam/vamosjuntos-backend`
- [X] T050 Quitar el Sync Impact Report de la constitución (v3.0.0), hacer commit de specs/008 y de los cambios en 001 y 002, y registrar en este tasks.md las notas de implementación y las desviaciones (en especial el resultado de T017)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Fase 1)**: T002 (responsable) bloquea T004 (`.env.local`), T017 y todas las pruebas con
  Firebase real.
- **Foundational (Fase 2)**: bloquea todas las historias. T017 (R4) va antes de construir pantallas:
  si aplica el plan B, cambian T007 y T020.
- **US1 (Fase 3)**: depende de la Fase 2. Es el MVP.
- **US2 y US3 (Fases 4 y 5)**: dependen de US1 (rutas y `MeProvider`).
- **US4 a US7 (Fases 6 a 9)**: dependen de US1; entre ellas son independientes.
- **Polish (Fase 10)**: al final. T047 publica, así que el contrato nuevo exige backend y frontend
  juntos.

### Parallel Opportunities

- T003 y T004.
- T018, T019, T020, T021 y T022 (archivos distintos del frontend) mientras se hace el backend de la
  Fase 2.
- T028 (SignInPage) en paralelo con T024 a T027 (backend de US1).
- T042, T043 y T044.

## Parallel Example: Foundational frontend

```bash
Task: "Crear src/auth/firebase.ts (T018)"
Task: "Crear AuthProvider y useAuth (T019)"
Task: "Bearer + reintento en src/api/client.ts (T020)"
Task: "Mensajes nuevos en src/messages.ts (T022)"
```

---

## Implementation Strategy

### MVP First (US1 + US2 + US3)

1. Fases 1 y 2, incluido T017: el riesgo del encabezado queda resuelto antes de construir UI.
2. US1: primer ingreso con vínculo.
3. US2 y US3: reingreso sin código y rutas protegidas.
4. **STOP**: con esto la familia ya entra con Google o correo.

### Incremental Delivery

US4 (vincular con código) → US5 (cambiar de opinión) → US6 (confirmar a otro) → US7 (cerrar
sesión) → Polish y publicación (T047).

---

## Notes

- T017 publica solo el backend con el contrato nuevo, así que desde ese momento y hasta T047 la app
  publicada no funciona: el frontend en producción todavía usa el login por código de la 001. Es
  aceptable porque aún no se han repartido los códigos. Si ya se repartieron, hay que hacer T017
  con `--no-traffic` y probar contra la URL de esa revisión.
- Si T017 obliga al plan B, se actualizan research R4, contracts/api.md (convenciones) y T007/T020
  antes de seguir.
- Ningún token ni código se escribe en logs, ni en el backend ni en la consola del navegador.

## Implementation Notes (2026-10-30)

**Hecho, sin publicar** (commits locales en las ramas `008-autenticacion-firebase`):

- **Backend**:
  - archivos nuevos `auth.go` (Admin SDK, `requireAccount`, `requireGuest`) y `links.go`
    (`links/{guestId}` con `Create` en transacción);
  - borrados `session.go` y `session_test.go`;
  - `GOOGLE_CLOUD_PROJECT` obligatoria y `SESSION_KEY` fuera;
  - workflow sin secrets.
- **Frontend**:
  - `src/auth/` (SDK en un solo archivo, como kaizen) y `MeProvider`;
  - pantallas nuevas `SignInPage`, `LinkByCodePage` y `ErrorPage`;
  - cliente con `Authorization: Bearer` y un reintento ante `401`;
  - borrados `deviceId`, `SessionProvider`, `OnboardingProvider` y `ReentryPage`;
  - `.env.example`.
- **Validación**: `go vet`, `golangci-lint` (0 issues) y `go test` en verde, con pruebas nuevas
  de `me`, vínculo por lista y por código, conflictos `409`, bloqueo por cuenta, `respond` sin
  código, rutas protegidas, cambio de opinión y confirmar a otro. `npm run build` y `npm run lint`
  limpios. 0 usos de `secrets.*` en los workflows.

**Desviaciones del plan**:

- **T035, sin `RequireGuest`**: la protección vive en `AppRoutes`, que solo monta las rutas que
  permite el estado de `/api/me` (`signedOut`, `unlinked`, `pending`/`declined`, `confirmed`,
  `error`). Es más simple y cubre lo mismo; el backend valida cada request.
- **Sin `links_test.go`**: el vínculo no tiene lógica pura aparte de la transacción. Sus reglas
  se prueban con el fake de `linkStore` en `handlers_test.go`.
- **El aviso de `403` del cliente incluye la ruta**: `App` no recarga `/api/me` por un `403` del
  propio `/api/me`, porque eso provocaría un ciclo infinito.
- **Paso nuevo en el workflow del frontend**: "Validate Firebase config" falla si falta alguna
  `vars.VITE_FIREBASE_*`. Sin él, el build pasaría y el sitio publicado se rompería al abrir.
- **Versión del SDK**: `npm` instaló `firebase@12.19.0` (dentro de `^12.17`).
- **`gofmt` en Windows**: con `core.autocrlf=true`, los archivos vuelven con CRLF al cambiar de
  rama y `gofmt -l` los marca. Se verificó el formato ignorando los CR; en CI (LF) no aplica.

**Pendiente**:

- **Responsable**:
  - T002: app web, proveedores, dominio autorizado y variables `VITE_FIREBASE_*`;
  - T049: borrar `SESSION_KEY`.
- **Con los valores de T002**: T004 (crear `.env.local`) y T046 (pruebas locales).
- **Con visto bueno**:
  - T017: publicar el backend y validar el encabezado `Authorization`;
  - T047: publicar ambos;
  - T048: limpieza.
- **Al final**: T050, con las notas del resultado de T017.

### Validación local T046 (2026-10-30)

- **App web**: registrada por API, `vamosjuntos-web` con `appId 1:54833988307:web:8b503bee87443d81ca0a26`.
  La configuración pública quedó en `.env.local`, ignorado por git (T004).
- **API**: backend local contra Firebase Auth y Firestore reales, 17/17 PASS. Se usaron 4 cuentas
  de correo de prueba y los códigos reales, sin imprimirlos. Cubre los escenarios 2, 3, 4, 5, 6,
  7, 9, 10, 11, 12, 13 y 14, además del `403 not_linked`.
- **UI**: Chrome a 360 px contra Vite local, 13/13 PASS: inicio de sesión, crear cuenta, lista,
  código equivocado, vincular, confirmar, recargar, cerrar sesión y volver a entrar sin código.
  Áreas táctiles de 44 px o más, sin scroll horizontal y sin excepciones de JavaScript.
- **Hallazgo corregido**: el enlace "Usar otra cuenta" medía 20 px de alto; ahora es un botón de
  44 px.
- **No automatizados**: 15 y 16 (cubiertos por pruebas de Go), 18 (correo de restablecimiento
  real), 19 (ventana de Google) y 20 (dominio, en producción).
- **Limpieza**:
  - 6 cuentas de prueba borradas, junto con sus vínculos, respuestas e intentos;
  - en `attendance` solo queda la confirmación real de Sergio, hecha con la app publicada
    (versión 001);
  - `links` está vacía.
- **Incidente**: un backend viejo (001) seguía escuchando en `:8091` y la primera corrida le
  pegó a él. No escribió nada, porque rechaza esas llamadas antes de tocar Firestore. Se detuvo
  el proceso y se repitió.

### Publicación T017 y T047 (2026-10-30)

- **Backend** `dd7e102` → `production`: ejecución #2 en verde; revisión
  `vamosjuntos-backend-00002-n48`, con variables `GOOGLE_CLOUD_PROJECT`, `EVENT_ID` y
  `FIRESTORE_DATABASE` y ya sin `SESSION_KEY`.
- **T017 (research R4)**: `GET https://vamosjuntos-club.web.app/api/me` responde `401` sin token
  y `200 {"status":"unlinked"}` con `Authorization: Bearer <token>`. **Firebase Hosting sí
  reenvía `Authorization`**, así que el plan B (cookie `__session`) no hizo falta.
- **Frontend** `e675aa9` → `production`: ejecución #2 en verde; "Validate Firebase config"
  confirmó las 4 variables de GitHub.
- **Producción en `https://vamosjuntos.club`**: el recorrido de Chrome a 360 px dio 13/13 PASS,
  con una cuenta de correo de prueba vinculada a Melissa.
- **Limpieza (T048)**:
  - cuenta y vínculo de prueba borrados, y Melissa vuelve a estar pendiente;
  - en producción quedan 0 cuentas en Authentication, `links` vacía y solo la confirmación real
    de Sergio en `attendance`.

### Hotfix "Entrar con Google" (2026-10-30)

- **Síntoma**: en producción, "Entrar con Google" mostraba "Algo salió mal. Intenta de nuevo en un
  momento."
- **Causa**: la variable de GitHub `VITE_FIREBASE_AUTH_DOMAIN` tenía un espacio al inicio. El SDK
  armaba `https://%20vamosjuntos-club.firebaseapp.com` y fallaba antes de abrir el popup, con un
  error sin código de Firebase.
- **Arreglo** (frontend, en `production` y `main`):
  - `0e9beff`: `src/auth/firebase.ts` quita los espacios de las cuatro `VITE_FIREBASE_*`;
  - `18149fb`: `friendlyAuthMessage` traduce `auth/unauthorized-domain`,
    `auth/operation-not-allowed`, `auth/invalid-api-key` y `auth/invalid-app-credential`.
- **Incidente**: hubo un deploy de Hosting desde local, por fuera del pipeline, que luego reemplazó
  el deploy por pipeline de `18149fb` (ver notas de la 002).
- **Sigue pendiente**: T049.
