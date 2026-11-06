# Research: Confirmar a otro invitado sin su código

**Feature**: `009-confirmar-otro-sin-codigo` | **Date**: 2026-11-06

No hay puntos `NEEDS CLARIFICATION`: la spec quedó aclarada con el responsable el 2026-11-06.

## R1. Lista propia para "Confirmar a otro invitado"

- **Decision**: una ruta nueva `GET /api/guests/confirmable` (🔒C) con los invitados pendientes y
  sin vínculo. `GET /api/onboarding/guests` no cambia.
- **Rationale**:
  - la lista del onboarding debe seguir mostrando a los pendientes ya vinculados: si esa persona
    entra con una segunda cuenta (Google y correo), elige su nombre y recibe el mensaje de
    `guest_already_linked` que la remite al organizador (008, Edge Cases);
  - esa lista la ve cualquier cuenta autenticada, incluso sin vincular. Marcar ahí quién ya tiene
    cuenta se lo contaría a cualquiera que cree una cuenta. La ruta nueva exige una cuenta
    vinculada y confirmada.
- **Alternatives considered**: un campo `linked` en `/api/onboarding/guests` (filtra información a
  cuentas sin vincular); un parámetro de consulta en la misma ruta (dos contratos en una).

## R2. Validar "pendiente y sin vínculo" en una sola transacción

- **Decision**: `AttendanceStore.ConfirmOther(ctx, guestID, by)` corre una transacción que lee
  `links/{guestId}` y `attendance/{guestId}`, aplica la regla pura `confirmOther(current, linked)`
  y escribe la respuesta.
- **Rationale**: FR-004 exige validar al registrar, incluso si la persona vincula su cuenta en ese
  instante. `LinkStore.Link` lee y crea `links/{guestId}` en su propia transacción, así que
  Firestore ordena las dos transacciones sobre ese documento: o la confirmación entra antes del
  vínculo, o ve el vínculo y se rechaza. Nunca queda confirmada por otro una persona ya vinculada.
- **Alternatives considered**: consultar `links` en el handler antes de `Respond`. Es más corto,
  pero deja una ventana entre la consulta y la escritura. Guardar una marca de "tiene cuenta" en
  `attendance` duplica el dato del vínculo.

## R3. Regla pura y orden de los errores

| Respuesta actual | ¿Vinculado? | Resultado |
|------------------|-------------|-----------|
| pendiente | no | `confirmed` |
| pendiente | sí | `errGuestLinked` → `409 guest_already_linked` |
| confirmado o "No asistirá" | da igual | `errAlreadyResponded` → `409 already_responded` |

Primero se mira la respuesta y luego el vínculo. Quien tiene cuenta casi siempre ya respondió, y
"ya respondió" es el mensaje más claro en ese caso. `confirmOther` reutiliza `nextStatus(current,
actionConfirmOther)` de la 001 para la primera parte.

## R4. El cuerpo de `POST /api/guests/{guestId}/confirmation` se ignora

- **Decision**: el handler ya no lee el cuerpo. No compara códigos ni llama a `checkCode`, así que
  no cuenta intentos (FR-007).
- **Rationale**: el backend se publica antes que el frontend. Durante esos minutos, y mientras
  algún celular tenga la versión anterior en caché, la app vieja sigue mandando `{"code": "..."}`.
  Ignorar el cuerpo la mantiene funcionando con las reglas nuevas.
- **Alternatives considered**: rechazar un cuerpo con `code` (`400`) rompería la app vieja entre
  los dos despliegues.

## R5. Códigos de error

- **Decision**: "ya tiene cuenta" reutiliza `409 guest_already_linked`. La UI, en el contexto de
  confirmar a otro, lo traduce a "{nombre} ya entró con su propia cuenta y responde desde ahí.".
- **Rationale**: es el mismo hecho (ese invitado ya tiene un vínculo) y es un conflicto, como manda
  el Principio V. Un código nuevo no aporta nada.
- **Salen de esta ruta**: `400 invalid_request`, `400 invalid_code` y `429 too_many_attempts`.
  También sale de la UI el mensaje "Ese código no corresponde a {nombre}.".

## R6. Pregunta previa en la misma pantalla

- **Decision**: al tocar un nombre, la misma pantalla cambia a "¿Confirmar a {nombre}?" con "Sí,
  confirmar a {nombre}" y "Cancelar", como "¿Eliminar este ítem?" de Compras (007). Sin modales ni
  librerías.
- **Rationale**: cumple FR-003 con el patrón que la familia ya conoce. Ante `404` o `409`, la
  pantalla muestra el mensaje y recarga la lista (FR-008).
