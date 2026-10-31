---

description: "Lista de tareas de la feature 004-banner-cuenta-regresiva"
---

# Tasks: Banner de Cuenta Regresiva

**Input**: Design documents from `/specs/004-banner-cuenta-regresiva/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/api.md](contracts/api.md), [quickstart.md](quickstart.md)

**Tests**: la spec no pide TDD. Igual que en la 003 y la 008, se escriben pruebas de Go junto al
código para la función pura y el handler, con los fakes de `handlers_test.go`. En el frontend, el
build con chequeo de tipos y `oxlint`. De punta a punta, el quickstart.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2)
- Include exact file paths in descriptions

## Path Conventions

Las rutas son relativas a `C:\golang\vamosjuntos-app\`. Mientras la 003 ocupe los working trees
principales, el código de esta feature se escribe en los worktrees `vamosjuntos-backend-004` y
`vamosjuntos-frontend-004` (T001). Las rutas `vamosjuntos-backend/…` y `vamosjuntos-frontend/…` de
abajo se refieren a esos worktrees.

Reglas que aplican a todas las tareas:

- **Go**: paquete `main` plano, `context.Context` en el I/O, errores envueltos y sin estado global
  mutable.
- **Hora**: el frontend nunca usa `Date.now()` ni `new Date()` para el número ni el tramo (FR-003,
  research R1 y R3).
- **Movimiento**: toda clase de animación lleva la variante `motion-safe:` (FR-006).
- **Contraste**: texto blanco solo sobre fondos de tono 700 o más oscuro (FR-010, research R5).
- **Archivos compartidos con la 003 y la 005**: en `handlers.go`, `HomePage.tsx` y `types.ts`, solo
  los cambios que pide cada tarea.
- **Etiquetas**: **(visto bueno)** publica en producción, así que se hace solo con aprobación
  explícita.

---

## Phase 1: Setup (Shared Infrastructure)

- [X] T001 Crear los worktrees con la rama `004-banner-cuenta-regresiva` desde `main`, sin cambiar
  de rama los working trees principales:
  - `git -C vamosjuntos-backend worktree add ../vamosjuntos-backend-004 -b 004-banner-cuenta-regresiva main`;
  - `git -C vamosjuntos-frontend worktree add ../vamosjuntos-frontend-004 -b 004-banner-cuenta-regresiva main`;
  - en `vamosjuntos-frontend-004`, correr `npm ci` y copiar `.env.local` desde `vamosjuntos-frontend`.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: la ruta `GET /api/countdown` completa. Las tres historias la consumen; el tramo se
calcula en una sola función pura que se prueba de una vez.

- [X] T002 [P] Crear `vamosjuntos-backend/countdown.go` con:
  - constantes de tramo `phaseFar = "far"`, `phaseNear = "near"`, `phaseLastWeek = "lastWeek"`,
    `phaseTomorrow = "tomorrow"`, `phaseToday = "today"`, `phaseOngoing = "ongoing"` y
    `phaseFinished = "finished"`;
  - `Countdown` con `Phase string \`json:"phase"\``, `DaysLeft int \`json:"daysLeft"\`` y
    `SecondsToNextChange int \`json:"secondsToNextChange,omitempty"\``;
  - la función pura `countdownAt(e Event, now time.Time) Countdown` con las reglas de
    data-model.md:
    - tramos en este orden: `now ≥ EndsAt` → `finished`; `now ≥ StartsAt` → `ongoing`; días = 0 →
      `today`; días = 1 → `tomorrow`; días 2 a 7 → `lastWeek`; días 8 a 30 → `near`; días > 30 →
      `far`;
    - días = "días calendario de Colombia entre `now` y `StartsAt`; `0` en `today`, `ongoing` y
      `finished`": comparar las fechas (año, mes, día) de `now.In(colombia)` y
      `e.StartsAt.In(colombia)`, no dividir la duración entre 24 h (research R2);
    - `SecondsToNextChange` = "segundos, redondeados hacia arriba, hasta la próxima medianoche de
      Colombia, `StartsAt` o `EndsAt`, el que llegue primero; ausente en `finished`". Solo cuentan
      los instantes posteriores a `now`;
  - un comentario que apunte a `specs/004-banner-cuenta-regresiva/data-model.md`.
