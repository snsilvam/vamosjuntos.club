# Feature Specification: Módulo Compras

**Feature Branch**: `007-modulo-compras`

**Created**: 2026-10-30

**Status**: Draft

**Input**: Derivado de la constitución v1.1.0 ("Dominio de la V1"): "Módulo Compras: comida y
suministros."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ver la lista de compras (Priority: P1)

Un invitado entra a Compras y ve qué hay que llevar al paseo, separado en "Comida" y
"Suministros": cada ítem con su cantidad, quién se encarga y si ya está comprado.

**Why this priority**: evita que falten cosas o que tres personas lleven lo mismo.

**Independent Test**: con ítems registrados, la pantalla los muestra por categoría con su estado y
responsable.

**Acceptance Scenarios**:

1. **Given** hay ítems registrados, **When** un invitado abre Compras, **Then** los ve agrupados en
   "Comida" y "Suministros", con cantidad, responsable (o "Sin responsable") y estado.
2. **Given** la lista, **When** se abre, **Then** se ve un resumen "X por comprar · Y comprados".
3. **Given** no hay ítems, **When** se abre Compras, **Then** se ve un mensaje que invita a agregar
   el primero.

---

### User Story 2 - Agregar un ítem (Priority: P1)

Un invitado confirmado agrega algo que hace falta: nombre, categoría, cantidad y una nota
opcional.

**Why this priority**: la lista se construye entre todos.

**Independent Test**: un ítem agregado aparece en su categoría como "por comprar" y "sin
responsable".

**Acceptance Scenarios**:

1. **Given** un invitado confirmado, **When** agrega "Arroz", categoría Comida, cantidad
   "5 kg", **Then** aparece en Comida como "por comprar" y "sin responsable".
2. **Given** el formulario, **When** se deja el nombre vacío, **Then** se muestra un mensaje
   amigable y no se guarda.

---

### User Story 3 - Encargarse de un ítem y marcarlo comprado (Priority: P2)

Un invitado toca "Yo lo llevo" en un ítem sin responsable y queda a su cargo; cuando lo compra, lo
marca como comprado. Si ya no puede, lo suelta.

**Why this priority**: es la coordinación que reemplaza el chat de WhatsApp.

**Independent Test**: un invitado se asigna un ítem, lo marca comprado y el resumen se actualiza.

**Acceptance Scenarios**:

1. **Given** un ítem sin responsable, **When** un invitado toca "Yo lo llevo", **Then** aparece
   como responsable.
2. **Given** un ítem con responsable, **When** otro invitado intenta tomarlo, **Then** ve "Este
   ítem ya lo lleva {nombre}." y no cambia.
3. **Given** un ítem a su cargo, **When** el responsable lo marca comprado, **Then** el estado pasa
   a "comprado" y el resumen se actualiza.
4. **Given** un ítem a su cargo, **When** el responsable lo suelta, **Then** vuelve a "sin
   responsable" y "por comprar".

---

### User Story 4 - Editar o eliminar un ítem (Priority: P3)

Cualquier invitado confirmado corrige la cantidad o el nombre de un ítem, o lo elimina si ya no hace falta.

**Why this priority**: las necesidades cambian, pero con menos frecuencia.

**Independent Test**: al editar la cantidad se ve el nuevo valor; al eliminar, el ítem desaparece.

**Acceptance Scenarios**:

1. **Given** un ítem existente, **When** se edita su cantidad, **Then** se ve la nueva cantidad.
2. **Given** un ítem existente, **When** se elimina tras confirmar, **Then** desaparece de la
   lista.

---

### Edge Cases

- Dos invitados tocan "Yo lo llevo" en el mismo ítem al mismo tiempo: solo uno queda como
  responsable; el otro ve el mensaje de ítem ya asignado.
- Alguien agrega un ítem con el mismo nombre que otro existente: se permite, pero se muestra un
  aviso "Ya hay un ítem llamado así" antes de guardar.
- Se elimina un ítem que tiene responsable: se pide confirmación indicando quién lo lleva.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El módulo MUST estar disponible solo para invitados con sesión activa (spec 001).
- **FR-002**: El sistema MUST mostrar los ítems agrupados en las categorías "Comida" y
  "Suministros", con nombre, cantidad, nota, responsable y estado.
- **FR-003**: Un ítem MUST tener nombre y categoría; cantidad (texto libre, por ejemplo "5 kg") y
  nota son opcionales.
- **FR-004**: Un ítem MUST tener como máximo un responsable, que es un invitado confirmado.
- **FR-005**: Solo el responsable de un ítem MUST poder marcarlo como comprado, desmarcarlo o
  soltarlo.
- **FR-006**: El sistema MUST mostrar un resumen con el número de ítems por comprar y comprados.
- **FR-007**: Cualquier invitado confirmado MUST poder crear, editar (nombre, categoría, cantidad
  y nota) y eliminar cualquier ítem (misma regla de la spec 005, FR-009).
- **FR-008**: El sistema MUST impedir que dos personas queden como responsables del mismo ítem,
  incluso con acciones simultáneas.
- **FR-009**: Eliminar un ítem MUST pedir confirmación.
- **FR-010**: Todo error MUST presentarse como un mensaje amigable en español.

### Key Entities

- **Ítem de compra**: nombre, categoría (Comida o Suministros), cantidad (opcional), nota
  (opcional), responsable (opcional) y estado (por comprar o comprado).
- **Invitado**: definido en la spec 001.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un invitado agrega un ítem en menos de 20 segundos.
- **SC-002**: Un día antes del paseo, el 100 % de los ítems tienen responsable o figuran
  explícitamente como "sin responsable".
- **SC-003**: En ninguna prueba de acciones simultáneas un ítem queda con dos responsables.
- **SC-004**: La familia reporta no haber llevado productos duplicados por falta de coordinación.

## Assumptions

- No se registran precios, costos ni división de gastos (la constitución prohíbe pagos y extras no
  pedidos).
- Las categorías son solo "Comida" y "Suministros", como fija la constitución.
- La cantidad es texto libre; no hay unidades ni cálculos.
- No hay integración con tiendas ni listas externas.
- Depende de la spec 001 (sesión) y de la regla de permisos de la spec 005.
