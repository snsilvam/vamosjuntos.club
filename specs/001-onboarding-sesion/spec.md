# Feature Specification: Onboarding y Sesión

**Feature Branch**: `001-onboarding-sesion`

**Created**: 2026-10-30

**Status**: Draft

> **Reemplazo parcial (constitución v3.0.0)**: la spec
> [008-autenticacion-firebase](../008-autenticacion-firebase/spec.md) reemplaza FR-008, FR-009 y
> FR-011 (sesión propia y reingreso solo con código) por inicio de sesión con Firebase
> Authentication. El resto de esta spec sigue vigente.

**Input**: Derivado de la constitución v1.1.0 (Principio II y "Dominio de la V1"), enmendada en la
v2.0.0: "Todo usuario
nuevo debe seleccionar su nombre de la lista, ingresar su código de invitación del evento y
confirmar asistencia; si confirma, se oculta de la lista de onboarding. Todo flujo en la app exige
un usuario con sesión activa e invitado a un evento."

## Clarifications

### Session 2026-10-30

- Q: ¿Cómo vuelve a entrar un invitado que ya confirmó y perdió la sesión? → A: Solo con su
  código; el código identifica a la persona.
- Q: ¿Qué pasa si el invitado responde que no confirma? → A: Queda registrado como "No asistirá",
  sale de la lista de onboarding y no entra a la app.
- Q: ¿Puede alguien confirmar la asistencia de otro invitado? → A: Sí, escribiendo el código de
  esa persona; después también puede reservarle cupo de transporte.
- Q: ¿Dónde vive el catálogo de invitados? (enmienda de la constitución v2.0.0, pedida por el
  responsable) → A: En la base de datos del evento: un registro por invitado con nombre y código,
  que el organizador crea y edita a mano. Ya no viene de la configuración del servidor.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Primer ingreso y confirmación de asistencia (Priority: P1)

Un invitado abre la app por primera vez. Ve la lista de invitados pendientes, elige su nombre,
escribe el código de invitación que le entregó el organizador y, tras validarlo, la app le muestra
el evento ("Paseo Familiar Diciembre 2026", fechas y destino) y le pregunta si confirma su
asistencia. Al confirmar, queda con sesión activa, entra a la pantalla principal y su nombre
desaparece de la lista para los demás.

**Why this priority**: sin esto nadie entra a la app; es la puerta de todas las demás features.

**Independent Test**: con el catálogo de invitados configurado y el evento creado, un invitado
completa el flujo y llega a la pantalla principal; al abrir la app en otro dispositivo sin sesión,
su nombre ya no aparece en la lista.

**Acceptance Scenarios**:

1. **Given** un dispositivo sin sesión, **When** se abre la app, **Then** se muestra la lista de
   invitados pendientes, en orden alfabético, y la opción "Ya confirmé, entrar con mi código".
2. **Given** un invitado eligió su nombre, **When** ingresa su código correcto, **Then** ve el
   nombre, las fechas y el destino del evento y la pregunta "¿Confirmas tu asistencia?".
3. **Given** un invitado eligió su nombre, **When** ingresa un código incorrecto, **Then** ve un
   mensaje amigable ("Ese código no coincide con tu nombre. Revísalo e intenta de nuevo.") y
   permanece en el mismo paso.
4. **Given** el invitado ve la pregunta de confirmación, **When** confirma, **Then** queda con
   sesión activa, llega a la pantalla principal y su nombre ya no aparece en la lista de
   onboarding para ningún dispositivo.

---

### User Story 2 - Acceso protegido por sesión (Priority: P1)

Cualquier pantalla u operación de la app, excepto el onboarding y el reingreso, solo está
disponible para un invitado confirmado con sesión activa. Quien no tenga sesión es llevado al
onboarding.

**Why this priority**: la constitución exige que nadie fuera de la lista acceda a la información
de la familia.

