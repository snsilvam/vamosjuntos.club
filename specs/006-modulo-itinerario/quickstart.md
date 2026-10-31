# Quickstart: validar el Módulo Itinerario

**Feature**: `006-modulo-itinerario` | **Date**: 2026-10-30

Contrato en [contracts/api.md](contracts/api.md) y reglas en [data-model.md](data-model.md).

## Entorno

- **Local**: backend contra Firebase Auth y Firestore reales y Vite con `.env.local`, igual que la
  008 (§3) y la 005.
- **Producción**: cuentas de correo de prueba vinculadas a invitados **pendientes**. La familia ya
  usa la app: no tocar actividades ni invitados reales.

Al terminar, borra solo lo que creó la prueba: actividades por id, cuentas, vínculos y respuestas.

## Escenarios

| # | Pasos | Resultado esperado | Cubre |
|---|-------|--------------------|-------|
| 1 | Abrir Itinerario sin actividades | "El itinerario aún no tiene actividades." | US1-3 |
| 2 | Agregar "Piscina", 24 de diciembre, 10:00 a 12:00 | Aparece bajo "Jueves 24 de diciembre" | US2-1 |
| 3 | Agregar "Desayuno", 24 de diciembre, 08:00 | Queda antes de "Piscina" | FR-002 |
| 4 | Agregar "Caminata", 24 de diciembre, 10:00 | Se ven las dos de las 10:00, "Caminata" antes de "Piscina" | US1-2, FR-002 |
| 5 | Agregar con fin 09:00 e inicio 10:00, sin "Termina al día siguiente" | "La hora de fin debe ser después de la hora de inicio." y no se guarda | US2-2, FR-005 |
| 6 | `POST` con `date` `2026-12-27` | `400 activity_out_of_range`; en la UI el día solo ofrece del 23 al 26 | US2-3, FR-004 |
| 7 | Agregar "Cena de Nochebuena", 24 de diciembre, 20:00 a 01:00 con "Termina al día siguiente" | En el 24, con "hasta la 1:00 a. m. del día siguiente" | Caso borde, FR-005 |
| 8 | `POST` con 22:00 a 23:00 y `endsNextDay` | `400 activity_time_order` (25 h) | FR-005 |
| 9 | Editar "Piscina" a las 07:00 | Pasa a ser la primera del 24 | US3-1 |
| 10 | Eliminar "Caminata" confirmando | Desaparece; cancelar no borra nada | US3-2, FR-008 |
| 11 | `PUT` o `DELETE` sobre un id inexistente | `404 activity_not_found` | Contrato |
| 12 | Título de 81 caracteres o descripción de 501 | `400 invalid_activity` | FR-003 |
| 13 | Con DevTools en zona horaria `Asia/Tokyo`, recargar | Mismos días y horas | FR-006 |
| 14 | `GET /api/itinerary` con una cuenta sin vincular | `403 not_linked` | FR-001 |
| 15 | A 360 px, un título y una descripción largos | Se leen completos, sin scroll horizontal | Caso borde |

## Chequeos automáticos

```bash
cd vamosjuntos-backend && gofmt -l . && go vet ./... && golangci-lint run && go test ./...
cd vamosjuntos-frontend && npm run build && npm run lint
```
