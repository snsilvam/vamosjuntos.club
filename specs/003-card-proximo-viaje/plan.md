# Implementation Plan: Card "Próximo viaje"

**Branch**: `003-card-proximo-viaje` | **Date**: 2026-10-30 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/003-card-proximo-viaje/spec.md`

## Summary

La pantalla principal muestra una card con los datos del paseo (nombre, salida, destino, inicio y
fin en hora de Colombia) y una línea de conteos: invitados, confirmados, no asistirán y pendientes.

Enfoque técnico:

- **Backend**: ruta nueva `GET /api/event` con `requireGuest`. Devuelve el `EventSummary` que ya
  existe y un `AttendanceSummary` calculado por una función pura, `summarizeAttendance`, que solo
  cuenta invitados del catálogo.
- **Frontend**: componente `TripCard` en `HomePage`. Pide `/api/event` al montarse y formatea las
  fechas con `formatEventDate`, ajustada para que coincida con el texto de la spec.
- **Sin escrituras**: no se crean colecciones, campos ni endpoints de escritura.

Detalle de las decisiones en [research.md](research.md).

## Technical Context

**Language/Version**: Go 1.26 (backend) y TypeScript + React 19 (frontend), sin cambios.

**Primary Dependencies**: ninguna nueva. Backend con `cloud.google.com/go/firestore` y
`firebase.google.com/go/v4`; frontend con `react`, `react-router` y Tailwind CSS.

**Storage**: Firestore `vamosjuntos-club-db`, solo lectura de `events/{EVENT_ID}`, `users` y
`attendance` ([data-model.md](data-model.md)).

**Testing**:
- **Backend**: `go test` con los fakes de `handlers_test.go`, más una tabla para
  `summarizeAttendance`.
- **Frontend**: build con chequeo de tipos.
- **De punta a punta**: manual, con [quickstart.md](quickstart.md), contra el Firebase real.

**Target Platform**: Cloud Run (backend) y Firebase Hosting (frontend), con el celular como cliente
principal.

**Project Type**: aplicación web (SPA + API REST), en dos repositorios.

**Performance Goals**: la card visible en menos de 10 segundos desde que se abre la pantalla
principal (SC-001). Una sola llamada nueva por carga.

**Constraints**:
- Fechas en hora de Colombia sin importar el dispositivo (FR-003, SC-003).
- Solo conteos, sin nombres (FR-007).
- Un mensaje amigable si el evento no está disponible (FR-006).

**Scale/Scope**: 25 invitados como máximo, un endpoint y un componente.

**Dependencia de la 008**: la spec dice "sesión activa (spec 001)". Desde la constitución v3.0.0 la
sesión es la de la 008, así que la card se valida en producción después de publicar la 008
(T047 de su `tasks.md`).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Se evalúa contra la constitución **v3.0.0**.

| Principio | Verificación | Estado |
|-----------|--------------|--------|
| I. Alcance cerrado | La card está en "Dominio de la V1". Solo lee el evento, el catálogo y la asistencia; no los edita. | ✅ |
| II. Acceso por invitación | `GET /api/event` exige ID token y un invitado vinculado y confirmado (`requireGuest`). Firestore sigue cerrado a clientes. La respuesta no lleva códigos. | ✅ |
| III. Simplicidad | Un handler y una función pura en el paquete `main` plano. Un componente en el frontend. Sin librerías nuevas. | ✅ |
| IV. Código idiomático | `summarizeAttendance` es pura y se prueba sin Firestore. El handler reutiliza `eventGetter`, `guestCatalog` y `attendanceRecorder`. Solo componentes funcionales, Hooks y Tailwind. | ✅ |
| V. Errores y HTTP | `401`, `403`, `404 event_not_found` y `500 internal` según el contrato. La card muestra solo el texto amigable de FR-006. | ✅ |
| VI. Cero código sombra | Solo lo de la spec: sin lista de nombres, sin tiempo real y sin botón de reintentar (research R4). | ✅ |
| Stack | Sin cambios. | ✅ |
| Despliegue continuo | Sin cambios en los pipelines. Las compuertas existentes (`gofmt`, `go vet`, `golangci-lint`, `npm run build`) cubren el código nuevo. | ✅ |

**Re-check post-diseño**: ✅ El diseño de la Fase 1 agrega una sola ruta de lectura y ningún dato
fuera de la spec.

## Project Structure

### Documentation (this feature)

```text
specs/003-card-proximo-viaje/
├── plan.md              # Este archivo
├── research.md          # Fase 0
├── data-model.md        # Fase 1
├── quickstart.md        # Fase 1
├── contracts/
│   └── api.md           # Fase 1: GET /api/event (se suma al contrato de la 008)
└── tasks.md             # Fase 2 (/speckit-tasks)
```

### Source Code

```text
vamosjuntos-backend/
├── attendance.go        # agrega AttendanceSummary y summarizeAttendance (pura)
├── attendance_test.go   # tabla de summarizeAttendance (incluye un invitado retirado del catálogo)
├── handlers.go          # ruta GET /api/event (requireGuest) y handleEvent
└── handlers_test.go     # 200 con conteos, 404 event_not_found, 403 sin confirmar, 401 sin token

vamosjuntos-frontend/
├── src/api/types.ts             # AttendanceSummary y EventResponse
├── src/format.ts                # formatEventDate sin coma tras el día de la semana; plural de los conteos
├── src/components/TripCard.tsx  # NUEVO: carga /api/event; estados loading, ready y unavailable
└── src/pages/HomePage.tsx       # muestra TripCard; el recuadro punteado queda solo para la cuenta regresiva (004)
```

**Structure Decision**: todo sigue en el paquete `main` plano del backend. En el frontend la card
va en `src/components/`, junto a `Screen` y `ui`, porque es una pieza de la página y no una ruta.

**Ramas**: `003-card-proximo-viaje` en ambos repos, desde `main`. El contrato solo agrega una ruta,
así que el orden de publicación da igual. Lo natural es publicar primero el backend.

## Complexity Tracking

No hay violaciones que justificar.
