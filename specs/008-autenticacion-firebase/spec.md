# Feature Specification: Autenticación con Firebase

**Feature Branch**: `008-autenticacion-firebase`

**Created**: 2026-10-30

**Status**: Implemented

> **Reemplazo parcial**: la spec [009-confirmar-otro-sin-codigo](../009-confirmar-otro-sin-codigo/spec.md)
> reemplaza la US6 y el FR-013 (confirmar a otro con su código): ahora basta con elegir su nombre,
> y solo se puede confirmar a pendientes sin cuenta vinculada. También acota el FR-014: el límite
> de intentos queda solo para vincular. El resto de esta spec sigue vigente.

**Input**: Pedido del responsable, recogido en la constitución v3.0.0 (Principio II): "Cada familiar
inicia sesión con Firebase Authentication usando Google o correo y contraseña, igual que
kaizen-app. La primera vez, ya autenticado, elige su nombre y escribe su código de invitación: eso
vincula su cuenta con ese invitado una sola vez. Las siguientes veces entra solo con su cuenta."

**Relación con la spec 001**: esta feature reemplaza la sesión que firmaba el propio sistema y el
reingreso solo con código (FR-008, FR-009 y FR-011 de la 001). Se conservan la lista de invitados
pendientes, la validación de códigos, la respuesta de asistencia, confirmar a otro invitado, el
bloqueo por intentos fallidos y el catálogo en la base de datos.

## User Scenarios & Testing *(mandatory)*

El actor es un familiar invitado al paseo, que usa la app casi siempre desde el celular.

### User Story 1 - Primer ingreso: iniciar sesión, vincularse y confirmar (Priority: P1)

Un familiar abre la app por primera vez e inicia sesión con su cuenta de Google o con su correo y
contraseña (si no tiene cuenta de correo en la app, la crea ahí mismo). Como su cuenta aún no está
vinculada a ningún invitado, la app le muestra la lista de invitados pendientes. Elige su nombre,
escribe su código de invitación y la app vincula su cuenta con ese invitado. Luego le muestra los
datos del paseo y le pregunta si asiste.

**Why this priority**: sin esto nadie entra a la app.

**Independent Test**: con el evento y el catálogo creados, una cuenta nueva completa inicio de
sesión, elección de nombre, código y confirmación, y llega a la pantalla principal.

**Acceptance Scenarios**:

1. **Given** un familiar sin sesión, **When** abre la app, **Then** ve la pantalla de inicio de
   sesión con "Entrar con Google" y la opción de correo y contraseña (entrar o crear cuenta).
2. **Given** una cuenta recién autenticada y sin vincular, **When** termina de iniciar sesión,
   **Then** ve la lista de invitados pendientes en orden alfabético.
3. **Given** la lista, **When** elige su nombre y escribe su código correcto (sin importar
   mayúsculas ni espacios al inicio o al final), **Then** su cuenta queda vinculada con ese
   invitado y ve el nombre, las fechas (hora Colombia) y el destino del paseo con la pregunta de
   si asiste.
4. **Given** la pregunta, **When** confirma, **Then** queda registrado como confirmado, sale de la
   lista de pendientes y entra a la pantalla principal.
5. **Given** la pregunta, **When** responde que no, **Then** queda registrado como "No asistirá",
   sale de la lista de pendientes, ve un mensaje de despedida amable y no entra a la app.
6. **Given** la lista, **When** escribe un código que no es el de ese nombre, **Then** ve un
   mensaje amigable, la cuenta no se vincula y puede reintentar.

---

### User Story 2 - Volver a entrar solo con la cuenta (Priority: P1)

Un familiar confirmado vuelve a la app (en el mismo celular o en otro) e inicia sesión con la misma
cuenta. Entra directo a la pantalla principal, sin elegir nombre ni escribir código.

**Why this priority**: es el uso diario; pedir el código cada vez es lo que el responsable quiere
eliminar.

**Independent Test**: una cuenta ya vinculada y confirmada inicia sesión en otro navegador y llega
a la pantalla principal sin pasar por la lista ni por el código.

**Acceptance Scenarios**:

1. **Given** una cuenta vinculada a un invitado confirmado, **When** inicia sesión, **Then** entra
   a la pantalla principal saludándolo por el nombre del invitado.
