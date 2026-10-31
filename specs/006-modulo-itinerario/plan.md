# Implementation Plan: Módulo Itinerario

**Branch**: `006-modulo-itinerario` | **Date**: 2026-10-30 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/006-modulo-itinerario/spec.md`

## Summary

Los confirmados ven las actividades del paseo agrupadas por día (23 al 26 de diciembre) y en orden
de hora, y cualquiera de ellos puede agregar, editar o eliminar cualquier actividad.

Enfoque técnico:

- **Datos**: un documento por actividad en `events/{EVENT_ID}/activities/{activityId}`. El día y
  las horas se guardan como texto en hora de Colombia (`2026-12-24`, `10:00`), sin conversiones de
  zona horaria (FR-006).
- **Reglas**: una función pura valida la actividad contra los días del evento (FR-003 a FR-005).
  Las actividades no dependen entre sí, así que no hace falta una transacción sobre toda la
  colección como en la 005.
- **API**: `/api/itinerary/**` detrás de `requireGuest` (cuenta vinculada y confirmada, spec 008).
- **Frontend**: una página `/itinerario` con un formulario para crear y editar, y un enlace en la
  pantalla principal.

Detalle en [research.md](research.md).

## Technical Context

**Language/Version**: Go 1.26 y TypeScript + React 19, sin cambios.

**Primary Dependencies**: sin dependencias nuevas.

**Storage**: Firestore `vamosjuntos-club-db`, nueva colección `events/{EVENT_ID}/activities`.

**Testing**:
- **Backend**: `go test`, con la regla pura (tabla de casos) y handlers con un fake del almacén.
- **Frontend**: el build y el lint.
- **De punta a punta**: con el [quickstart](quickstart.md).

**Target Platform**: Cloud Run y Firebase Hosting, con el celular como cliente principal.

**Project Type**: aplicación web (SPA + API REST).

**Performance Goals**: encontrar la siguiente actividad en menos de 15 s (SC-001) y agregar una en
menos de 1 min (SC-003).

**Constraints**:
- Días solo del 23 al 26 de diciembre de 2026, tomados del evento (FR-004).
- Horas en hora de Colombia, también desde otras zonas horarias (FR-006).
- Mensajes en español (FR-009).

**Scale/Scope**: 25 confirmados como máximo, unas decenas de actividades, 4 endpoints y 1 página.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Se evalúa contra la constitución **v3.0.0**.

| Principio | Verificación | Estado |
|-----------|--------------|--------|
| I. Alcance cerrado | Es el módulo "Itinerario: actividades y horarios" de "Dominio de la V1". Sin recordatorios, inscripciones ni "en curso" (spec, Assumptions). | ✅ |
| II. Acceso por invitación | Todas las rutas usan `requireGuest`. | ✅ |
| III. Simplicidad | Archivos `itinerary.go` e `itinerary_handlers.go` en el paquete `main` plano; un documento por actividad, sin transacciones sobre la colección. | ✅ |
| IV. Código idiomático | Regla pura `validateActivity(in, days)`; `ItineraryStore` con `NewItineraryStore`; interfaz pequeña definida en los handlers; errores de dominio con `errors.Is`. | ✅ |
| V. Errores y HTTP | `400` datos inválidos (con códigos para el orden de horas y el día fuera del paseo), `404` actividad inexistente; la UI los traduce. | ✅ |
| VI. Cero código sombra | Solo FR-001 a FR-009. La casilla "Termina al día siguiente" sale de la aclaración del 2026-10-30. | ✅ |

**Re-check post-diseño**: ✅

## Project Structure

### Documentation (this feature)

```text
specs/006-modulo-itinerario/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── api.md
└── tasks.md             # /speckit-tasks
```

### Source Code

```text
vamosjuntos-backend/
├── itinerary.go                 # NUEVO: Activity, validateActivity (puro), eventDays, ItineraryStore
├── itinerary_test.go            # NUEVO: tabla de la regla pura y del orden
├── itinerary_handlers.go        # NUEVO: handlers /api/itinerary/**
├── itinerary_handlers_test.go   # NUEVO: handlers con fakeItinerary
├── handlers.go                  # campo itinerary en server, NewServer y 4 rutas al final de routes()
├── errors.go                    # códigos nuevos
└── main.go                      # NewItineraryStore

vamosjuntos-frontend/
├── src/pages/ItineraryPage.tsx  # NUEVO: lista por día, eliminar con confirmación
├── src/pages/ActivityForm.tsx   # NUEVO: crear/editar actividad
├── src/api/types.ts             # tipos del itinerario
├── src/messages.ts              # mensajes nuevos
├── src/App.tsx                  # ruta /itinerario (bloque confirmed)
└── src/pages/HomePage.tsx       # enlace "Itinerario"
```

**Structure Decision**: igual que la 005: archivos nuevos propios y cambios mínimos en los
compartidos (`handlers.go`, `App.tsx`, `HomePage.tsx`), para integrar sin conflictos con la 007.

## Complexity Tracking

No hay violaciones de la constitución que justificar.