**Independent Test**: sin sesión, intentar abrir directamente cualquier pantalla interna (o invocar
cualquier operación) siempre termina en el onboarding o en un rechazo, nunca en datos del evento.

**Acceptance Scenarios**:

1. **Given** un dispositivo sin sesión, **When** se intenta abrir una pantalla interna por enlace
   directo, **Then** la app muestra el onboarding.
2. **Given** una solicitud sin sesión válida, **When** llega al sistema, **Then** se rechaza como
   "no autenticado" sin devolver datos.
3. **Given** una sesión de alguien que ya no está en el catálogo de invitados, **When** hace una
   solicitud, **Then** se rechaza como "no autorizado" y la app lo devuelve al onboarding.

---

### User Story 3 - Volver a entrar con el código (Priority: P2)

Un invitado que ya confirmó pierde su sesión (cambió de celular, borró datos del navegador o
cerró sesión). Como su nombre ya no está en la lista, toca "Ya confirmé, entrar con mi código",
escribe su código y vuelve a entrar. El código basta para identificarlo.

**Why this priority**: sin esto, un confirmado que pierde la sesión queda fuera de la app hasta
el paseo.

**Independent Test**: un invitado confirmado, desde un dispositivo nuevo, recupera el acceso
escribiendo solo su código, sin ayuda del organizador.

**Acceptance Scenarios**:

1. **Given** un invitado confirmado sin sesión, **When** escribe su código en "Ya confirmé, entrar
   con mi código", **Then** obtiene una sesión como esa persona y llega a la pantalla principal.
2. **Given** un invitado confirmado, **When** reingresa, **Then** su estado sigue siendo
   "confirmado" y no se duplica su confirmación.
3. **Given** un código que no pertenece a ningún invitado, **When** se escribe, **Then** se muestra
   "No encontramos ese código. Revísalo e intenta de nuevo." y el intento cuenta para el bloqueo
   de FR-010.
4. **Given** el código de un invitado pendiente, **When** se escribe en el reingreso, **Then** se
   muestra "Aún no has confirmado. Busca tu nombre en la lista." y no se crea sesión.
5. **Given** el código de un invitado marcado "No asistirá", **When** se escribe en el reingreso,
   **Then** el sistema le vuelve a preguntar "¿Confirmas tu asistencia?" para que pueda cambiar de
   opinión.

---

### User Story 4 - Confirmar a otro invitado (Priority: P2)

Un invitado con sesión confirma la asistencia de otra persona que no usa la app (por ejemplo, un
niño o un abuelo): elige su nombre de la lista de pendientes, escribe el código de esa persona y
confirma. Sigue en su propia sesión.

**Why this priority**: varios invitados no tienen celular o son niños; sin esto, la confirmación
de familias completas es engorrosa.

**Independent Test**: desde la sesión de un invitado, se confirma a otro con su código; el otro sale
de la lista de pendientes y los conteos de la card cambian, mientras la sesión sigue siendo la del
primero.

**Acceptance Scenarios**:

1. **Given** un invitado con sesión, **When** elige "Confirmar a otro invitado", escoge un nombre
   pendiente y escribe el código correcto de esa persona, **Then** esa persona queda confirmada y
   sale de la lista de onboarding.
2. **Given** el mismo flujo, **When** el código no corresponde a la persona elegida, **Then** se
   muestra un mensaje amigable y no se confirma.
3. **Given** se confirmó a otra persona, **When** termina el flujo, **Then** la sesión sigue siendo
   la del invitado que hizo la confirmación.

---

### User Story 5 - Decir que no asistirá (Priority: P3)

Un invitado valida su código y responde que no asistirá. Queda registrado así, sale de la lista y
no entra a la app.

**Why this priority**: es el camino menos frecuente, pero deja un dato claro para la card
"Próximo viaje".

**Independent Test**: un invitado responde "No" y queda como "No asistirá", fuera de la lista y
sin sesión; la card lo cuenta en "no asistirán".

**Acceptance Scenarios**:

