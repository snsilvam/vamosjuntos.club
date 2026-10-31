# Quickstart: validar la card "Próximo viaje"

**Feature**: `003-card-proximo-viaje`

Se valida contra el Firebase y el Firestore reales, igual que la 008: el emulador de Auth necesita
Java, que no está instalado.

## 1. Requisitos

- La 008 publicada en producción, con al menos dos cuentas de prueba vinculadas y confirmadas.
- El documento `events/{EVENT_ID}` completo, con los datos de la constitución.
- Acceso a la consola de Firestore para comparar los conteos.

## 2. Chequeos automáticos

| Repo | Comando | Esperado |
|------|---------|----------|
| backend | `go test ./...` | pasan, incluidos los casos de `summarizeAttendance` y de `GET /api/event` |
| backend | `gofmt -l .`, `go vet ./...`, `golangci-lint run` | sin salida ni errores |
| frontend | `npm run build` | compila, con el chequeo de tipos |

## 3. Escenarios

| # | Pasos | Esperado | Cubre |
|---|-------|----------|-------|
| 1 | Entrar con una cuenta confirmada y abrir la pantalla principal | La card muestra "Paseo Familiar Diciembre 2026", salida "Bogotá · Tocancipá" y destino "Acacías, Meta" | US1-1, FR-002 |
| 2 | Mirar las fechas de la card | Inicio "miércoles 23 de diciembre de 2026, 2:00 p. m." y fin "sábado 26 de diciembre de 2026, 11:00 a. m." | US1-2, FR-003 |
| 3 | En Chrome DevTools → *Sensors* → *Location*, poner Tokio y recargar | Las mismas fechas del escenario 2 | SC-003 |
| 4 | Contar en la consola los documentos de `attendance` con `confirmed` y con `declined` y compararlos con la card | Coinciden. Pendientes = invitados − confirmados − no asistirán | US2-1, FR-004, SC-002 |
| 5 | Desde otra cuenta confirmar a un invitado pendiente ("Confirmar a otro invitado") y recargar en la primera | Confirmados +1 y pendientes −1 | US2-2, FR-005 |
| 6 | Renombrar temporalmente el campo `destination` del evento y recargar; después restaurarlo | La card dice "La información del viaje no está disponible por ahora." y el resto de la pantalla sigue funcionando | FR-006 |
| 7 | En DevTools → *Network*, revisar la respuesta de `/api/event` | Solo evento y conteos: sin nombres, ids ni códigos | FR-007 |
| 8 | `curl -i https://vamosjuntos.club/api/event` sin token | `401` con `unauthenticated` | FR-001 |
| 9 | Ver la card en un celular (o 360 px de ancho en DevTools) | Todo legible y sin scroll horizontal | SC-001 |

Al terminar, deshacer lo del escenario 5 si se usó un invitado real: borrar su documento en
`attendance`.
