# Feature Specification: Confirmar a otro invitado sin su código

**Feature Branch**: `009-confirmar-otro-sin-codigo`

**Created**: 2026-11-06

**Status**: Implemented (sin publicar)

> **Reemplazo parcial de la 008**: esta spec reemplaza la US6 y el FR-013 de
> [008-autenticacion-firebase](../008-autenticacion-firebase/spec.md) (confirmar a otro escribiendo
> su código) y acota su FR-014: el límite de intentos fallidos queda solo para vincular una cuenta.
> El resto de la 008 sigue vigente.

**Input**: Pedido del responsable (2026-11-06): "La confirmación de un participante a un evento se
puede realizar por otro participante que ya confirmó y que se registró en Firebase." Ese flujo ya
existía (008, US6), pero exigía el código de la otra persona. El responsable aclaró que lo que
cambia es eso: ya no se pide el código.

## Clarifications

### Session 2026-11-06

- Q: El flujo "Confirmar a otro invitado" ya existe (008, US6/FR-013). ¿Qué cambia? → A: ya no se
  pide el código de la otra persona; basta con elegir su nombre.
- Q: Sin código, ¿a qué pendientes puede confirmar otro invitado? → A: solo a los pendientes que no
  han vinculado una cuenta. Quien ya entró con su cuenta responde por sí mismo y nadie le gana la
  respuesta.
- Q: Un toque equivocado confirma a la persona equivocada y un confirmado no puede pasar a "No
  asistirá". ¿Cómo se corrige? → A: la app pregunta "¿Confirmar a {nombre}?" antes de enviar. Si
  igual hay un error, lo corrige el organizador en la base de datos, como ya pasa con los vínculos.

## User Scenarios & Testing *(mandatory)*

El actor es un familiar que ya inició sesión con su cuenta, la vinculó a su invitado y confirmó su
asistencia. Confirma a personas que no usan la app, como un niño o un abuelo sin celular.

### User Story 1 - Confirmar a otro invitado eligiendo su nombre (Priority: P1)

Un familiar confirmado entra a "Confirmar a otro invitado" y ve la lista de invitados pendientes
que no tienen cuenta. Toca un nombre y la app le pregunta "¿Confirmar a {nombre}?". Si acepta, esa
persona queda confirmada. No escribe ningún código.

**Why this priority**: es lo que pidió el responsable. Buscar y transcribir el código de cada niño
o abuelo es el paso que sobra.

**Independent Test**: con una cuenta vinculada y confirmada, confirmar a un invitado pendiente sin
cuenta sin escribir códigos. Esa persona sale de la lista, los confirmados de la card "Próximo
viaje" suben en 1 y la sesión de quien confirmó no cambia.

**Acceptance Scenarios**:

1. **Given** un familiar confirmado, **When** abre "Confirmar a otro invitado", **Then** ve en
   orden alfabético los invitados pendientes que no tienen una cuenta vinculada, sin ningún campo
   de código.
2. **Given** la lista, **When** toca un nombre, **Then** la app le pregunta "¿Confirmar a
   {nombre}?", le advierte que solo el organizador puede deshacerlo y todavía no registra nada.
3. **Given** la pregunta, **When** acepta, **Then** esa persona queda confirmada, se registra quién
   la confirmó y cuándo, sale de la lista, la app muestra "¡{nombre} quedó confirmado!" y la sesión
   de quien confirmó no cambia.
4. **Given** la pregunta, **When** cancela, **Then** vuelve a la lista y nada cambia.
5. **Given** una persona confirmada así, **When** más tarde inicia sesión con su propia cuenta y
   usa "Ya me confirmaron, entrar con mi código", **Then** se vincula con su código y entra directo
   a la pantalla principal (008, US4).

---

### User Story 2 - Quien ya tiene cuenta responde por sí mismo (Priority: P1)

Un invitado que ya vinculó su cuenta pero aún no responde no aparece en la lista de "Confirmar a
otro invitado". Si alguien intenta confirmarlo por otra vía, el sistema lo rechaza.

**Why this priority**: sin el código, nada impediría confirmar a alguien que está usando la app.
Esta regla le deja la decisión a esa persona (aclaración del 2026-11-06).

**Independent Test**: con una cuenta vinculada y pendiente, comprobar que su nombre no aparece en
la lista de otro confirmado y que una confirmación directa se rechaza con un mensaje amigable.

**Acceptance Scenarios**:

1. **Given** un invitado pendiente con cuenta vinculada, **When** un confirmado abre "Confirmar a
   otro invitado", **Then** ese invitado no aparece en la lista.
2. **Given** que ese invitado vinculó su cuenta mientras otro tenía la lista abierta, **When** el
   otro intenta confirmarlo, **Then** el sistema lo rechaza, la app muestra "{nombre} ya entró con
   su propia cuenta y responde desde ahí." y actualiza la lista.
3. **Given** un invitado que ya respondió (confirmado o "No asistirá"), **When** alguien intenta
   confirmarlo, **Then** el sistema lo rechaza con "Esta persona ya respondió la invitación." y la
   app actualiza la lista (como en la 008).

