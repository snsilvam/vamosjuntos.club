# Implementation Plan: Onboarding y Sesión

**Branch**: `001-onboarding-sesion` | **Date**: 2026-10-30 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-onboarding-sesion/spec.md`

## Summary

Puerta de entrada de la app. Un invitado elige su nombre de la lista de pendientes, valida su
código y responde si asiste. Si confirma, recibe una sesión; si no, queda como "No asistirá". Los
confirmados vuelven a entrar solo con su código, y cualquier confirmado puede confirmar a otro
invitado escribiendo el código de esa persona.

Enfoque técnico:

- **Backend** en Go con la biblioteca estándar (`net/http`). Lee el catálogo de la subcolección
  `events/{EVENT_ID}/users` de Firestore en cada request que lo usa y lo valida (también al
  arrancar). *Enmienda v2.0.0: antes venía de la variable de entorno `GUESTS_JSON` (R12).*
- **Firestore** guarda solo las respuestas de asistencia y los intentos fallidos. Los cambios de
  estado pasan por transacciones y una función pura de transición.
- **Sesión** sin estado: cookie `__session` firmada con HMAC, por la restricción de Firebase
  Hosting.
- **Frontend** en React + TypeScript + Tailwind, con React Router y una guarda de sesión que
  reutilizarán las specs 003 a 007.

Detalle de las decisiones en [research.md](research.md).

## Technical Context

**Language/Version**: Go 1.26 (backend), TypeScript 7 + React 19 (frontend)

**Primary Dependencies**:
- Backend: `cloud.google.com/go/firestore`; el resto es biblioteca estándar (`net/http`,
  `encoding/json`, `crypto/hmac`, `time/tzdata`).
- Frontend: Vite 8, Tailwind CSS v4 (`@tailwindcss/vite`), React Router 7 y oxlint como linter.

**Storage**: Firestore (modo nativo). `events/{EVENT_ID}` es de solo lectura;
`events/{EVENT_ID}/attendance/{guestId}` guarda las respuestas y `loginAttempts/{deviceId}` los
intentos. `events/{EVENT_ID}/users/{guestId}` guarda el catálogo (nombre y código), que el
organizador mantiene a mano; la app solo lo lee.

**Testing**: `go test` (biblioteca estándar `testing` + `httptest`) para la lógica pura y los
handlers. En el frontend, el build con chequeo de tipos; la prueba de punta a punta es manual con
[quickstart.md](quickstart.md).

**Target Platform**: backend en Cloud Run (Linux, contenedor); frontend en Firebase Hosting, con
el navegador del celular como cliente principal.

**Project Type**: aplicación web (SPA + API REST), en dos repositorios hermanos.

**Performance Goals**: completar el onboarding en menos de 1 minuto (SC-001) y reingresar en menos
de 30 s (SC-005). Basta con respuestas de API por debajo de 1 s con arranque en frío incluido.

**Constraints**:
- Única cookie posible: `__session`.
- Hora de Colombia en todo lo visible.
- Ningún código en el repo, en Firestore ni en los logs.
- Mensajes de error amigables.

**Scale/Scope**: hasta 25 invitados, 1 evento, 7 endpoints y unas 6 pantallas.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Verificación | Estado |
|-----------|--------------|--------|
| I. Alcance cerrado | Un solo evento (`EVENT_ID`), que la app solo lee; tope de 25 invitados validado al arrancar; sin CRUD de eventos. La siembra local se hace con `curl` al emulador, no con código de la app. | ✅ |
| II. Acceso por invitación | El backend valida sesión y estado confirmado en cada request protegido (FR-008); el onboarding sigue el orden nombre → código → confirmación; el catálogo y los códigos vienen de `events/{EVENT_ID}/users` (solo lectura, reglas de Firestore cerradas a clientes) y nunca se versionan. | ✅ |
| III. Simplicidad | Un solo paquete `main` plano en el backend; cliente oficial de Firestore sin capas de repositorio; sin framework HTTP; frontend sin librerías de estado. | ✅ |
| IV. Código idiomático | Go: structs con constructores `NewXxx`, interfaces pequeñas definidas en los handlers, `context.Context` en el I/O, errores envueltos con `%w`, sin estado global ni `panic`; reglas de transición, token y límite de intentos como funciones puras. React: solo componentes funcionales y hooks, Tailwind, `camelCase` / `PascalCase`. | ✅ |
| V. Errores y HTTP | Contrato con 400, 401, 403, 404, 409 y 429 y cuerpo `{"error":{"code"}}`; la UI traduce cada código a un mensaje en español ([contracts/api.md](contracts/api.md)). | ✅ |
| VI. Cero código sombra | Solo lo que piden los FR de la spec. Se descartaron sesiones en Firestore, Vitest y TanStack Query (R1, R9, R10). | ✅ |
| Stack | Cloud Run + Firebase Hosting con rewrite de `/api/**`; cookie `__session`; zona `America/Bogota`. | ✅ |

**Re-check post-diseño**: ✅ El diseño de la Fase 1 no agrega dependencias, colecciones ni
endpoints fuera de lo anterior.

## Project Structure

### Documentation (this feature)

```text
specs/001-onboarding-sesion/
├── plan.md              # Este archivo
├── research.md          # Fase 0
├── data-model.md        # Fase 1
├── quickstart.md        # Fase 1
├── contracts/
│   └── api.md           # Fase 1
└── tasks.md             # Fase 2 (/speckit-tasks)
```

### Source Code

Dos repositorios hermanos de `vamosjuntos.club`, según la constitución:

```text
vamosjuntos-backend/
├── go.mod
├── main.go              # Configuración, cliente de Firestore, rutas y arranque del servidor
├── config.go            # Lectura y validación de variables de entorno
├── catalog.go           # Validación y búsqueda del catálogo (puro) + GuestStore (lee events/{id}/users)
├── catalog_test.go
├── attendance.go        # nextStatus (puro) + almacén en Firestore con transacciones
├── attendance_test.go
├── attempts.go          # registerAttempt (puro) + almacén en Firestore
├── attempts_test.go
├── session.go           # Firma y verificación del token, cookie y middleware requireSession
├── session_test.go
├── event.go             # Lectura del evento y EventSummary
├── handlers.go          # Handlers de /api/onboarding/*, /api/session*, /api/guests/*
├── handlers_test.go     # httptest con fakes de las interfaces
└── errors.go            # writeError(w, status, code)

vamosjuntos-frontend/
├── index.html
├── vite.config.ts       # Plugin de Tailwind + proxy de /api a :8080
├── src/
│   ├── main.tsx
│   ├── App.tsx          # Rutas: /, /codigo, /confirmar, /ya-confirme, /inicio, /confirmar-otro
│   ├── index.css        # @import "tailwindcss"
│   ├── api/
│   │   ├── client.ts    # fetch + ApiError(code) + X-Device-Id
│   │   └── deviceId.ts
│   ├── messages.ts      # code → mensaje amigable
│   ├── session/
│   │   ├── SessionProvider.tsx  # Context + useSession
│   │   └── RequireSession.tsx   # Guarda: 401/403 → onboarding
│   └── pages/
│       ├── GuestListPage.tsx
│       ├── CodePage.tsx
│       ├── ConfirmPage.tsx
│       ├── ReentryPage.tsx
│       ├── ConfirmOtherPage.tsx
│       └── HomePage.tsx         # Esqueleto: lo llenan las specs 003 y 004; incluye "Cerrar sesión"
└── package.json
```

**Structure Decision**: el backend es un único paquete `main` plano. Con unos 10 archivos, partir
en paquetes no aporta nada (Principio III), y las specs siguientes agregan archivos al mismo
paquete. Si en algún momento pasa de unos 25 archivos, se reconsidera con una enmienda. El
frontend se organiza por función (`api`, `session`, `pages`), sin capas adicionales. Los
artefactos de despliegue (Dockerfile, workflows, `firebase.json`) pertenecen a la spec 002 y no se
crean aquí.

## Complexity Tracking

No hay violaciones de la constitución que justificar.