2. **Given** una sesión iniciada, **When** cierra y vuelve a abrir la app en el mismo dispositivo,
   **Then** sigue dentro sin volver a iniciar sesión.

---

### User Story 3 - Acceso protegido (Priority: P1)

Ninguna pantalla ni operación de la app, salvo el inicio de sesión y el onboarding, funciona sin
una cuenta autenticada, vinculada a un invitado del catálogo y confirmada.

**Why this priority**: la app maneja datos de una familia (Principio II).

**Independent Test**: abrir una ruta interna sin sesión lleva al inicio de sesión; con una cuenta
sin vincular, lleva al onboarding; y el sistema rechaza las operaciones directas en ambos casos.

**Acceptance Scenarios**:

1. **Given** alguien sin sesión, **When** abre una pantalla interna, **Then** la app lo lleva al
   inicio de sesión y el sistema rechaza la operación como "no autenticado".
2. **Given** una cuenta autenticada pero sin vincular, **When** intenta usar una operación interna,
   **Then** el sistema la rechaza como "no autorizado" y la app la lleva al onboarding.
3. **Given** una cuenta vinculada a un invitado que el organizador quitó del catálogo, o que no está
   confirmado, **When** intenta usar la app, **Then** el sistema la rechaza como "no autorizado".

---

### User Story 4 - Vincularse cuando ya no aparezco en la lista (Priority: P2)

Un invitado que otro familiar ya confirmó (US6) no aparece en la lista de pendientes. Inicia sesión
y usa "Ya me confirmaron, entrar con mi código": escribe su código y la app vincula su cuenta con
ese invitado.

**Why this priority**: confirmar a otro (US5) deja personas confirmadas sin cuenta; sin esta
historia no podrían entrar nunca.

**Independent Test**: un invitado confirmado por otra persona inicia sesión con una cuenta nueva,
escribe su código en "entrar con mi código" y llega a la pantalla principal.

**Acceptance Scenarios**:

1. **Given** una cuenta sin vincular y un invitado confirmado sin cuenta, **When** escribe el código
   de ese invitado en "entrar con mi código", **Then** la cuenta queda vinculada y entra a la
   pantalla principal.
2. **Given** una cuenta sin vincular y un invitado pendiente, **When** escribe su código en "entrar
   con mi código", **Then** la cuenta queda vinculada y la app le pregunta si asiste.
3. **Given** una cuenta sin vincular y un invitado "No asistirá" sin cuenta (por ejemplo, porque
   el organizador deshizo su vínculo), **When** escribe su código, **Then** la cuenta queda
   vinculada y la app le vuelve a preguntar si asiste.

---

### User Story 5 - Cambiar de opinión (Priority: P2)

Un invitado que respondió "No asistirá" vuelve a iniciar sesión con su cuenta vinculada. La app le
vuelve a mostrar los datos del paseo y le pregunta si ahora asiste.

**Why this priority**: los planes familiares cambian; pasa poco, pero debe ser posible.

**Independent Test**: una cuenta vinculada a un invitado "No asistirá" inicia sesión, confirma y
entra a la pantalla principal.

**Acceptance Scenarios**:

1. **Given** una cuenta vinculada a un invitado "No asistirá", **When** inicia sesión, **Then** ve
   la pregunta de asistencia con los datos del paseo.
2. **Given** esa pregunta, **When** confirma, **Then** queda confirmado y entra a la pantalla
   principal; **When** vuelve a decir que no, **Then** sigue como "No asistirá" y no entra.

---

### User Story 6 - Confirmar a otro invitado (Priority: P2)

Un familiar confirmado confirma a un invitado pendiente (por ejemplo, un niño o un abuelo sin
celular) escribiendo el código de esa persona. Esto no vincula ninguna cuenta.

**Why this priority**: se conserva tal cual de la spec 001 (FR-014).

**Independent Test**: un confirmado confirma a otro pendiente con su código; esa persona sale de la
lista y la sesión de quien confirmó no cambia.

**Acceptance Scenarios**:

1. **Given** un familiar confirmado, **When** elige a un pendiente y escribe su código correcto,
   **Then** esa persona queda confirmada, se registra quién la confirmó y la sesión actual no cambia.
2. **Given** lo anterior, **When** escribe un código que no es de esa persona, **Then** ve un
   mensaje amigable y su sesión sigue activa.