### Edge Cases

- **Dos confirmados confirman a la misma persona al mismo tiempo**: solo una confirmación cuenta.
  El otro ve "Esta persona ya respondió la invitación." y su lista se actualiza.
- **La persona vincula su cuenta en el mismo instante en que otro la confirma**: el sistema aplica
  las dos en un orden. O quedó confirmada antes de vincular (y al vincular entra directo), o ya
  estaba vinculada y la confirmación se rechaza (US2-2). Nunca queda confirmada por otro después de
  haber vinculado su cuenta.
- **Confirmación por error** (pasó la pregunta igual): la persona queda confirmada y la app no
  ofrece deshacer. El organizador borra esa respuesta en la base de datos y la persona vuelve a
  pendiente.
- **El organizador quita a la persona del catálogo con la lista abierta**: "No encontramos a esa
  persona en la lista de invitados." y la lista se actualiza.
- **No hay pendientes sin cuenta**: la app dice "No hay invitados pendientes sin cuenta. Quien ya
  entró con su cuenta responde desde ahí."
- **Quien confirma deja de estar confirmado o pierde su vínculo** (lo cambió el organizador): el
  sistema rechaza la operación como "no autorizado", como cualquier pantalla interna (008, FR-010).
- **Cupo de 25 invitados**: no cambia; confirmar no agrega personas al catálogo.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Un invitado con cuenta vinculada y confirmado MUST poder confirmar a otro invitado
  eligiendo su nombre, sin escribir el código de esa persona. Reemplaza el FR-013 de la 008.
- **FR-002**: La lista de "Confirmar a otro invitado" MUST mostrar solo los invitados del catálogo
  que no han respondido y no tienen una cuenta vinculada, en orden alfabético. MUST NOT mostrar
  códigos, correos ni datos de las cuentas.
- **FR-003**: Antes de registrar la confirmación, la app MUST preguntar "¿Confirmar a {nombre}?" y
  advertir que solo el organizador puede deshacerla. Cancelar MUST NOT registrar nada.
- **FR-004**: El sistema MUST confirmar solo a invitados que no han respondido y no tienen cuenta
  vinculada. Lo MUST validar al registrar la confirmación, no solo al mostrar la lista, incluso si
  en ese mismo instante la persona vincula su cuenta u otro familiar la confirma.
- **FR-005**: Al confirmar, el sistema MUST registrar el estado confirmado con su fecha y hora y
  quién la confirmó. MUST NOT vincular ninguna cuenta ni cambiar la sesión de quien confirma.
- **FR-006**: La app MUST NOT ofrecer deshacer una confirmación hecha por otro; solo el
  organizador la corrige, fuera de la app.
- **FR-007**: Confirmar a otro MUST NOT contar para el límite de intentos fallidos. Ese límite (008,
  FR-014) queda solo para vincular una cuenta con un código.
- **FR-008**: Cada rechazo MUST mostrarse con un mensaje amigable en español y actualizar la lista:
  ya respondió, ya tiene cuenta, ya no está en el catálogo. El éxito MUST mostrarse como "¡{nombre}
  quedó confirmado!".
- **FR-009**: Ser confirmado por otro MUST NOT dar acceso a la app. Para entrar, la persona MUST
  vincular su propia cuenta con su código (008, FR-004 y US4).

### Key Entities

- **Invitado**: una persona del catálogo (nombre y código), sin cambios.
- **Vínculo** (008): la relación entre una cuenta y un invitado. Ahora también decide si un
  invitado puede ser confirmado por otro: con vínculo, no.
- **Respuesta de asistencia** (001 y 008): sin cambios. Cuando confirma otro, "quién la registró"
  es el invitado que confirmó.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un familiar confirmado confirma a otra persona desde la pantalla principal en menos
  de 20 segundos, con 3 toques y sin escribir nada.
- **SC-002**: El 100 % de los intentos de confirmar a un invitado con cuenta vinculada, o que ya
  respondió, se rechazan, incluso con acciones simultáneas.
- **SC-003**: Ninguna confirmación de otro se registra sin que quien confirma haya aceptado la
  pregunta "¿Confirmar a {nombre}?".
- **SC-004**: En el flujo de confirmar a otro no se pide ni se muestra ningún código de invitación.
- **SC-005**: Ninguna cuenta obtiene acceso a la app por haber sido confirmada por otro.

## Assumptions

- La familia confía en sus miembros confirmados: cualquiera de ellos puede confirmar a cualquier
  pendiente sin cuenta. Queda registrado quién confirmó a quién, así el organizador puede revisar.
- Confirmar a otro solo confirma. Responder "No asistirá" en nombre de otro sigue fuera de alcance.
- Quien ya vinculó su cuenta y no ha respondido recibe la pregunta de asistencia al iniciar sesión
  (008, FR-007), así que no necesita que otro lo confirme.
- Para deshacer una confirmación hecha por error, el organizador borra la respuesta de esa persona
  en la base de datos; la persona vuelve a pendiente.
- Las confirmaciones hechas antes con código quedan igual; no hay datos que migrar.
