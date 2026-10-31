# Research: Banner de Cuenta Regresiva

**Feature**: `004-banner-cuenta-regresiva` | **Date**: 2026-10-30

No quedan puntos `NEEDS CLARIFICATION`. La spec se escribió antes de la 008: donde dice "sesión
activa (spec 001)" se lee como la sesión de la 008, es decir, una cuenta de Firebase vinculada a un
invitado del catálogo y confirmado. Es la única cuenta que llega a la pantalla principal.

## R1. La hora confiable sale del backend

- **Decision**: el backend calcula la cuenta regresiva con su propio reloj y la entrega en una ruta
  nueva, `GET /api/countdown`, con `requireGuest` (🔒C). El frontend nunca lee `Date.now()` para
  decidir el número ni el tramo.
- **Rationale**:
  - FR-003 exige una referencia confiable de hora de Colombia y no el reloj del dispositivo. El
    reloj de Cloud Run está sincronizado por Google; el del celular puede estar mal a propósito o
    por descuido.
  - La fecha de inicio y la de fin ya viven en el backend (`events/{EVENT_ID}`, Principio I). Si el
    cálculo se hace donde están los datos y el reloj, el frontend solo pinta.
  - La ruta es independiente de la 003 (`GET /api/event`): se pueden publicar en cualquier orden.
- **Alternatives considered**:
  - **Leer el encabezado `Date` de cualquier respuesta y calcular en el frontend**: igual haría
    falta pedir las fechas del evento (que hoy solo da `GET /api/event` de la 003, todavía sin
    publicar), y la lógica de tramos quedaría en un repositorio sin pruebas automáticas.
  - **Devolver solo `serverNow` y calcular el desfase en el frontend**: mueve la lógica de tramos a
    TypeScript, donde solo hay chequeo de tipos. En Go se prueba con una tabla (Principio IV).
  - **Una API pública de hora (worldtimeapi, etc.)**: dependencia externa sin necesidad; el backend
    ya es una fuente confiable.
  - **Escribir las fechas del evento en el frontend**: viola el Principio I (el evento se lee de
    Firestore).

## R2. Días y tramos con una función pura

- **Decision**: `countdownAt(e Event, now time.Time) Countdown` en el backend, sin I/O.
  - **Días restantes** (FR-002): diferencia de **días calendario** entre la fecha de `now` y la de
    `e.StartsAt`, ambas convertidas a `colombia` (UTC-5 fijo, ya definido en `event.go`). No es
    `(StartsAt − now) / 24 h`: el 22 de diciembre a las 11:59 p. m. falta 1 día aunque falten 14
    horas y pico.
  - **Tramos** (FR-007), en este orden:

    | Condición | Tramo |
    |-----------|-------|
    | `now ≥ EndsAt` | `finished` |
    | `now ≥ StartsAt` | `ongoing` |
    | días = 0 | `today` |
    | días = 1 | `tomorrow` |
    | días 2 a 7 | `lastWeek` |
    | días 8 a 30 | `near` |
    | días > 30 | `far` |

- **Rationale**: Principio IV (lógica de negocio pura y testeable sin Firestore). SC-002 y SC-003
  piden el resultado correcto en transiciones de medianoche y en cada tramo; una tabla de `go test`
  los cubre todos con instantes exactos.
- **Verificación con la spec**: el 2026-10-30 en Colombia faltan 1 (31 oct) + 30 (nov) + 23 (dic) =
  **54** días (US1, escenario 1).
- **Alternatives considered**: **contar horas y redondear**: falla justo en las medianoches, que es
  lo que SC-002 prueba.

## R3. Actualizar al cambiar el día sin recargar (FR-009)

- **Decision**:
  - La respuesta trae `secondsToNextChange`: segundos hasta el próximo instante en que cambia lo que
    muestra el banner. Es el menor entre la próxima medianoche de Colombia, `StartsAt` y `EndsAt`
    que sea posterior a `now`, redondeado hacia arriba. No viene cuando el tramo es `finished`,
    porque ya no cambia.
  - El frontend programa un `setTimeout` con ese valor y vuelve a pedir `/api/countdown`.
  - También vuelve a pedirla cuando la pestaña pasa a visible (`visibilitychange`).
  - Si una de esas recargas falla, el banner conserva lo último que mostró.
- **Rationale**:
  - Un valor relativo no depende del reloj del dispositivo: `setTimeout` mide tiempo transcurrido,
    no la hora del sistema. Un celular con la hora mal puesta igual se actualiza a la medianoche de
    Colombia.
  - Los navegadores móviles congelan o retrasan los temporizadores con la pantalla apagada; pedir de
    nuevo al volver a la pestaña cubre ese caso. Es parte de cumplir FR-009, no una función extra.
  - El valor máximo es 24 h (`86 400 s`), muy por debajo del límite de `setTimeout` (~24,8 días).
- **Alternatives considered**:
  - **Consultar cada hora**: el número podría quedar hasta una hora desactualizado después de la
    medianoche y hace 24 llamadas al día por invitado sin necesidad.
  - **Devolver `nextChangeAt` absoluto**: obligaría al frontend a compararlo con su propio reloj, que
    es justo lo que FR-003 prohíbe.

## R4. El banner es lo primero de la pantalla

- **Decision**:
  - El banner reemplaza el recuadro punteado que la 001 dejó en `HomePage` como lugar reservado.
  - El saludo con el nombre pasa del encabezado de `Screen` al banner: `Screen` acepta `title`
    opcional y `HomePage` deja de pasarlo. El `<h1>` de la página es el mensaje del banner, por
    ejemplo "Carmencita, faltan 54 días para el paseo".
  - La línea de marca "Vamos Juntos" de `Screen` se queda arriba, pequeña.
