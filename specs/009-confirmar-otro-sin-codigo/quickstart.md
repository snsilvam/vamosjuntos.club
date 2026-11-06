# Quickstart: validar "Confirmar a otro invitado sin su código"

**Feature**: `009-confirmar-otro-sin-codigo` | **Date**: 2026-11-06

Contrato en [contracts/api.md](contracts/api.md) y modelo en [data-model.md](data-model.md).

## Preparación

- Publicar primero el backend y luego el frontend (research R4).
- Cuentas de correo de prueba:
  - **A**: vinculada a un invitado pendiente y confirmada; es quien confirma;
  - **B** y **C**: cuentas nuevas.
- Anotar los `guestId` que se toquen para borrar al final solo lo creado.

## Escenarios

| # | Pasos | Resultado esperado | Cubre |
|---|-------|--------------------|-------|
| 1 | Con A: Inicio → "Confirmar a otro invitado" | Lista alfabética de pendientes sin cuenta; ningún campo de código | US1-1, FR-002, SC-004 |
| 2 | Tocar un nombre X y luego "Cancelar" | Aparece "¿Confirmar a X?" con la advertencia; al cancelar vuelve la lista y `attendance/X` no existe | US1-2, US1-4, FR-003 |
| 3 | Tocar X y "Sí, confirmar a X" | "¡X quedó confirmado!"; X sale de la lista; `attendance/X` = `confirmed` con `respondedBy` = invitado de A; no hay `links/X`; la card suma 1 confirmado; A sigue en su sesión | US1-3, FR-005, SC-001 |
| 4 | Con B: "Ya me confirmaron, entrar con mi código" con el código de X | B se vincula a X y entra directo | US1-5, FR-009 |
| 5 | Con C: elegir un pendiente Y y vincularlo sin responder. Con A: recargar la lista | Y no aparece | US2-1 |
| 6 | Con A, abrir la lista antes del paso 5 y, después de que C vincule a Y, confirmar a Y | "Y ya entró con su propia cuenta y responde desde ahí."; la lista se actualiza; `attendance/Y` no existe | US2-2, FR-004 |
| 7 | `curl -X POST …/api/guests/X/confirmation` con el token de A | `409 already_responded` | US2-3 |
| 8 | Dos `curl` simultáneos de dos cuentas confirmadas sobre el mismo pendiente Z | Un `200` y un `409 already_responded`; un solo documento en `attendance/Z` | Edge Cases, SC-002 |
| 9 | `curl -X POST …/api/guests/W/confirmation -d '{"code":"cualquiera"}'` con el token de A (app vieja) | `200`; `loginAttempts/{uid de A}` no cambia | R4, FR-007 |
| 10 | `GET /api/guests/confirmable` sin token y con el token de C antes de vincular | `401 unauthenticated` y `403 not_linked` | 008 FR-010 |

**Limpieza al terminar** (solo por id):
- borrar en *Authentication → Usuarios* las cuentas A, B y C;
- borrar sus `links`;
- borrar `attendance/X`, `attendance/Z` y `attendance/W`, y la respuesta del invitado de A;
- borrar los `loginAttempts` de esas cuentas, si existen.

## Chequeos automáticos

```bash
cd vamosjuntos-backend && gofmt -l . && go vet ./... && golangci-lint run && go test ./...
cd vamosjuntos-frontend && npm run build && npm run lint
```
