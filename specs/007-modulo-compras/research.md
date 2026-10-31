# Research: Módulo Compras

**Feature**: `007-modulo-compras` | **Date**: 2026-10-30

No hay puntos `NEEDS CLARIFICATION`. La spec quedó aclarada el 2026-10-30 (sesión de la 008, "Yo
lo llevo" solo para uno mismo y soltar vuelve a "por comprar").

## R1. Responsable y estado dentro del documento del ítem

- **Decision**: `events/{EVENT_ID}/items/{itemId}` guarda `responsibleId` (vacío si no hay) y
  `bought`. No hay una colección aparte de asignaciones.
- **Rationale**: el ítem, su responsable y su estado se leen y escriben juntos. El resumen es
  contar `bought`.

## R2. Concurrencia: transacción de un documento con regla pura

- **Decision**: toda acción sobre un ítem existente (editar, eliminar, tomar, soltar, marcar)
  corre en una transacción.
  1. Lee el documento; si no existe, devuelve `errItemNotFound`.
  2. Llama a la regla pura `applyItem(item, cmd, me)`, que devuelve el ítem nuevo, si hay que
     borrarlo, o un error de dominio.
  3. Escribe.
- **Rationale**: si dos personas tocan "Yo lo llevo" a la vez, Firestore reintenta la segunda
  transacción con el dato nuevo, y la regla responde `errItemTaken` (FR-008, SC-003). Ningún
  ítem depende de otro, así que basta con leer uno.
- **Alternatives considered**: la transacción sobre toda la colección de la 005. No protege
  ninguna regla extra y aumenta los reintentos.

## R3. Reglas de la función pura

| Comando | Regla |
|---------|-------|
| editar | Valida nombre, categoría, cantidad y nota; no cambia responsable ni estado (FR-007) |
| eliminar | Siempre permitido; la confirmación (con quién lo lleva) es de la UI (FR-009) |
| tomar | Sin responsable → `me`. Si ya es de `me`, no cambia. Si es de otro → `errItemTaken` |
| soltar | Solo el responsable (`errNotResponsible`). Queda sin responsable y `bought = false` (aclaración) |
| marcar / desmarcar | Solo el responsable (`errNotResponsible`). Cambia `bought` |

Un ítem sin responsable no se puede marcar comprado: primero alguien se encarga (FR-005). Así
"comprado" siempre tiene un responsable.

## R4. Escrituras responden con la lista completa

- **Decision**: `POST` responde `201`; editar, tomar, soltar y marcar responden `200`, todos con el
  cuerpo de `GET /api/purchases`. `DELETE` responde `204`. Es la convención de la 005 y la 006.
- **Rationale**: la pantalla se actualiza sin otra llamada. Ante `409 item_taken`, la UI recarga y
  muestra "Este ítem ya lo lleva {nombre}." con el responsable de la respuesta nueva. Así el
  cuerpo de error sigue siendo solo `{"error":{"code"}}`.

## R5. Orden y resumen

- **Decision**:
  - el backend devuelve los ítems por categoría (`comida` y luego `suministros`), luego por nombre
    sin acentos (`foldAccents`) e `id`;
  - el resumen `{"pending", "bought"}` cuenta `bought` sobre todos los ítems (FR-006);
  - el responsable se resuelve con el catálogo; si ya no está, la UI dice "alguien que ya no está
    en la lista", como en Transporte.

## R6. Aviso de nombre repetido

- **Decision**: el frontend compara el nombre sin acentos ni mayúsculas con los ítems cargados.
  Si coincide, muestra "Ya hay un ítem llamado así" y pide un segundo toque en "Guardar de todas
  formas". El backend no lo impide (caso borde: "se permite").

## R7. Pantalla

- **Decision**: igual que Transporte e Itinerario:
  - página `/compras` dentro del bloque `confirmed` de `App.tsx`;
  - `ItemForm` para crear y editar;
  - acciones en cada tarjeta: "Yo lo llevo", "Ya lo compré" / "Aún no lo compro", "Soltar",
    "Editar" y "Eliminar" con confirmación en la misma tarjeta. Si tiene responsable, la
    confirmación dice "Lo lleva {nombre}".
