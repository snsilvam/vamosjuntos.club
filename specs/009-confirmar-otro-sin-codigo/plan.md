# Implementation Plan: Confirmar a otro invitado sin su código

**Branch**: `009-confirmar-otro-sin-codigo` | **Date**: 2026-11-06 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/009-confirmar-otro-sin-codigo/spec.md`

## Summary

Un invitado confirmado confirma a otro eligiendo su nombre, sin escribir su código. Solo puede
confirmar a pendientes sin cuenta vinculada, y la app le pregunta "¿Confirmar a {nombre}?" antes
de enviar.

Enfoque técnico:

- **Lista**: ruta nueva `GET /api/guests/confirmable` (🔒C) con los pendientes sin vínculo. La
  lista del onboarding no cambia (research R1).
- **Regla**: función pura `confirmOther(current, linked)`. `AttendanceStore.ConfirmOther` la corre
  en una transacción que lee `links/{guestId}` y `attendance/{guestId}`, así que un vínculo
  simultáneo nunca deja a alguien confirmado por otro (research R2 y R3).
- **API**: `POST /api/guests/{guestId}/confirmation` deja de leer el cuerpo y de contar intentos
  (research R4). "Ya tiene cuenta" es `409 guest_already_linked` (research R5).
- **Frontend**: `ConfirmOtherPage` sin campo de código, con la pregunta previa en la misma pantalla
  y recarga de la lista ante `404` o `409` (research R6).

## Technical Context

**Language/Version**: Go 1.26 y TypeScript + React 19, sin cambios.

**Primary Dependencies**: sin dependencias nuevas.

**Storage**: Firestore `vamosjuntos-club-db`, sin colecciones ni campos nuevos (data-model.md).

**Testing**:
- **Backend**: `go test`, con la regla pura (tabla de casos) y los handlers con fakes.
- **Frontend**: el build y el lint.
- **De punta a punta**: con el [quickstart](quickstart.md).

**Target Platform**: Cloud Run y Firebase Hosting, con el celular como cliente principal.

**Project Type**: aplicación web (SPA + API REST).

**Performance Goals**: confirmar a otro en menos de 20 s y 3 toques desde Inicio (SC-001).

**Constraints**:
- Validar "pendiente y sin vínculo" al escribir, incluso con acciones simultáneas (FR-004, SC-002).
- Los códigos no se piden ni se muestran (SC-004) y siguen sin salir del backend.
- La app anterior, que manda `{"code"}`, debe seguir funcionando entre los dos despliegues (R4).

**Scale/Scope**: 25 invitados como máximo, 1 ruta nueva, 1 ruta cambiada y 1 página ajustada.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Se evalúa contra la constitución **v3.0.0**.

| Principio | Verificación | Estado |
|-----------|--------------|--------|
| I. Alcance cerrado | Es un ajuste del onboarding que ya existía (001 y 008). No crea ni edita invitados ni eventos y no cambia el tope de 25. | ✅ |
| II. Acceso por invitación | Las dos rutas usan `requireGuest`. Confirmar no vincula ni da acceso: para entrar sigue haciendo falta el código propio (FR-009). Los códigos no salen del backend. | ✅ |
| III. Simplicidad | Sin colecciones nuevas; una transacción de dos lecturas y una escritura; cambios en archivos existentes del paquete `main`. | ✅ |
| IV. Código idiomático | Regla pura `confirmOther`; métodos nuevos en los almacenes existentes; las interfaces de `handlers.go` quedan en 3 métodos; errores de dominio con `errors.Is`. | ✅ |
| V. Errores y HTTP | `404` invitado inexistente, `409` ya respondió o ya tiene cuenta; la UI los traduce y recarga la lista. | ✅ |
| VI. Cero código sombra | Solo FR-001 a FR-009. Sin deshacer en la app, sin "No asistirá" en nombre de otro. Se quita lo que deja de usarse (campo de código, mensaje `invalid_code` de "otro"). | ✅ |

**Re-check post-diseño**: ✅

## Project Structure

### Documentation (this feature)

```text
specs/009-confirmar-otro-sin-codigo/
├── spec.md
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── api.md
├── checklists/
│   └── requirements.md
└── tasks.md             # /speckit-tasks
```

### Source Code

```text
vamosjuntos-backend/
├── links.go             # linksCollection (ruta compartida) y LinkStore.Linked
├── attendance.go        # confirmOther (pura) y AttendanceStore.ConfirmOther (transacción)
├── attendance_test.go   # tabla de confirmOther
├── handlers.go          # interfaces, GET /api/guests/confirmable, handleConfirmOther sin código
└── handlers_test.go     # fakes y pruebas de US1 y US2

vamosjuntos-frontend/
├── src/pages/ConfirmOtherPage.tsx  # lista nueva, pregunta previa, sin código
└── src/messages.ts                 # guest_already_linked en "otro"; sale invalid_code de "otro"
```

**Structure Decision**: cambios mínimos en archivos existentes; no hace falta ningún archivo nuevo
de código.

## Complexity Tracking

No hay violaciones de la constitución que justificar.
