# Research: Onboarding y Sesión

**Feature**: `001-onboarding-sesion` | **Date**: 2026-10-30

Cada decisión resuelve una incógnita del Technical Context de [plan.md](plan.md).

## R1. Mecanismo de sesión

- **Decision**: cookie `__session` con un token firmado sin estado:
  `base64url(payload) + "." + base64url(HMAC-SHA256(payload))`, donde el payload es
  `{"g":"<guestId>","exp":<unix>}`. La firma usa `SESSION_KEY` (32 bytes, en Secret Manager).
  La cookie va con `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/` y `Expires` igual a `exp`.
- **Rationale**: Firebase Hosting solo reenvía a Cloud Run la cookie llamada `__session`
  (constitución, sección Stack). Un token firmado con la biblioteca estándar (`crypto/hmac`) evita
  una colección de sesiones y su limpieza. El cierre de sesión (FR-012) solo exige que termine en
  el dispositivo: basta con borrar la cookie. La revocación efectiva viene de FR-008, porque cada
  request vuelve a verificar que el invitado siga en el catálogo y confirmado.
- **Alternatives considered**:
  - Sesiones opacas guardadas en Firestore: permiten revocar en el servidor, pero suman una
    colección, una lectura extra por request y limpieza de vencidas. No hace falta en la V1.
  - JWT con una biblioteca externa: es el mismo resultado con una dependencia más.
  - Token en `localStorage` con encabezado `Authorization`: expuesto a XSS y no aprovecha que
    front y back comparten dominio.

## R2. Vencimiento de la sesión

- **Decision**: `exp = event.endsAt + 7 días` (2027-01-02 11:00 hora Colombia), calculado al
  crear la sesión.
- **Rationale**: es exactamente FR-009, y sale del dato del evento en vez de quedar fijo en el
  código.
- **Alternatives considered**: una duración fija (30 días) obligaría a reingresar antes del paseo.

## R3. Catálogo de invitados e identificador estable

> **Reemplazada por R12** (enmienda de la constitución v2.0.0): el catálogo ya no viene de
> `GUESTS_JSON` sino de Firestore, y el `guestId` es el id del documento. Se conserva el texto
> original como historia.

- **Decision**: la variable de entorno `GUESTS_JSON` contiene `[["Nombre","código"], ...]`. Al
  arrancar, el backend la parsea y la valida: máximo 25 invitados, códigos únicos sin distinguir
  mayúsculas y minúsculas tras recortar espacios, mínimo 6 caracteres e IDs únicos. Si algo falla,
  `main` registra el error y sale con código distinto de cero (sin `panic`). El `guestId` es un
  *slug* del nombre: minúsculas, sin tildes ni eñes (`strings.Replacer` de la biblioteca estándar)
  y espacios convertidos en `-`. Ejemplos: "papá de lore" → `papa-de-lore`,
  "Nicolás López" → `nicolas-lopez`.
- **Rationale**: lo exige la constitución (catálogo en variables de entorno, códigos fuera del
  repo) junto con FR-013 y FR-016. Si el catálogo es inválido, Cloud Run no da por sana la nueva
  revisión y la anterior sigue sirviendo. El slug no cambia aunque se reordene la lista.
- **Alternatives considered**:
  - El índice en la lista como ID: cambia si se reordena el catálogo.
  - `golang.org/x/text` para quitar tildes: es una dependencia para un problema de 12 caracteres.

## R4. Modelo en Firestore y respuestas concurrentes

- **Decision**: `events/{EVENT_ID}` lo crea el organizador a mano y se lee al arrancar cada
  operación. Las respuestas viven en `events/{EVENT_ID}/attendance/{guestId}`; si el documento no
  existe, el invitado está pendiente. Cada cambio de estado corre en una transacción de Firestore
  (`RunTransaction`): lee el documento, aplica la regla pura de transición (ver
  [data-model.md](data-model.md)) y escribe.
