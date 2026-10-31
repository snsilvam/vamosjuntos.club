# Implementation Plan: Banner de Cuenta Regresiva

**Branch**: `004-banner-cuenta-regresiva` | **Date**: 2026-10-30 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/004-banner-cuenta-regresiva/spec.md`

## Summary

La pantalla principal abre con un banner grande y animado. Saluda al invitado por su nombre y le
dice cuántos días faltan para el paseo, con un mensaje e intensidad que cambian por tramos hasta
el día del evento, durante el paseo y después.

Enfoque técnico:

- **Backend**: ruta nueva `GET /api/countdown` con `requireGuest`.
  - La función pura `countdownAt(event, now)` calcula el tramo, los días calendario en hora de
    Colombia y los segundos hasta el próximo cambio.
  - La hora es la del backend, no la del celular (FR-003).
- **Frontend**: componente `CountdownBanner`, primero en `HomePage`.
  - Toma el nombre de `/api/me` y el tramo de `/api/countdown`.
  - Vuelve a pedir el tramo cuando vence `secondsToNextChange` o la pestaña vuelve a estar visible
    (FR-009).
  - Animación con Tailwind bajo `motion-safe:` (FR-006).
- **Sin escrituras**: no se crean colecciones, campos ni endpoints de escritura.

Detalle de las decisiones en [research.md](research.md).

## Technical Context

**Language/Version**: Go 1.26 (backend) y TypeScript + React 19 (frontend), sin cambios.

**Primary Dependencies**: ninguna nueva. Backend con `cloud.google.com/go/firestore` y
`firebase.google.com/go/v4`; frontend con `react`, `react-router` y Tailwind CSS 4.

**Storage**: Firestore, solo lectura de `events/{EVENT_ID}` ([data-model.md](data-model.md)).

**Testing**:
- **Backend**: `go test` con una tabla para `countdownAt` (7 tramos, medianoches, inicio y fin
  exactos, otra zona horaria) y pruebas del handler con los fakes de `handlers_test.go`.
- **Frontend**: build con chequeo de tipos y `oxlint`.
- **De punta a punta**: manual, con [quickstart.md](quickstart.md).

**Target Platform**: Cloud Run (backend) y Firebase Hosting (frontend), con el celular como cliente
principal.

**Project Type**: aplicación web (SPA + API REST), en dos repositorios.

**Performance Goals**: el número visible en menos de 3 segundos desde que se abre la app (SC-001).
Una llamada por carga y una más por cada cambio de día.

**Constraints**:
- La hora de Colombia sale del backend, sin importar el reloj ni la zona del dispositivo (FR-003,
  SC-002).
- El número es el texto más grande de la pantalla (FR-005).
- Sin animaciones con "reducir movimiento" (FR-006).
- Contraste suficiente a pleno sol (FR-010).

**Scale/Scope**: 25 invitados como máximo, un endpoint y un componente.

**Dependencia de la 008**: la spec dice "sesión activa (spec 001)". Desde la constitución v3.0.0 la
sesión es la de la 008, que ya está en `main`.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Se evalúa contra la constitución **v3.0.0**.

| Principio | Verificación | Estado |
|-----------|--------------|--------|
| I. Alcance cerrado | El banner está en "Dominio de la V1". Las fechas se leen de `events/{EVENT_ID}`; no se escriben en el código ni se editan. | ✅ |
| II. Acceso por invitación | `GET /api/countdown` exige ID token y un invitado vinculado y confirmado (`requireGuest`). La respuesta no lleva datos de invitados ni códigos. | ✅ |
| III. Simplicidad | Una función pura, un handler en un archivo nuevo del paquete `main` y un componente. Sin librerías de fechas ni de animación. | ✅ |
| IV. Código idiomático | `countdownAt` es pura y se prueba sin Firestore. El reloj se inyecta como campo de `server` sin estado global (research R8). Solo componentes funcionales, Hooks y Tailwind. | ✅ |
| V. Errores y HTTP | `401`, `403`, `404 event_not_found` y `500 internal` según el contrato. El banner muestra solo textos amigables (research R7). | ✅ |
| VI. Cero código sombra | Solo lo de la spec: sin notificaciones ni recordatorios, sin modo para simular fechas en producción. La recarga al volver a la pestaña sirve a FR-009 (research R3). | ✅ |
| Stack | Sin cambios. | ✅ |
| Tiempo | Días calculados con la zona `colombia` (UTC-5) existente en `event.go`. | ✅ |
| Despliegue continuo | Sin cambios en los pipelines. Las compuertas existentes (`gofmt`, `go vet`, `golangci-lint`, `npm run build`) cubren el código nuevo. | ✅ |

**Re-check post-diseño**: ✅ El diseño de la Fase 1 agrega una sola ruta de lectura y no saca
datos fuera de la spec.

## Project Structure

### Documentation (this feature)

```text
specs/004-banner-cuenta-regresiva/
├── plan.md              # Este archivo
├── research.md          # Fase 0
├── data-model.md        # Fase 1
├── quickstart.md        # Fase 1
├── contracts/
│   └── api.md           # Fase 1: GET /api/countdown (se suma al contrato de la 008)
└── tasks.md             # Fase 2 (/speckit-tasks)
```

### Source Code

```text
vamosjuntos-backend/
├── countdown.go         # NUEVO: Countdown, tramos, countdownAt (pura) y handleCountdown
├── countdown_test.go    # NUEVO: tabla de countdownAt y pruebas del handler (200, 401, 403, 404)
└── handlers.go          # campo server.now (time.Now en NewServer) y ruta GET /api/countdown

vamosjuntos-frontend/
├── src/api/types.ts                    # tipo Countdown
├── src/components/CountdownBanner.tsx  # NUEVO: carga, temporizador, tramos y textos
├── src/components/Screen.tsx           # title opcional
├── src/index.css                       # @keyframes de la entrada del banner en @theme
└── src/pages/HomePage.tsx              # el banner reemplaza el recuadro punteado y el saludo del título
```

**Structure Decision**: todo sigue en el paquete `main` plano del backend, con la lógica en un
archivo nuevo para no crecer `handlers.go`. En el frontend el banner va en `src/components/`, junto
a `Screen`, `ui` y la `TripCard` de la 003, porque es una pieza de la página y no una ruta.

**Ramas y trabajo en paralelo** (003 y 005 avanzan al mismo tiempo):

- Rama `004-banner-cuenta-regresiva` desde `main` en ambos repos, en worktrees aparte
  (`../vamosjuntos-backend-004`, `../vamosjuntos-frontend-004`). Los working trees principales los
  usa la 003.
- Archivos compartidos con cambios mínimos:
  - `handlers.go`: una línea de ruta, un campo y su inicialización.
  - `HomePage.tsx`: el import, el título y el recuadro punteado, que la 003 dejó reservado para el
    banner.
  - `types.ts`: el tipo nuevo va junto a `EventSummary` y no al final, donde agrega la 003.
- Si la 003 llega antes a `main`, se hace rebase y se resuelven esas líneas. Integración y
  publicación a `production`, una feature a la vez y con el OK del responsable.
- El contrato solo agrega una ruta, así que lo natural es publicar primero el backend.

## Complexity Tracking

No hay violaciones que justificar.