1. **Given** el invitado ve la pregunta de confirmación, **When** responde "No", **Then** queda
   registrado como "No asistirá", sale de la lista de onboarding, no obtiene sesión y ve "¡Te
   vamos a extrañar! Si cambias de opinión, entra con tu código."

---

### User Story 6 - Cerrar sesión (Priority: P3)

Un invitado que usó el celular de un familiar cierra su sesión para que el siguiente pueda entrar.

**Why this priority**: en una familia es común compartir dispositivos.

**Independent Test**: tras cerrar sesión, la app muestra el onboarding y ninguna pantalla interna
es accesible desde ese dispositivo.

**Acceptance Scenarios**:

1. **Given** un invitado con sesión, **When** elige "Cerrar sesión", **Then** la sesión termina y
   la app muestra el onboarding.

---

### Edge Cases

- El evento todavía no existe en la base de datos: el onboarding muestra "El paseo aún no está
  disponible. Intenta más tarde." y no permite continuar.
- El mismo invitado completa el onboarding casi al mismo tiempo en dos dispositivos: ambos pueden
  quedar con sesión; la confirmación se registra una sola vez.
- Dos invitados intentan confirmar a la misma persona al mismo tiempo: la confirmación se registra
  una sola vez.
- Un invitado elige por error el nombre de otra persona: el código no coincide y no puede avanzar.
- Muchos intentos fallidos de código: se bloquean temporalmente los intentos (FR-010).
- Todos los invitados ya respondieron: la lista se muestra vacía con un mensaje y la opción de
  reingreso.
- El catálogo supera 25 invitados, tiene códigos repetidos, códigos de menos de 6 caracteres o
  un invitado sin nombre: si ocurre al arrancar, el sistema no inicia; si ocurre por una edición
  posterior, las operaciones que usan el catálogo responden con el mensaje amigable genérico.
  En ambos casos el error queda registrado para el organizador y ningún invitado ve un error
  técnico.
- Se agrega un invitado nuevo al catálogo (hasta completar 25): aparece en la lista de pendientes
  de inmediato, sin volver a publicar la app y sin afectar a los demás.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST mostrar la lista de invitados tomada del catálogo que el
  organizador mantiene a mano en la base de datos del evento. La app MUST NOT permitir agregar,
  editar ni borrar invitados, y los cambios del organizador MUST verse sin volver a publicar.
- **FR-002**: La lista de onboarding MUST mostrar solo los invitados pendientes (ni confirmados ni
  "No asistirá"), en orden alfabético.
- **FR-003**: Tras elegir un nombre, el sistema MUST pedir el código de invitación propio de ese
  invitado y validarlo ignorando espacios al inicio o al final y diferencias entre mayúsculas y
  minúsculas.
- **FR-004**: El sistema MUST obtener los datos del evento del registro que crea el organizador.
  Si no existe, el onboarding MUST mostrar un mensaje amigable y no permitir continuar.
- **FR-005**: Con el código válido, el sistema MUST mostrar nombre, fechas (hora Colombia) y
  destino del evento, y preguntar si el invitado confirma su asistencia.
- **FR-006**: Si el invitado confirma, el sistema MUST registrar la confirmación con su fecha y
  hora, iniciar su sesión y retirarlo de la lista de onboarding.
- **FR-007**: Si el invitado no confirma, el sistema MUST registrarlo como "No asistirá",
  retirarlo de la lista de onboarding y no iniciar sesión.
- **FR-008**: Toda pantalla y operación, salvo el onboarding y el reingreso, MUST exigir una sesión
  activa de un invitado confirmado presente en el catálogo; sin ella, la app MUST llevar al
  onboarding y el sistema MUST rechazar la operación como "no autenticado" (sin sesión) o "no
  autorizado" (sesión de alguien que ya no está invitado o ya no está confirmado).
- **FR-009**: La sesión MUST mantenerse activa en el dispositivo hasta que el invitado cierre
  sesión o hasta 7 días después del fin del evento (2027-01-02 hora Colombia), lo que ocurra
  primero.