---

### User Story 7 - Cerrar sesión (Priority: P3)

Un familiar cierra sesión desde la app (por ejemplo, en un celular prestado).

**Why this priority**: necesario pero de uso esporádico.

**Independent Test**: tras cerrar sesión, la app muestra el inicio de sesión y las operaciones
internas se rechazan.

**Acceptance Scenarios**:

1. **Given** una sesión iniciada, **When** pulsa "Cerrar sesión", **Then** vuelve a la pantalla de
   inicio de sesión y ninguna pantalla interna es accesible sin volver a entrar.

### Edge Cases

- **Cuenta ya vinculada intenta vincular otro invitado**: el sistema lo rechaza; una cuenta
  pertenece a un solo invitado.
- **Invitado ya vinculado a otra cuenta**: si alguien escribe su código desde otra cuenta, el
  sistema lo rechaza con un mensaje que lo remite al organizador. Solo el organizador puede deshacer
  un vínculo, editando la base de datos (por ejemplo, si el familiar perdió su cuenta).
- **Dos cuentas intentan vincular el mismo invitado al mismo tiempo**: solo una lo logra; la otra
  recibe el mensaje anterior.
- **El inicio de sesión falla o se cancela** (ventana de Google cerrada, contraseña incorrecta,
  correo ya registrado, sin conexión): mensaje amigable en español y se puede reintentar.
- **Olvidó la contraseña de correo**: puede pedir desde la app un correo para restablecerla.
- **La misma persona entra con Google y con correo** (dos cuentas distintas para el proveedor): solo
  la primera que se vinculó tiene acceso; la otra recibe el mensaje de invitado ya vinculado.
- **Muchos códigos fallidos** al vincular o al confirmar a otro: se bloquean los intentos de esa
  cuenta durante 15 minutos.
- **No existe el evento o el catálogo es inválido**: mensaje amigable, como en la spec 001.
- **El organizador quita a un invitado vinculado del catálogo**: esa cuenta pierde el acceso en su
  siguiente operación.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST permitir iniciar sesión con una cuenta de Google y con correo y
  contraseña, crear una cuenta con correo y contraseña, y pedir un correo para restablecer la
  contraseña, usando el proveedor de autenticación que fija la constitución.
- **FR-002**: Sin sesión, la app MUST mostrar la pantalla de inicio de sesión y no permitir ninguna
  otra pantalla.
- **FR-003**: Con una sesión cuya cuenta no está vinculada, la app MUST mostrar el onboarding: la
  lista de invitados pendientes (ni confirmados ni "No asistirá"), en orden alfabético, y la opción
  "entrar con mi código" para quien ya no aparece en la lista.
- **FR-004**: Elegir un nombre y escribir su código correcto, o escribir un código correcto en
  "entrar con mi código", MUST vincular la cuenta con ese invitado. La validación del código MUST
  ignorar espacios al inicio o al final y diferencias entre mayúsculas y minúsculas.
- **FR-005**: Una cuenta MUST vincularse con un solo invitado y un invitado con una sola cuenta. El
  sistema MUST rechazar cualquier vínculo que rompa esa regla, incluso si dos cuentas lo intentan
  al mismo tiempo.
- **FR-006**: La app MUST NOT ofrecer deshacer ni cambiar un vínculo; solo el organizador puede
  hacerlo, fuera de la app.
- **FR-007**: Tras vincular, el sistema MUST seguir según el estado del invitado: si está pendiente
  o "No asistirá", mostrar los datos del evento y preguntar si asiste; si está confirmado, entrar a
  la pantalla principal.
- **FR-008**: Al confirmar, el sistema MUST registrar la confirmación con su fecha y hora y retirar
  al invitado de la lista de pendientes. Al decir que no, MUST registrarlo como "No asistirá",
  retirarlo de la lista y no darle acceso a las pantallas internas.
- **FR-009**: Una cuenta vinculada MUST entrar sin código: si su invitado está confirmado, directo
  a la pantalla principal; si está "No asistirá", a la pregunta de asistencia (FR-007).
- **FR-010**: Toda pantalla y operación, salvo el inicio de sesión y el onboarding, MUST exigir una
  sesión válida de una cuenta vinculada a un invitado presente en el catálogo y confirmado. El
  sistema MUST rechazar las demás como "no autenticado" (sin sesión válida) o "no autorizado"
  (cuenta sin vincular, invitado fuera del catálogo o no confirmado).
