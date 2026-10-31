# Quickstart: validar el Módulo Transporte

**Feature**: `005-modulo-transporte` | **Date**: 2026-10-30

Contrato en [contracts/api.md](contracts/api.md) y reglas en [data-model.md](data-model.md).

## Entorno

Es el mismo de la spec 008, §3:
- backend local contra Firebase Auth y Firestore reales;
- Vite con `.env.local`;
- cuentas de correo de prueba vinculadas y confirmadas.

Al terminar, borra:
- los vehículos de `events/{EVENT_ID}/vehicles`;
- las cuentas, vínculos y respuestas de prueba.

## Escenarios

| # | Pasos | Resultado esperado | Cubre |
|---|-------|--------------------|-------|
| 1 | Abrir Transporte sin vehículos | Mensaje que invita a registrar el primero; aparecen todos los confirmados sin transporte | US1-3, FR-010 |
| 2 | Registrar "Carro de A" (A conduce, Bogotá, 3 cupos, 06:30) | Aparece en "Sale de Bogotá" con 3 de 3 libres | US2-1 |
| 3 | Registrar con 0 o 25 cupos | Mensaje amigable; no se guarda | US2-2 |
| 4 | B se reserva en "Carro de A" | B es pasajero; quedan 2 libres; B sale de "sin transporte" | US3-1 |
| 5 | Registrar "Van de C" (Tocancipá, 1 cupo) y B se reserva ahí | B cambia de vehículo; "Carro de A" vuelve a 3 libres | US3-2, FR-005 |
| 6 | D se reserva en la "Van de C" llena | "Este vehículo ya no tiene cupos." | US3-3 |
| 7 | A (conductor) intenta reservarse en la van | "Esa persona va manejando su propio vehículo." | Caso borde |
| 8 | A reserva para D en su carro (para otro) | D es pasajero de "Carro de A" | US3-5, FR-013 |
| 9 | Cualquiera libera a D | D vuelve a "sin transporte" | US3-4, US3-6 |
| 10 | Con 2 pasajeros, editar "Carro de A" a 1 cupo | Mensaje amigable; no cambia | US4-1, FR-007 |
| 11 | Eliminar la van, confirmando | B vuelve a "sin transporte" | US4-2, FR-008 |
| 12 | Dos reservas simultáneas por el último cupo (dos `curl` en paralelo) | Una `200` y otra `409 vehicle_full`; nunca más pasajeros que cupos | SC-002, FR-006 |
| 13 | Llamar `/api/transport` con una cuenta sin vincular | `403 not_linked` | FR-001 |

## Chequeos automáticos

```bash
cd vamosjuntos-backend && gofmt -l . && go vet ./... && golangci-lint run && go test ./...
cd vamosjuntos-frontend && npm run build && npm run lint
```
