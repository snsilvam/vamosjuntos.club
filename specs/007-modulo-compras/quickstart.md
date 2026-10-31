# Quickstart: validar el Módulo Compras

**Feature**: `007-modulo-compras` | **Date**: 2026-10-30

Contrato en [contracts/api.md](contracts/api.md) y reglas en [data-model.md](data-model.md).

## Entorno

- **Local**: pruebas de Go para la API y Chrome a 360 px con `/api/*` simulado para la UI.
- **Producción**: cuentas de correo de prueba vinculadas a invitados **pendientes**, e ítems
  "[PRUEBA] …". La familia ya usa la app: borrar solo lo creado, por id.

## Escenarios

| # | Pasos | Resultado esperado | Cubre |
|---|-------|--------------------|-------|
| 1 | Abrir Compras sin ítems | "Aún no hay nada en la lista…" | US1-3 |
| 2 | A agrega "Arroz", Comida, "5 kg" | En Comida, "por comprar" y "Sin responsable"; resumen "1 por comprar · 0 comprados" | US2-1, US1-2 |
| 3 | Agregar con el nombre vacío | Mensaje amigable; no se guarda | US2-2 |
| 4 | Agregar "Carbón" en Suministros y "arroz" otra vez | Suministros aparece aparte; con "arroz" se ve "Ya hay un ítem llamado así" antes de guardar | US1-1, caso borde |
| 5 | B toca "Yo lo llevo" en Arroz | B es el responsable | US3-1 |
| 6 | A intenta tomar Arroz | "Este ítem ya lo lleva {B}." y no cambia | US3-2 |
| 7 | B marca Arroz comprado | "comprado"; resumen actualizado | US3-3, FR-006 |
| 8 | A intenta desmarcar o soltar Arroz | `409 not_responsible` | FR-005 |
| 9 | B suelta Arroz | "Sin responsable" y "por comprar" | US3-4 |
| 10 | A y B toman Carbón a la vez (dos `curl` en paralelo) | Uno `200` y otro `409 item_taken`; un solo responsable | FR-008, SC-003 |
| 11 | Editar la cantidad de Arroz a "10 kg" | Se ve "10 kg"; responsable y estado no cambian | US4-1 |
| 12 | Eliminar Carbón (con responsable) | La confirmación dice quién lo lleva; luego desaparece | US4-2, FR-009 |
| 13 | `GET /api/purchases` con una cuenta sin vincular | `403 not_linked` | FR-001 |
| 14 | A 360 px, nombres y notas largos | Se leen completos, sin scroll horizontal | — |

## Chequeos automáticos

```bash
cd vamosjuntos-backend && gofmt -l . && go vet ./... && golangci-lint run && go test ./...
cd vamosjuntos-frontend && npm run build && npm run lint
```
