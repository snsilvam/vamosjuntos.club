# Tasks: Módulo Compras

**Input**: Design documents from `/specs/007-modulo-compras/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/api.md, quickstart.md

**Tests**: se incluyen pruebas de Go para la regla pura y los handlers, como en la 005 y la 006.
El Principio IV exige que la lógica de negocio sea testeable sin Firestore.

**Organization**: por historia de usuario. Código en ramas `007-modulo-compras` de
`vamosjuntos-backend` y `vamosjuntos-frontend`.

## Format: `[ID] [P?] [Story] Description`

---

## Phase 1: Setup

- [X] T001 Crear la rama `007-modulo-compras` desde `main` en `vamosjuntos-backend` y en `vamosjuntos-frontend`

---

## Phase 2: Foundational

**Purpose**: modelo, regla pura, almacén y tipos que usan todas las historias.

- [X] T002 Crear vamosjuntos-backend/purchases.go con:
  - `Item` (campos de Firestore `name`, `category`, `quantity`, `note`, `responsibleId` y `bought`, e `ID` con `firestore:"-"`) e `ItemInput`;
  - errores `errInvalidItem`, `errItemNotFound`, `errItemTaken` y `errNotResponsible`;
  - `validateItemInput`: nombre "De 1 a 60 caracteres tras recortar espacios", categoría "`comida` o `suministros`", cantidad "texto libre de hasta 30 caracteres", nota "hasta 200 caracteres";
  - `applyItem(item Item, cmd itemCmd, me string) (Item, bool, error)` con las reglas de research R3: editar no cambia responsable ni estado; tomar asigna a `me` o falla con `errItemTaken`; soltar, marcar y desmarcar solo para el responsable (`errNotResponsible`); soltar deja `bought = false`;
  - `sortItems`: `comida` antes de `suministros`, nombre sin acentos (`foldAccents`) e `ID`.
- [X] T003 [P] Crear vamosjuntos-backend/purchases_test.go con tablas para `validateItemInput` (límites de cada campo y recorte), `applyItem` (todas las transiciones de data-model.md y sus errores) y `sortItems`
- [X] T004 Agregar a vamosjuntos-backend/purchases.go `PurchaseStore` con `NewPurchaseStore(client, eventID)` sobre `events/{EVENT_ID}/items`:
  - `List`;
  - `Create(ctx, in, by)`;
  - `Apply(ctx, id, cmd, me)`: transacción que lee el documento (`errItemNotFound` si no existe), corre `applyItem` y escribe o borra.
  Las escrituras guardan `updatedAt` y `updatedBy`.
- [X] T005 Agregar a vamosjuntos-backend/errors.go los códigos `invalid_item`, `item_not_found`, `item_taken` y `not_responsible`
- [X] T006 En vamosjuntos-backend/handlers.go:
  - campo `purchases` en `server` y parámetro en `NewServer`;
  - en vamosjuntos-backend/main.go, `NewPurchaseStore`;
  - ajustar las llamadas a `NewServer` en las pruebas.
- [X] T007 [P] Agregar a vamosjuntos-frontend/src/api/types.ts `Item`, `ItemInput`, `ItemCategory` y `Purchases`, y a vamosjuntos-frontend/src/messages.ts los mensajes de contracts/api.md

**Checkpoint**: la regla pura pasa sus pruebas y el servidor compila con el almacén nuevo.

---

## Phase 3: User Story 1 - Ver la lista de compras (Priority: P1) 🎯 MVP

**Goal**: ver los ítems por categoría, con responsable, estado y resumen.

**Independent Test**: escenarios 1, 2 (vista) y 13 del quickstart.

- [X] T008 [US1] Crear vamosjuntos-backend/purchases_handlers.go con la interfaz `purchaseStore` y `GET /api/purchases` (`items` ordenados con `responsible` resuelto por el catálogo, y `summary`), y registrar la ruta al final de `routes()` en vamosjuntos-backend/handlers.go con `requireGuest`
- [X] T009 [P] [US1] Crear vamosjuntos-backend/purchases_handlers_test.go con `fakePurchases` (usa `applyItem`): lista vacía con resumen en cero, `401` sin token y `403` sin vincular
- [X] T010 [US1] Crear vamosjuntos-frontend/src/pages/PurchasesPage.tsx:
  - resumen "X por comprar · Y comprados";
  - secciones "Comida" y "Suministros" con nombre, cantidad, nota, responsable (o "Sin responsable") y estado;
  - estado vacío "Aún no hay nada en la lista. Agrega lo primero que haga falta." y error amigable al cargar;
  - "Volver al inicio".
- [X] T011 [US1] Agregar la ruta `/compras` en el bloque `confirmed` de vamosjuntos-frontend/src/App.tsx y el botón "Compras: comida y suministros" en vamosjuntos-frontend/src/pages/HomePage.tsx, debajo del de Itinerario

**Checkpoint**: la lista de compras se ve (MVP).

---

## Phase 4: User Story 2 - Agregar un ítem (Priority: P1)

**Goal**: cualquier confirmado agrega lo que hace falta.

**Independent Test**: escenarios 2, 3 y 4 del quickstart.

- [X] T012 [US2] Agregar `POST /api/purchases/items` en vamosjuntos-backend/purchases_handlers.go (`201` con la lista) y `writePurchaseError`, con pruebas para `201` y `invalid_item`
- [X] T013 [US2] Crear vamosjuntos-frontend/src/pages/ItemForm.tsx:
  - nombre, categoría (dos opciones), cantidad y nota;
  - aviso "Ya hay un ítem llamado así." con "Guardar de todas formas" si el nombre sin acentos ni mayúsculas ya existe (research R6);
  - en PurchasesPage.tsx, botón "Agregar ítem".

---

## Phase 5: User Story 3 - Encargarse de un ítem y marcarlo comprado (Priority: P2)

**Goal**: "Yo lo llevo", marcar comprado y soltar, con un solo responsable.

**Independent Test**: escenarios 5 a 10 del quickstart.

- [X] T014 [US3] Agregar `POST /api/purchases/items/{itemID}/take`, `/release` y `/bought` (`{"bought": bool}`) en vamosjuntos-backend/purchases_handlers.go, con pruebas para tomar, `item_taken`, marcar, `not_responsible` (otro invitado y un ítem sin responsable) y soltar (vuelve a "por comprar")
- [X] T015 [US3] En vamosjuntos-frontend/src/pages/PurchasesPage.tsx:
  - "Yo lo llevo" en ítems sin responsable;
  - para el responsable, "Ya lo compré" / "Aún no lo compro" y "Soltar";
  - ante `item_taken`, recargar y mostrar "Este ítem ya lo lleva {nombre}." con el nombre de la lista nueva.

---

## Phase 6: User Story 4 - Editar o eliminar un ítem (Priority: P3)

**Goal**: corregir o eliminar cualquier ítem.

**Independent Test**: escenarios 11 y 12 del quickstart.

- [X] T016 [US4] Agregar `PUT` y `DELETE /api/purchases/items/{itemID}` en vamosjuntos-backend/purchases_handlers.go, con pruebas para editar (conserva responsable y estado), eliminar (`204`) y `item_not_found`
- [X] T017 [US4] En vamosjuntos-frontend/src/pages/PurchasesPage.tsx:
  - "Editar" abre ItemForm con los datos;
  - "Eliminar" pide confirmación en la misma tarjeta, con "Lo lleva {nombre}" si tiene responsable;
  - ante `item_not_found`, recargar.

---

## Phase 7: Polish & Cross-Cutting Concerns

- [X] T018 Pasar las compuertas: backend (`gofmt -l .` sin salida, `go vet ./...`, `golangci-lint run`, `go test ./...`) y frontend (`npm run lint`, `npm run build`)
- [X] T019 Validar en local la UI a 360 px con `/api/*` simulado (escenarios 1 a 9, 11, 12 y 14)
- [X] T020 (visto bueno) Integrar en `main`, publicar backend y luego frontend y recorrer el quickstart en `vamosjuntos.club` con cuentas de prueba sobre invitados pendientes, incluido el escenario 10 (carrera), borrando solo lo creado
- [X] T021 Registrar aquí las notas de implementación y hacer commit de specs/007-modulo-compras

---

## Dependencies & Execution Order

- **Setup → Foundational**: T001 → T002 a T007. T003 y T007 son paralelas al resto.
- **Historias**: US1 → US2 → US3 → US4. US1 da la pantalla que usan las demás; en el backend,
  todas agregan rutas al mismo archivo.
- **Polish**: al final. T020 publica el backend antes que el frontend.

## Parallel Example

```bash
Task: "purchases_test.go (T003)"
Task: "tipos y mensajes del frontend (T007)"
Task: "purchases_handlers_test.go (T009) mientras se hace PurchasesPage.tsx (T010)"
```

## Implementation Strategy

- **MVP**: Setup, Foundational, US1 y US2. Con eso la familia ya arma la lista entre todos.
- **Luego**: US3 (encargarse y marcar), que reemplaza el chat de WhatsApp, y US4 (editar o
  eliminar).

## Implementation Notes (2026-10-31)

- **Backend** `625a9b3`:
  - `purchases.go`: `Item`, `validateItemInput`, `applyItem` (pura), `sortItems` y
    `PurchaseStore`, con una transacción por documento;
  - `purchases_handlers.go`: la interfaz `purchaseStore` y los 7 handlers;
  - pruebas en `purchases_test.go` y `purchases_handlers_test.go`.
- **Frontend** `2e9c2a9`:
  - `PurchasesPage.tsx` e `ItemForm.tsx`;
  - ruta `/compras` y botón en la pantalla principal.
- **Hallazgo corregido en T019**: al editar un ítem sin cambiarle el nombre, el aviso de nombre
  repetido aparecía si existía otro parecido. Ahora solo se avisa con un nombre nuevo o cambiado
  (`2e9c2a9`).
- **Desviaciones**: ninguna respecto al contrato. Ante `item_taken`, el nombre del mensaje sale de
  la lista recargada (research R4).
- **Validación**:
  - compuertas en verde;
  - UI a 360 px con `/api/*` simulado: 18/18 en local y 18/18 en `vamosjuntos.club`, incluida la
    carrera simulada ("Este ítem ya lo lleva Liam.");
  - API en producción con 2 cuentas de prueba sobre invitados pendientes: 15/15, incluido el
    escenario 10 real (dos "Yo lo llevo" simultáneos: `200`/`409` y un solo responsable).
- **Limpieza**: ítems y cuentas de prueba borrados. `items` quedó en 0 y `links` y `attendance`
  en 4, como antes.
