# Implementation Plan: Módulo Transporte

**Branch**: `005-modulo-transporte` | **Date**: 2026-10-30 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/005-modulo-transporte/spec.md`

## Summary

Los confirmados ven los vehículos del paseo agrupados por lugar de salida (Bogotá y Tocancipá),
con conductor, cupos totales y libres, pasajeros, hora estimada y nota, y la lista de confirmados
sin transporte. Cualquiera de ellos puede registrar, editar o eliminar un vehículo, y reservar o
liberar cupos para sí mismo o para otro confirmado.

Enfoque técnico:

- **Datos**: un documento por vehículo en `events/{EVENT_ID}/vehicles/{vehicleId}`, con los
  pasajeros dentro como lista de `guestId`.
- **Escrituras**: toda escritura es una transacción que lee todos los vehículos (≤ 25). Una regla
  pura recibe la lista y la acción, y devuelve los vehículos a escribir o un error de dominio.
  Así ninguna persona queda en dos lugares (FR-005) y ningún vehículo supera sus cupos (FR-006),
  aunque haya escrituras simultáneas.
- **API**: `/api/transport/**` detrás de `requireGuest` (cuenta vinculada y confirmada, spec 008).
- **Frontend**: una página `/transporte`, con un formulario de vehículo y acciones de reserva.

Detalle en [research.md](research.md).

## Technical Context

**Language/Version**: Go 1.26 y TypeScript + React 19, sin cambios.

**Primary Dependencies**: sin dependencias nuevas.

**Storage**: Firestore `vamosjuntos-club-db`, nueva colección `events/{EVENT_ID}/vehicles`.

**Testing**:
- **Backend**: `go test`, con la regla pura de transporte (tabla de casos) y handlers con un fake
  del almacén.
- **Frontend**: el build.
- **De punta a punta**: con el [quickstart](quickstart.md).

**Target Platform**: Cloud Run y Firebase Hosting, con el celular como cliente principal.

**Project Type**: aplicación web (SPA + API REST).

**Performance Goals**: reservar en menos de 30 segundos desde que se abre Transporte (SC-001).

**Constraints**:
- Un lugar por persona (FR-005).
- Nunca más pasajeros que cupos, incluso con reservas simultáneas (SC-002).
- Mensajes en español (FR-012).

**Scale/Scope**: 25 confirmados como máximo, unos 10 vehículos, 6 endpoints y 1 página.

**Coordinación**: otras sesiones trabajan en la 003 y la 004 sobre los mismos repos. El código va
en worktrees aparte (`vamosjuntos-backend-005` y `vamosjuntos-frontend-005`, rama
`005-modulo-transporte`). Los cambios en archivos compartidos son mínimos: rutas al final de
`routes()`, una ruta en `App.tsx` y un enlace en `HomePage.tsx`.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Se evalúa contra la constitución **v3.0.0**.

| Principio | Verificación | Estado |
|-----------|--------------|--------|
| I. Alcance cerrado | Es el módulo "Transporte: vehículos y cupos" de "Dominio de la V1". Sin costos, mapas ni seguimiento (spec, Assumptions). | ✅ |
| II. Acceso por invitación | Todas las rutas usan `requireGuest`. Conductores y pasajeros deben ser confirmados del catálogo (FR-011). | ✅ |
| III. Simplicidad | Archivo `transport.go` en el paquete `main` plano. Un documento por vehículo con los pasajeros dentro, y transacciones del cliente oficial de Firestore. | ✅ |
| IV. Código idiomático | Regla pura `applyTransport(vehicles, cmd)`; `TransportStore` con `NewTransportStore`; una interfaz pequeña en los handlers; errores de dominio con `errors.Is`. | ✅ |
| V. Errores y HTTP | `400` datos inválidos, `404` vehículo o pasajero inexistente, `409` conflictos (lleno, cupos por debajo de los pasajeros, persona ya ubicada); la UI los traduce. | ✅ |
| VI. Cero código sombra | Solo FR-001 a FR-013. Sin ida y regreso separados, sin costos, sin notificaciones y sin historial. | ✅ |

**Re-check post-diseño**: ✅

## Project Structure

### Documentation (this feature)

```text
specs/005-modulo-transporte/
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
├── transport.go         # NUEVO: Vehicle, applyTransport (puro), TransportStore (transacciones)
├── transport_test.go    # NUEVO: tabla de la regla pura
├── handlers.go          # rutas /api/transport/** al final de routes() + handlers
├── handlers_test.go     # pruebas de los handlers de transporte con fakeTransport
├── errors.go            # códigos nuevos
└── main.go              # NewTransportStore en NewServer

vamosjuntos-frontend/
├── src/pages/TransportPage.tsx  # NUEVO: lista por salida, sin transporte, reservar/liberar
├── src/pages/VehicleForm.tsx    # NUEVO: crear/editar vehículo
├── src/api/types.ts             # tipos de transporte
├── src/messages.ts              # mensajes nuevos
├── src/App.tsx                  # ruta /transporte (bloque confirmed)
└── src/pages/HomePage.tsx       # enlace "Transporte"
```

**Structure Decision**: archivos nuevos propios y cambios mínimos en los compartidos, para que el
rebase con la 003 y la 004 sea simple.

## Complexity Tracking

No hay violaciones de la constitución que justificar.