- **Rationale**:
  - FR-001 y US1 piden que el banner sea lo primero que se ve y que salude por el nombre. Con el
    encabezado actual ("¡Hola, Carmencita!") arriba, el nombre saldría dos veces y el banner sería el
    segundo elemento.
  - La página conserva un solo `<h1>`, ahora con el contenido principal.
- **Alternatives considered**:
  - **Dejar el encabezado y poner el banner debajo**: incumple "primer elemento visible" y repite el
    nombre.
  - **Un `Screen` especial para la pantalla principal**: duplica el layout por una línea.

## R5. Diseño expresivo, legible y con movimiento reducido

- **Decision**:
  - **Número protagonista** (FR-005): `text-8xl` (96 px) y `font-black`. El título más grande de la
    app es `text-3xl`, y la card de la 003 usa tamaños menores.
  - **Intensidad por tramo** (US2): cada tramo tiene su gradiente, su emoji y su mensaje. La
    intensidad sube de `far` a `lastWeek`; los tramos sin número (`today`, `ongoing`, `finished`)
    usan el mensaje como texto grande. La tabla está en [data-model.md](data-model.md).
  - **Contraste a pleno sol** (FR-010): texto blanco sobre fondos de tono 700 o más oscuro de
    Tailwind (`orange-700`, `rose-700`, `fuchsia-700`, `red-700`, `emerald-700`, `sky-700`, `violet-700`).
    Todos superan 4,5:1 con blanco, el mínimo de WCAG AA para texto normal; el número y el mensaje,
    que son texto grande, quedan muy por encima del 3:1 que se exige para ese tamaño.
  - **Animación de entrada** (FR-006): un `@keyframes` propio declarado en `@theme` de
    `src/index.css` (escala y opacidad, ~0,7 s). En `lastWeek` el emoji además rebota
    (`animate-bounce`).
  - **Reducir movimiento** (US1, escenario 3): toda clase de animación va con la variante
    `motion-safe:`. Con `prefers-reduced-motion: reduce` el banner aparece quieto y con el mismo
    contenido. No hace falta JavaScript.
  - **Nombres largos**: `break-words` y `text-balance`, como el título actual de `Screen`. Se prueba
    con "Nicolás López" y "mamá de lore" a 360 px de ancho.
- **Rationale**: todo sale de Tailwind y de CSS estándar, sin librerías de animación ni
  ilustraciones (Principio III). Los emojis cumplen "emojis o ilustración" sin assets nuevos.
- **Alternatives considered**:
  - **Una librería de animación (Framer Motion, confetti)**: una dependencia para una sola entrada
    que CSS ya resuelve.
  - **Gradientes en tono 500**: más vivos, pero el blanco sobre `orange-500` queda cerca de 3:1 y no
    se lee bien al sol.

## R6. Textos, singular y plural

- **Decision**:
  - Los textos viven en el frontend, en el componente del banner.
  - Plural (FR-008): `días === 1 ? 'falta 1 día' : \`faltan ${días} días\``. Solo `tomorrow` puede
    tener 1 día; los demás tramos con número tienen 2 o más.
  - Mensajes por tramo: los ejemplos de la spec. La spec permite ajustarlos si se respetan los
    tramos.
- **Rationale**: el backend solo entrega datos (`phase`, `daysLeft`), igual que en la 003, donde
  los textos también los arma el frontend.
- **Alternatives considered**: **mandar el texto desde Go**: mezcla presentación con la API y obliga
  a desplegar el backend para cambiar una frase.

## R7. Carga y errores dentro del banner

- **Decision**:
  - **Cargando**: el banner aparece con "¡Hola, {nombre}!" y "Contando los días…". El nombre ya
    está en `/api/me`, así que no espera a la nueva ruta.
  - **No disponible** (`404 event_not_found`, `500` o sin red en la primera carga): el banner muestra
    "¡Hola, {nombre}!" y "La cuenta regresiva no está disponible por ahora.".
  - **`403`**: ningún texto propio; el manejador global vuelve a pedir `/api/me` (008).
  - **`401`**: ningún texto propio; el cliente renueva el token o cierra la sesión (008).
- **Rationale**: Principio V (mensaje amigable, sin códigos). Es el mismo criterio que la card de la
  003. La spec no define un estado de error; este es el mínimo para no mostrar un banner vacío ni un
  error crudo. No hay botón de reintentar: la siguiente visita o el regreso a la pestaña reintentan
  solos (R3).
- **Alternatives considered**: **ocultar el banner si falla**: el invitado no sabría por qué falta lo
  más llamativo de la pantalla.

## R8. Reloj inyectable en el backend

- **Decision**: `server` gana un campo `now func() time.Time`. `NewServer` lo inicializa con
  `time.Now`, así que su firma no cambia. Las pruebas del handler, en el mismo paquete, lo
  reemplazan por una función con una fecha fija.
- **Rationale**:
  - Principio IV: inyección de dependencias sin estado global mutable.
  - No cambiar la firma de `NewServer` evita tocar `main.go` y `newTestEnv`, que comparten las
    sesiones de la 003 y la 005.
- **Alternatives considered**:
  - **Variable de paquete `var now = time.Now`**: es estado global mutable.
  - **Un parámetro más en `NewServer`**: obliga a cambiar `main.go` y todas las pruebas por algo
    que en producción siempre vale `time.Now`.
  - **Una variable de entorno para simular la fecha**: sería código sombra en producción. Las fechas
    se simulan en las pruebas y, para revisar el diseño, con la anulación de respuestas de las
    DevTools ([quickstart.md](quickstart.md)).