- **FR-010**: Tras 5 intentos de código fallidos seguidos desde un mismo dispositivo, el sistema
  MUST bloquear nuevos intentos desde ese dispositivo durante 15 minutos y mostrar un mensaje
  amigable. Aplica al onboarding, al reingreso y a la confirmación de otro invitado.
- **FR-011**: El sistema MUST ofrecer un reingreso solo con código: si el código pertenece a un
  invitado confirmado, inicia su sesión; si pertenece a un invitado "No asistirá", le vuelve a
  preguntar si confirma; si pertenece a un pendiente, lo remite a la lista sin crear sesión.
- **FR-012**: Un invitado con sesión MUST poder cerrarla desde la app.
- **FR-013**: El sistema MUST rechazar un catálogo con más de 25 invitados.
- **FR-014**: Un invitado con sesión MUST poder confirmar la asistencia de un invitado pendiente
  escribiendo el código de esa persona. La confirmación MUST registrar quién la hizo y MUST NOT
  cambiar la sesión actual.
- **FR-015**: Ningún mensaje visible al invitado MUST contener detalles técnicos; todo error se
  presenta como un mensaje amigable en español.
- **FR-016**: Los códigos del catálogo MUST ser únicos (sin distinguir mayúsculas y minúsculas) y
  tener al menos 6 caracteres; el sistema MUST rechazar un catálogo que no cumpla.

### Key Entities

- **Evento**: el "Paseo Familiar Diciembre 2026". Nombre, fecha y hora de inicio y fin, destino,
  lugares de salida y capacidad máxima (25). Lo crea el organizador a mano.
- **Invitado**: una persona del catálogo, registrada a mano por el organizador en la base de datos
  del evento. Nombre visible, código de invitación propio y único, y estado de asistencia:
  pendiente, confirmado o "No asistirá".
- **Respuesta de asistencia**: el registro de que un invitado confirmó o dijo que no asistirá, con
  fecha y hora, y quién la registró (el propio invitado u otro invitado).
- **Sesión**: el acceso activo de un invitado confirmado en un dispositivo, con vencimiento.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un invitado completa el primer ingreso (nombre, código y confirmación) en menos de
  1 minuto.
- **SC-002**: Al menos 90 % de los invitados completan el onboarding al primer intento sin pedir
  ayuda al organizador.
- **SC-003**: El 100 % de las pantallas internas y operaciones probadas sin sesión terminan en el
  onboarding o en un rechazo; ninguna muestra datos del evento.
- **SC-004**: Ningún invitado confirmado o "No asistirá" vuelve a aparecer en la lista de
  onboarding después de recargar la app.
- **SC-005**: Un invitado confirmado recupera el acceso desde un dispositivo nuevo en menos de 30
  segundos.
- **SC-006**: Ningún mensaje mostrado al usuario contiene códigos de error, trazas o textos
  técnicos.

## Assumptions

- El código de invitación es único por invitado, como fija la constitución ("Cada invitado tiene
  un código de confirmación propio").
- El organizador elige códigos difíciles de adivinar (no secuenciales como "codigo 1", "codigo 2")
  y los entrega personalmente (por ejemplo, por WhatsApp); la app no envía ni recupera códigos.
  Quien olvide su código se lo pide al organizador.
- Los 3 cupos restantes (hasta 25) los agrega el organizador directamente en la base de datos;
  no hay registro de usuarios nuevos dentro de la app.
- Hay un único evento en la V1, así que no existe un paso de "elegir evento".
- El reingreso de un invitado "No asistirá" para cambiar de opinión no rompe el orden del
  onboarding de la constitución: esa persona ya pasó por la selección de nombre y el código.
- Confirmar a otro invitado no le da sesión a esa persona; si luego quiere usar la app, entra con
  su código por el reingreso.
- Los invitados usan principalmente el navegador de su celular.
