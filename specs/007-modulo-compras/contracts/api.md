# API Contract: Módulo Compras

**Feature**: `007-modulo-compras` | **Base**: `/api`

Convenciones de la [spec 008](../../008-autenticacion-firebase/contracts/api.md):
- `Authorization: Bearer <ID token>` en cada llamada;
- errores con la forma `{"error":{"code"}}`;
- `Cache-Control: no-store`.

**Todas las rutas son 🔒C**: exigen una cuenta vinculada y confirmada. Si no la hay, responden
`401 unauthenticated`, `403 not_linked` o `403 not_invited`.

## Tipos

```jsonc
// Item
{
  "id": "aB3…",
  "name": "Arroz",
  "category": "comida" | "suministros",
  "quantity": "5 kg",      // opcional
  "note": "Integral",      // opcional
  "responsible": Guest,    // ausente si no tiene
  "bought": false
}

// ItemInput
{ "name": "Arroz", "category": "comida", "quantity": "5 kg", "note": "" }

// Purchases
{ "items": [Item], "summary": { "pending": 3, "bought": 1 } }
```

`items` va en el orden de [data-model.md](../data-model.md).

## Rutas

| Ruta | Request | Éxito | Errores |
|------|---------|-------|---------|
| `GET /api/purchases` | — | `200 Purchases` | — |
| `POST /api/purchases/items` | `ItemInput` | `201 Purchases` | `400 invalid_request`, `400 invalid_item` |
| `PUT /api/purchases/items/{id}` | `ItemInput` | `200 Purchases` | `400 invalid_request`, `400 invalid_item`, `404 item_not_found` |
| `DELETE /api/purchases/items/{id}` | — | `204` | `404 item_not_found` |
| `POST /api/purchases/items/{id}/take` | — | `200 Purchases` | `404 item_not_found`, `409 item_taken` |
| `POST /api/purchases/items/{id}/release` | — | `200 Purchases` | `404 item_not_found`, `409 not_responsible` |
| `POST /api/purchases/items/{id}/bought` | `{"bought": true}` | `200 Purchases` | `400 invalid_request`, `404 item_not_found`, `409 not_responsible` |

- **Editar** no cambia responsable ni estado (FR-007).
- **Tomar** asigna a quien hace la petición; si ya es suyo, responde `200` sin cambios.
- **Soltar** deja el ítem sin responsable y "por comprar".

## Mensajes en la UI (FR-010)

| Caso | Mensaje |
|------|---------|
| `invalid_item` | "Revisa los datos: el nombre es obligatorio y los textos no pueden ser tan largos." |
| `item_taken` | "Este ítem ya lo lleva {nombre}." (nombre tomado de la lista recargada) |
| `not_responsible` | "Solo quien lleva este ítem puede cambiarlo." |
| `item_not_found` | "Ese ítem ya no existe. Actualizamos la lista." |
| Nombre repetido (solo UI) | "Ya hay un ítem llamado así." + "Guardar de todas formas" |
| Lista vacía | "Aún no hay nada en la lista. Agrega lo primero que haga falta." |
| Error al cargar | "No pudimos cargar la lista de compras. Intenta de nuevo en un momento." |