- **FR-011**: La sesión MUST mantenerse en el dispositivo hasta que el usuario la cierre.
- **FR-012**: Un usuario con sesión MUST poder cerrarla desde la app.
- **FR-013**: Un invitado confirmado MUST poder confirmar a un invitado pendiente escribiendo el
  código de esa persona; MUST registrarse quién lo confirmó y MUST NOT vincular ninguna cuenta ni
  cambiar la sesión actual.
- **FR-014**: Tras 5 códigos incorrectos seguidos desde una misma cuenta (al vincular o al
  confirmar a otro), el sistema MUST bloquear nuevos intentos de esa cuenta durante 15 minutos y
  mostrar un mensaje amigable.
- **FR-015**: El catálogo de invitados y el evento MUST seguir saliendo de la base de datos que
  mantiene el organizador, con las reglas de la spec 001 (máximo 25 invitados, códigos únicos de 6
  o más caracteres); si faltan o son inválidos, la app MUST mostrar un mensaje amigable.
- **FR-016**: Ningún mensaje visible MUST contener detalles técnicos. Los errores del proveedor de
  autenticación (contraseña incorrecta, correo ya registrado, ventana cerrada, sin conexión) MUST
  traducirse a mensajes amigables en español.
- **FR-017**: El sistema MUST dejar de usar la sesión propia firmada por el servidor y el reingreso
  solo con código de la spec 001.
- **FR-018**: Los códigos de invitación MUST NOT salir del servidor ni guardarse en la cuenta del
  proveedor de autenticación.

### Key Entities

- **Cuenta**: la identidad del familiar en el proveedor de autenticación. Tiene un identificador
  único, un correo y el método con que entra (Google o correo).
- **Vínculo**: la relación uno a uno entre una cuenta y un invitado, con la fecha en que se creó.
  La crea el sistema al validar el código; solo el organizador la borra.
- **Invitado**: una persona del catálogo (nombre y código), como en la spec 001.
- **Respuesta de asistencia**: confirmado o "No asistirá", con fecha, hora y quién la registró, como
  en la spec 001.
- **Intentos de código**: los fallos seguidos de una cuenta y, si aplica, hasta cuándo está
  bloqueada.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un familiar completa el primer ingreso (iniciar sesión, elegir nombre, código y
  confirmación) en menos de 2 minutos.
- **SC-002**: Un familiar ya vinculado y confirmado llega a la pantalla principal en menos de 15
  segundos desde que abre la app, sin escribir ningún código.
- **SC-003**: El 100 % de las pantallas y operaciones internas probadas sin sesión, o con una
  cuenta sin vincular, terminan en el inicio de sesión, en el onboarding o en un rechazo.
- **SC-004**: Ninguna cuenta queda vinculada a más de un invitado y ningún invitado a más de una
  cuenta, incluso con intentos simultáneos.
- **SC-005**: Al menos 90 % de los familiares completan el primer ingreso sin pedir ayuda al
  organizador.
- **SC-006**: Ningún mensaje mostrado contiene códigos de error, trazas ni textos técnicos del
  proveedor.

## Assumptions

- El proveedor de autenticación es Firebase Authentication, en el mismo proyecto de la app, con los
  métodos Google y correo/contraseña, como en kaizen-app (constitución v3.0.0).
- No se exige verificar el correo: lo que prueba la identidad del familiar es su código de
  invitación, usado una sola vez para vincular.
- El restablecimiento de contraseña usa el correo estándar del proveedor.
- Hoy ningún invitado tiene respuestas ni sesiones registradas, así que no hay datos que migrar.
- Quien no tenga cuenta (por ejemplo, un niño) no necesita entrar: otro familiar puede confirmarlo
  (US6).
- Para deshacer un vínculo (cuenta perdida, vínculo equivocado), el organizador borra el registro
  del vínculo en la base de datos; la app no ofrece esa opción.
- El bloqueo por intentos se cuenta por cuenta, ya que todas las operaciones con código exigen
  sesión; el límite del proveedor para crear cuentas evita abusar creando muchas.
- El dominio `vamosjuntos.club` y el dominio por defecto del sitio quedan autorizados para iniciar
  sesión.
