# Data Model: Módulo Compras

**Feature**: `007-modulo-compras` | **Date**: 2026-10-30

## Ítem: Firestore `events/{EVENT_ID}/items/{itemId}`

El id lo genera Firestore. Solo escribe el backend; las reglas de Firestore siguen cerradas a los
clientes.

| Campo | Tipo | Regla |
|-------|------|-------|
| `name` | string | De 1 a 60 caracteres tras recortar espacios |
| `category` | string | `comida` o `suministros` |
| `quantity` | string | Opcional; texto libre de hasta 30 caracteres ("5 kg") |
| `note` | string | Opcional; hasta 200 caracteres |
| `responsibleId` | string | Vacío o el `guestId` de quien se encargó (FR-004) |
| `bought` | bool | Solo puede ser `true` con `responsibleId` |
| `updatedAt` | timestamp | Hora del servidor de la última escritura |
| `updatedBy` | string | `guestId` de quien hizo la última escritura |

**Invariantes** (las garantiza `applyItem` dentro de una transacción sobre el documento):
- Como máximo un responsable (FR-004, FR-008).
- `bought = true` implica `responsibleId ≠ ""`.

## Estados de un ítem

```text
por comprar, sin responsable ──tomar (me)──► por comprar, de me ──marcar──► comprado, de me
          ▲                                       │    ▲                        │
          └────────────── soltar (me) ────────────┘    └──────desmarcar─────────┘
          ▲                                                                     │
          └─────────────────────────── soltar (me) ─────────────────────────────┘
```

Solo el responsable mueve un ítem que ya tiene dueño (FR-005). Tomar un ítem de otro devuelve
`errItemTaken`.

## Orden y resumen (calculados)

- **Orden**: `comida` antes de `suministros`, luego nombre sin acentos ni mayúsculas, luego `id`.
- **Resumen**: `pending` = ítems con `bought = false`; `bought` = ítems con `bought = true` (FR-006).

## Errores de dominio

| Error | Cuándo | HTTP |
|-------|--------|------|
| `errInvalidItem` | nombre vacío o largo, categoría desconocida, cantidad o nota largas | `400 invalid_item` |
| `errItemNotFound` | el id no existe | `404 item_not_found` |
| `errItemTaken` | tomar un ítem que ya es de otro | `409 item_taken` |
| `errNotResponsible` | soltar, marcar o desmarcar sin ser el responsable (incluido un ítem sin responsable) | `409 not_responsible` |