- [X] T003 [P] Crear `vamosjuntos-backend/countdown_test.go` con una prueba de tabla de
  `countdownAt` usando el evento de `newTestEnv` (2026-12-23 14:00 a 2026-12-26 11:00, `colombia`).
  Debe cubrir todas las filas de "Ejemplos" de data-model.md:
  - 2026-10-30 10:00 → `far`, 54, 50 400;
  - 2026-11-22 23:59:59 → `far`, 31, 1, y 2026-11-23 00:00 → `near`, 30, 86 400;
  - 2026-12-15 12:00 → `near`, 8; 2026-12-16 00:00 → `lastWeek`, 7;
  - 2026-12-22 08:00 → `tomorrow`, 1, 57 600;
  - 2026-12-23 00:00 → `today`, 0, 50 400 (hasta `StartsAt`);
  - 2026-12-23 14:00 exacto → `ongoing`, 0, 36 000;
  - 2026-12-26 10:59:59 → `ongoing`, 1 s;
  - 2026-12-26 11:00 exacto → `finished`, 0, sin `SecondsToNextChange`.

  Además, el mismo instante de 2026-10-30 expresado en `Europe/Madrid` (o una `time.FixedZone`
  de +1) da el mismo resultado (SC-002).
- [X] T004 En `vamosjuntos-backend/handlers.go`:
  - agregar el campo `now func() time.Time` a `server` e inicializarlo con `time.Now` dentro de
    `NewServer`, sin cambiar su firma (research R8);
  - registrar `mux.HandleFunc("GET /api/countdown", s.requireGuest(s.handleCountdown))` en
    `routes()`.
- [X] T005 En `vamosjuntos-backend/countdown.go`, agregar
  `handleCountdown(w, r, _ Account, _ Guest)`: usa `s.loadEvent` (que ya responde
  `404 event_not_found` y `500 internal`) y responde `200` con `countdownAt(event, s.now())`.
  Depende de T002 y T004.
- [X] T006 En `vamosjuntos-backend/countdown_test.go`, pruebas del handler. Se arma con
  `newTestEnv`, se crea otro `server` con `NewServer(env.guests, env.events, env.attendance,
  env.attempts, testAccounts, env.links)`, se fija `srv.now` en 2026-10-30 10:00 de Colombia y se
  asigna `env.handler = srv.routes()`:
  - invitado vinculado y confirmado → `200` con `{"phase":"far","daysLeft":54,"secondsToNextChange":50400}`
    y `Cache-Control: no-store`;
  - evento ausente (`env.events.err = errEventNotFound`) → `404 event_not_found`;
  - cuenta vinculada pero pendiente → `403 not_invited`;
  - cuenta sin vincular → `403 not_linked`;
  - sin token → `401 unauthenticated`.
- [X] T007 [P] En `vamosjuntos-frontend/src/api/types.ts`, agregar justo después de
  `EventSummary` (no al final del archivo, donde agrega la 003):
  - `export type CountdownPhase = 'far' | 'near' | 'lastWeek' | 'tomorrow' | 'today' | 'ongoing' | 'finished'`;
  - `export interface Countdown { phase: CountdownPhase; daysLeft: number; secondsToNextChange?: number }`;
  - un comentario que apunte a `specs/004-banner-cuenta-regresiva/contracts/api.md`.

**Checkpoint**: `go test ./...` pasa y `GET /api/countdown` responde según el contrato.

---

## Phase 3: User Story 1 - Ver cuántos días faltan (Priority: P1) 🎯 MVP

**Goal**: un invitado confirmado abre la pantalla principal y lo primero que ve es un banner grande,
colorido y animado con su nombre y el número de días que faltan.

**Independent Test**: el 2026-10-30, el banner es lo primero de la pantalla, dice "{nombre}, faltan
54 días para el paseo" con "54" como el texto más grande, entra con animación y aparece quieto con
"reducir movimiento" (quickstart, pasos 1–6, 8–11).

- [X] T008 [P] [US1] En `vamosjuntos-frontend/src/index.css`, después de `@import "tailwindcss";`,
  agregar en un bloque `@theme` la variable `--animate-banner-in: banner-in 0.7s cubic-bezier(0.2, 0.8, 0.2, 1.2) both;`
  y su `@keyframes banner-in` (de `opacity: 0; transform: scale(0.85) translateY(12px)` a
  `opacity: 1; transform: none`) (research R5).
