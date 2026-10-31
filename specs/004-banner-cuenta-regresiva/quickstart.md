# Quickstart: Banner de Cuenta Regresiva

**Feature**: `004-banner-cuenta-regresiva`

Guía para comprobar la feature de punta a punta. El contrato está en
[contracts/api.md](contracts/api.md) y los tramos en [data-model.md](data-model.md).

## Requisitos

- Lo mismo que el [quickstart de la 008](../008-autenticacion-firebase/quickstart.md): proyecto
  de Firebase, `events/{EVENT_ID}` con `startsAt` = 2026-12-23 14:00 y `endsAt` = 2026-12-26 11:00
  (hora de Colombia), y una cuenta vinculada a un invitado confirmado.
- Ramas `004-banner-cuenta-regresiva` en `vamosjuntos-backend` y `vamosjuntos-frontend`. Si otra
  sesión tiene ocupado el working tree principal, usa un worktree aparte (`git worktree add`).

## 1. Backend: tramos y medianoches (SC-002, SC-003)

```sh
cd vamosjuntos-backend
gofmt -l .            # sin salida
go vet ./...
golangci-lint run
go test ./... -run 'Countdown' -v
```

Esperado: pasa la tabla de `countdownAt` con los ejemplos de [data-model.md](data-model.md),
incluidos:

- Los 7 tramos.
- 2026-10-30 → `far`, 54 días (US1).
- Las medianoches 22→23 de noviembre (`far` → `near`) y 15→16 de diciembre (`near` → `lastWeek`).
- `StartsAt` y `EndsAt` exactos.
- Un `now` expresado en otra zona horaria (por ejemplo Madrid) da el mismo resultado.

Y pasan las pruebas del handler: `200` con el cuerpo esperado, `401` sin token, `403` sin
confirmar y `404 event_not_found`.

## 2. Frontend: build

```sh
cd vamosjuntos-frontend
npm run build
npm run lint
```

Esperado: sin errores de tipos ni de lint.

## 3. De punta a punta

Con backend y frontend corriendo (o publicados), entra con la cuenta confirmada.

| # | Paso | Esperado |
|---|------|----------|
| 1 | Abre `/inicio` el 2026-10-30 | El banner es lo primero de la pantalla, dice "{nombre}, faltan 54 días para el paseo" y "54" es el texto más grande (US1, FR-001, FR-005) |
| 2 | Mira la entrada | El banner aparece con una animación corta |
| 3 | DevTools → Rendering → "Emulate CSS prefers-reduced-motion: reduce" y recarga | El banner aparece quieto, con el mismo contenido (US1, escenario 3) |
| 4 | Cambia la hora y la zona horaria del sistema (por ejemplo, +3 días y Madrid) y recarga | Sigue diciendo 54 (FR-003, edge cases) |
| 5 | DevTools → Sensors → Location → zona horaria `Asia/Tokyo` y recarga | Sigue diciendo 54 |
| 6 | Entra con un invitado de nombre largo ("Nicolás López", "mamá de lore") a 360 px de ancho | El nombre se ve completo y nada se sale del banner |
| 7 | En DevTools → Network, anula la respuesta de `/api/countdown` (Override content) con cada tramo de [data-model.md](data-model.md) y recarga | Cada tramo muestra su mensaje, emoji y fondo; `tomorrow` dice "falta 1 día" (US2, US3, FR-008, SC-003) |
| 8 | Anula la respuesta con `{"phase":"far","daysLeft":54,"secondsToNextChange":5}` y espera 5 s | Se hace una nueva petición a `/api/countdown` sin recargar (FR-009) |
| 9 | Con el banner abierto, cambia a otra pestaña y vuelve | Se hace una nueva petición a `/api/countdown` |
| 10 | Corta la red (DevTools → Offline) y recarga | "La cuenta regresiva no está disponible por ahora.", sin códigos ni errores técnicos (Principio V) |
| 11 | Con el banner cargado, pasa a Offline y vuelve a la pestaña | El banner sigue mostrando lo mismo |
| 12 | Abre la pantalla al sol con el brillo bajo | El texto se lee en todos los tramos (FR-010) |

El paso 7 sustituye simular fechas en producción: el backend no tiene un modo para fingir la hora
(research R8).
