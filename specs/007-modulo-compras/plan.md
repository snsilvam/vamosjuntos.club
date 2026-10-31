# Implementation Plan: Módulo Compras

**Branch**: `007-modulo-compras` | **Date**: 2026-10-30 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/007-modulo-compras/spec.md`

## Summary

Los confirmados ven qué hay que llevar al paseo, separado en "Comida" y "Suministros", con cantidad,
nota, responsable y estado, y un resumen "X por comprar · Y comprados". Cualquiera agrega, edita o
elimina ítems. Cada uno puede encargarse de un ítem sin responsable ("Yo lo llevo"), y solo el
responsable lo marca comprado, lo desmarca o lo suelta.

Enfoque técnico:

- **Datos**: un documento por ítem en `events/{EVENT_ID}/items/{itemId}`, con `responsibleId` y
  `bought` dentro.
- **Reglas**: una función pura `applyItem(item, cmd, me)` decide cada cambio. El almacén la corre
  en una transacción sobre el documento del ítem, así dos "Yo lo llevo" simultáneos nunca dejan dos
  responsables (FR-008). Los ítems no dependen entre sí, así que no hace falta leer la colección
  completa como en la 005.
- **API**: `/api/purchases/**` detrás de `requireGuest` (cuenta vinculada y confirmada, spec 008).
- **Frontend**: una página `/compras` con un formulario para crear y editar, y un enlace en la
  pantalla principal.

Detalle en [research.md](research.md).

## Technical Context

**Language/Version**: Go 1.26 y TypeScript + React 19, sin cambios.

**Primary Dependencies**: sin dependencias nuevas.

**Storage**: Firestore `vamosjuntos-club-db`, nueva colección `events/{EVENT_ID}/items`.

**Testing**:
- **Backend**: `go test`, con la regla pura (tabla de casos) y handlers con un fake del almacén.
- **Frontend**: el build y el lint.
- **De punta a punta**: con el [quickstart](quickstart.md).

**Target Platform**: Cloud Run y Firebase Hosting, con el celular como cliente principal.

**Project Type**: aplicación web (SPA + API REST).

**Performance Goals**: agregar un ítem en menos de 20 s (SC-001).

**Constraints**:
- Como máximo un responsable por ítem, incluso con acciones simultáneas (FR-008, SC-003).
- Solo el responsable marca, desmarca o suelta (FR-005).
- Mensajes en español (FR-010).

**Scale/Scope**: 25 confirmados como máximo, unas decenas de ítems, 7 endpoints y 1 página.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Se evalúa contra la constitución **v3.0.0**.

| Principio | Verificación | Estado |
|-----------|--------------|--------|
| I. Alcance cerrado | Es el módulo "Compras: comida y suministros" de "Dominio de la V1". Sin precios, costos ni división de gastos (spec, Assumptions). | ✅ |
| II. Acceso por invitación | Todas las rutas usan `requireGuest`. El responsable es siempre quien hace la petición. | ✅ |
| III. Simplicidad | Archivos `purchases.go` y `purchases_handlers.go` en el paquete `main` plano; un documento por ítem; transacción de un solo documento. | ✅ |
| IV. Código idiomático | Regla pura `applyItem`; `PurchaseStore` con `NewPurchaseStore`; interfaz pequeña definida en los handlers; errores de dominio con `errors.Is`. | ✅ |
| V. Errores y HTTP | `400` datos inválidos, `404` ítem inexistente, `409` ítem ya tomado o acción de alguien que no es el responsable; la UI los traduce. | ✅ |
| VI. Cero código sombra | Solo FR-001 a FR-010. Sin "asignar a otro", sin precios y sin historial (aclaraciones del 2026-10-30). | ✅ |

**Re-check post-diseño**: ✅

## Project Structure

### Documentation (this feature)

```text
specs/007-modulo-compras/
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
├── purchases.go                 # NUEVO: Item, applyItem (puro), orden, PurchaseStore
├── purchases_test.go            # NUEVO: tabla de la regla pura y del orden
├── purchases_handlers.go        # NUEVO: handlers /api/purchases/**
├── purchases_handlers_test.go   # NUEVO: handlers con fakePurchases
├── handlers.go                  # campo purchases en server, NewServer y 7 rutas al final de routes()
├── errors.go                    # códigos nuevos
└── main.go                      # NewPurchaseStore

vamosjuntos-frontend/
├── src/pages/PurchasesPage.tsx  # NUEVO: lista por categoría, resumen, acciones
├── src/pages/ItemForm.tsx       # NUEVO: crear/editar ítem
├── src/api/types.ts             # tipos de compras
├── src/messages.ts              # mensajes nuevos
├── src/App.tsx                  # ruta /compras (bloque confirmed)
└── src/pages/HomePage.tsx       # enlace "Compras"
```

**Structure Decision**: igual que la 005 y la 006: archivos propios y cambios mínimos en los
compartidos.

## Complexity Tracking

No hay violaciones de la constitución que justificar.