- [X] T009 [P] [US1] En `vamosjuntos-frontend/src/components/Screen.tsx`, volver `title` opcional
  (`title?: string`) y renderizar el `<h1>` solo cuando viene. El resto del layout no cambia
  (research R4).
- [X] T010 [US1] Crear `vamosjuntos-frontend/src/components/CountdownBanner.tsx`, componente
  funcional `CountdownBanner({ name }: { name: string })`:
  - **Carga**: pide `apiFetch<Countdown>('/api/countdown')` al montarse; ignora la respuesta si el
    componente ya se desmontó.
  - **Estados** (data-model.md, "Estados del banner"):
    - `loading`: "¡Hola, {nombre}!" y "Contando los días…";
    - `ready`;
    - `unavailable`: "¡Hola, {nombre}!" y "La cuenta regresiva no está disponible por ahora." Se usa
      solo si falla la primera carga.
  - **Actualización sin recargar** (FR-009, research R3):
    - si la respuesta trae `secondsToNextChange`, programar un `setTimeout` de
      `secondsToNextChange * 1000` ms que vuelva a pedir la ruta;
    - escuchar `visibilitychange` y volver a pedirla cuando `document.visibilityState === 'visible'`;
    - limpiar el temporizador y el listener al desmontar;
    - si una de estas recargas falla, conservar el `ready` anterior.
  - **Nunca** usar `Date.now()` ni `new Date()`.
  - **Vista `ready`** con tramo `far`. Las US2 y US3 completan los demás tramos.
    - `<section>` con `rounded-3xl`, gradiente `bg-linear-to-br from-orange-700 to-red-700`,
      texto blanco, `shadow-xl`, padding generoso y `motion-safe:animate-banner-in`.
    - Emoji 🎒 grande y el mensaje "¡Ya empieza la cuenta regresiva!".
    - Un `<h1>` con `text-balance break-words` que contiene "{nombre}, faltan", el número en un
      `<span className="block text-8xl font-black">` y "días para el paseo". Así el número es el
      texto más grande de la pantalla (FR-005) y un nombre largo como "Nicolás López" no rompe el
      diseño.
  - **`403` y `401`**: no muestran texto propio; los maneja el cliente global (contracts/api.md,
    "Mensajes en la UI").
- [X] T011 [US1] En `vamosjuntos-frontend/src/pages/HomePage.tsx`:
  - reemplazar el `<section>` del recuadro punteado y su comentario por
    `<CountdownBanner name={name} />` como primer hijo de `Screen`;
  - quitar `title` de `<Screen>`, porque el saludo con el nombre ahora está en el banner
    (research R4);
  - importar `CountdownBanner`.

  No tocar el resto de la página (la 003 agrega `TripCard` debajo).

**Checkpoint**: el banner con el número de días funciona y se puede publicar solo.

---

## Phase 4: User Story 2 - El mensaje cambia a medida que se acerca el paseo (Priority: P2)

**Goal**: el mensaje, el emoji y la intensidad cambian en los tramos `far`, `near`, `lastWeek` y
`tomorrow`, con singular y plural correctos.

**Independent Test**: anulando la respuesta de `/api/countdown` con cada uno de esos tramos, el
banner muestra el mensaje y el fondo de data-model.md, y `tomorrow` dice "falta 1 día" (quickstart,
paso 7).

- [X] T012 [US2] En `vamosjuntos-frontend/src/components/CountdownBanner.tsx`, agregar una tabla
  `Record<CountdownPhase, …>` con mensaje, emoji y clases de gradiente por tramo, tal como la tabla
  "Tramos" de data-model.md:
  - `far`: "¡Ya empieza la cuenta regresiva!", 🎒, `from-orange-700 to-red-700`;
  - `near`: "¡Ve alistando la maleta!", 🧳, `from-orange-700 to-rose-700`;
  - `lastWeek`: "¡Ya casi! Última semana", 🔥 con `motion-safe:animate-bounce`,
    `from-rose-700 to-fuchsia-700`, más una sombra y un anillo más marcados (máxima intensidad,
    US2 escenario 3);
  - `tomorrow`: "¡Mañana nos vamos!", 🚌, `from-red-700 to-fuchsia-700`.

  Las clases van completas en el código, nunca armadas con plantillas, para que Tailwind las
  genere.