- **Rationale**: son ≤ 25 documentos y no requieren índices. La transacción garantiza que la
  confirmación se registre una sola vez cuando hay dos dispositivos simultáneos (Edge Cases). La
  regla de transición es una función pura, testeable sin Firestore (Principio IV).
- **Alternatives considered**:
  - Guardar el estado en el arreglo del documento del evento: genera contención en un solo
    documento y escrituras más grandes.
  - `Create` sin transacción: no cubre el cambio de "No asistirá" a "confirmado".

## R5. Límite de intentos fallidos por dispositivo (FR-010)

- **Decision**: el frontend genera un `deviceId` (`crypto.randomUUID()`), lo guarda en
  `localStorage` y lo envía en el encabezado `X-Device-Id` en las operaciones con código. El
  backend guarda `loginAttempts/{deviceId}` con `failures` y `blockedUntil`: con 5 fallos seguidos
  bloquea 15 minutos, y un acierto reinicia el contador. Si falta el encabezado, la respuesta es
  `400`.
- **Rationale**: Firebase Hosting no deja usar una segunda cookie para identificar el dispositivo,
  y limitar por IP bloquearía a toda una familia que comparte el wifi de la casa. El `deviceId`
  puede falsificarse, pero eso no es grave: con códigos aleatorios de 6 o más caracteres
  alfanuméricos (36⁶ ≈ 2 × 10⁹ combinaciones) y 25 códigos válidos, adivinar por fuerza bruta no
  es viable. El límite existe para la experiencia que pide el spec, no como única barrera.
- **Alternatives considered**:
  - Un contador en memoria: se pierde cuando Cloud Run escala a cero o levanta otra instancia.
  - Limitar por IP: castiga a familias detrás de la misma red.

## R6. API HTTP del backend

- **Decision**: `net/http` con los patrones de `ServeMux` de Go 1.22+ (`POST /api/session/reentry`)
  y JSON con `encoding/json`. Sin framework. Los errores tienen la forma
  `{"error":{"code":"<código>"}}` y el frontend traduce cada `code` a un mensaje amigable.
- **Rationale**: la constitución pide "Golang puro y duro" y estructura plana. Traducir en la UI
  cumple el Principio V sin acoplar textos al backend.
- **Alternatives considered**: Gin o Echo, que no aportan nada para 7 endpoints.

## R7. Códigos HTTP por caso

- **Decision**:
  - `400`: petición inválida, o código incorrecto al confirmar a otro invitado.
  - `401`: sin sesión, o código incorrecto en endpoints públicos de autenticación.
  - `403`: sesión de alguien que ya no está invitado o confirmado.
  - `404`: el evento no existe.
  - `409`: estado incompatible (ya respondió, todavía pendiente).
  - `429`: bloqueado por intentos.
- **Rationale**: el Principio V fija 400, 401 y 409 como mínimo. Confirmar a otro usa `400` y no
  `401` porque el que llama sí está autenticado; además, el frontend interpreta cualquier `401`
  en rutas protegidas como "volver al onboarding", y no debe sacar al usuario de su sesión por
  equivocarse en el código de otra persona.

## R8. Zona horaria

- **Decision**: las fechas se guardan como `Timestamp` de Firestore (UTC) y la API las devuelve en
  RFC 3339 con offset `-05:00`. El binario importa `time/tzdata` para que
  `time.LoadLocation("America/Bogota")` funcione en una imagen mínima sin zoneinfo. El frontend
  formatea con `Intl.DateTimeFormat('es-CO', { timeZone: 'America/Bogota' })`.
- **Rationale**: lo fija la constitución (sección Tiempo). Así la presentación no depende de la
  zona horaria del dispositivo.

> **Ajuste en la implementación:** se usa `time.FixedZone("COT", -5h)` en lugar de
> `time/tzdata`. Colombia no tiene horario de verano, así que el resultado es idéntico y el
> binario no depende de la base de zonas horarias.

## R9. Frontend

- **Decision**:
  - React 19 + TypeScript + Vite, con Tailwind CSS v4 mediante su plugin de Vite.
  - React Router 7 en modo librería, para rutas y la guarda de sesión.
  - `fetch` nativo envuelto en un cliente mínimo.
  - El estado de sesión vive en un Context con el hook `useSession`.
  - En desarrollo, el proxy de Vite manda `/api` a `localhost:8080`, igual que el rewrite de
    Firebase Hosting en producción.
- **Rationale**: es el stack de la constitución con lo mínimo necesario. El router se justifica
  porque las specs 003 a 007 agregan pantallas detrás de la misma guarda.
- **Alternatives considered**:
  - TanStack Query o Redux: no hacen falta para 6 llamadas.
  - Next.js: requiere un servidor y no encaja con un hosting estático.

## R10. Pruebas

- **Decision**: en el backend, `go test` con la biblioteca estándar `testing`. Se prueban como
  funciones puras: catálogo, slug, comparación de códigos, transiciones de asistencia, firma y
  verificación del token y el límite de intentos. Los handlers se prueban con `httptest` y fakes
  de las interfaces pequeñas. En el frontend, en esta feature solo el chequeo de tipos del build;
  la validación de punta a punta es manual, siguiendo [quickstart.md](quickstart.md).
- **Rationale**: la constitución pide lógica pura testeable, pero no exige TDD ni pruebas de UI.
  Agregar Vitest y Testing Library ahora sería código "por si acaso" (Principio VI).

## R11. Desarrollo local

- **Decision**: emulador de Firestore (`gcloud emulators firestore start`), con
  `FIRESTORE_EMULATOR_HOST` para el backend. El evento se siembra con una llamada `curl` a la
  API REST del emulador, documentada en el quickstart; el binario no tiene código para crear
  eventos (Principio I). Se usa un `GUESTS_JSON` local con códigos de prueba, nunca los
  reales.
- **Rationale**: permite probar todo el flujo sin tocar producción ni versionar secretos.

## R12. Catálogo en Firestore (enmienda de la constitución v2.0.0)

- **Decision**: el catálogo vive en `events/{EVENT_ID}/users`, un documento por invitado con los
  campos `Nombre` y `código`, tal como los creó el organizador en la consola.
  - El `guestId` es el id del documento, generado por Firestore. Ya no hay *slug*.
  - Un `GuestStore` lee la subcolección completa (≤ 25 documentos) en cada request que usa el
    catálogo y la pasa por la misma validación pura de antes (`newCatalog`): 1 a 25 invitados,
    nombres no vacíos, códigos de 6 o más caracteres y únicos sin distinguir mayúsculas.
  - `main` también la lee y valida al arrancar: si es inválida, sale con error y Cloud Run
    mantiene la revisión anterior.
  - Con el servicio en marcha, un catálogo inválido hace responder `500 internal_error` y el
    motivo queda en el log, sin códigos.
  - Otros campos del documento (como `confirmado`) se ignoran, porque la asistencia vive en
    `attendance`.
- **Rationale**: es el pedido del responsable. Así puede agregar o corregir invitados en la consola
  de Firestore sin volver a desplegar, y el catálogo deja de pasar por GitHub y por la
  configuración de Cloud Run.
  - Leer en cada request es lo más simple y siempre está al día. Cuesta unas 25 lecturas por
    request, unos centavos durante todo el paseo.
  - El id del documento no cambia si se corrige un nombre, así que no rompe sesiones ni
    respuestas guardadas.
  - Las reglas de Firestore de la base ya niegan todo acceso desde clientes (verificado: una
    lectura sin credenciales responde 403), así que solo el backend puede leer los códigos.
- **Alternatives considered**:
  - Leer solo al arrancar: un invitado nuevo no aparecería hasta un nuevo despliegue.
  - Caché con vencimiento: menos lecturas, pero agrega estado compartido y un mutex sin
    necesidad real.
  - Usar `confirmado` del documento como estado de asistencia: no puede representar "No
    asistirá" ni quién respondió, y duplicaría `attendance`.
  - Guardar el código con hash: los códigos son cortos y el organizador necesita leerlos para
    entregarlos.