- [X] T013 [US2] En `vamosjuntos-frontend/src/components/CountdownBanner.tsx`, usar singular y plural
  en el `<h1>` (FR-008): con `daysLeft === 1`, "{nombre}, falta" + "1" + "día para el paseo"; en
  otro caso, "{nombre}, faltan" + "{N}" + "días para el paseo".

**Checkpoint**: los cuatro tramos con número se ven distintos y con el texto correcto.

---

## Phase 5: User Story 3 - El día del paseo, durante y después (Priority: P3)

**Goal**: el día del evento, mientras dura y cuando termina, el banner muestra un mensaje propio y
nunca un número negativo.

**Independent Test**: anulando la respuesta con `today`, `ongoing` y `finished`, el banner muestra
"¡Hoy es el día!", "¡Estamos de paseo!" y "¡Qué buen paseo! Gracias por venir", sin número
(quickstart, paso 7).

- [X] T014 [US3] En `vamosjuntos-frontend/src/components/CountdownBanner.tsx`, completar la tabla
  de tramos:
  - `today`: "¡Hoy es el día!", 🎉, `from-fuchsia-700 to-red-700`;
  - `ongoing`: "¡Estamos de paseo!", 🌴, `from-emerald-700 to-sky-700`;
  - `finished`: "¡Qué buen paseo! Gracias por venir", ❤️, `from-violet-700 to-rose-700`.

  En estos tramos no hay número: el `<h1>` dice "{nombre}, " seguido del mensaje con el tamaño
  grande (`text-5xl font-black`), y el mensaje no se repite arriba.

**Checkpoint**: los 7 tramos completos (SC-003).

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T015 En `vamosjuntos-backend-004`, correr `gofmt -l .` (sin salida), `go vet ./...`,
  `golangci-lint run` y `go test ./...`; deben pasar sin errores (Principio IV, compuertas del
  pipeline).
- [X] T016 [P] En `vamosjuntos-frontend-004`, correr `npm run lint` y `npm run build` (con el chequeo
  de tipos); deben pasar sin errores.
- [X] T017 [P] Revisar a 360 px de ancho, con DevTools, los 7 tramos y los estados `loading` y
  `unavailable`, con "Nicolás López" y "mamá de lore" como nombre (quickstart, pasos 3, 6, 7 y 10).
- [ ] T018 (visto bueno) Integrar y publicar, una feature a la vez y en coordinación con las
  sesiones de la 003 y la 005:
  - rebase de `004-banner-cuenta-regresiva` sobre `main` en los dos repos, resolviendo las líneas
    compartidas de `handlers.go`, `HomePage.tsx` y `types.ts`;
  - integrar en `main`;
  - publicar con `git push origin main:production`, primero el backend y luego el frontend;
  - recorrer [quickstart.md](quickstart.md);
  - quitar los worktrees con `git worktree remove`.

---

## Dependencies & Execution Order

- **Setup (T001)** antes de todo.
- **Foundational (T002–T007)** antes de las historias:
  - T005 depende de T002 y T004;
  - T006 depende de T003 y T005;
  - T007 no depende del backend.
- **US1 (T008–T011)**:
  - T010 depende de T007 y T008;
  - T011 depende de T009 y T010.
- **US2 (T012–T013)** depende de T010.
- **US3 (T014)** depende de T012, porque completa la misma tabla.
- **Polish (T015–T018)** después de las historias que se vayan a publicar. T018 además depende del
  visto bueno y del turno de integración acordado con las otras sesiones.

## Parallel Example

```text
# Foundational: backend y tipos a la vez
T002 (countdown.go)  |  T003 (countdown_test.go)  |  T007 (types.ts)

# US1: estilos y layout a la vez, luego el banner
T008 (index.css)  |  T009 (Screen.tsx)   →   T010 (CountdownBanner.tsx)   →   T011 (HomePage.tsx)
```

## Implementation Strategy

1. **MVP**: T001–T011. El banner con el número de días ya cumple la parte emocional de la feature
   (US1).
2. **Incremento**: T012–T013 hacen que el mensaje cambie en las semanas previas, y T014 cubre el día
   del evento y lo que sigue.
3. **Cierre**: T015–T017 en local. T018 con el visto bueno y en el turno que acuerden las sesiones.
