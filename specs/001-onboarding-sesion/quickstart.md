# Quickstart: validar Onboarding y Sesión

**Feature**: `001-onboarding-sesion` | **Date**: 2026-10-30

Guía para comprobar de punta a punta, en local, que la feature cumple el spec. Contratos en
[contracts/api.md](contracts/api.md) y modelo en [data-model.md](data-model.md).

## Prerrequisitos

- Go 1.26, Node 24 y Google Cloud CLI con el componente del emulador de Firestore
  (`gcloud components install cloud-firestore-emulator`).
- Los repos hermanos `vamosjuntos-backend` y `vamosjuntos-frontend`, con la feature implementada.

## 1. Levantar el entorno

```bash
# Terminal 1: emulador de Firestore
gcloud emulators firestore start --host-port=localhost:8081

# Terminal 2: backend (códigos de PRUEBA, nunca los reales)
cd vamosjuntos-backend
export FIRESTORE_EMULATOR_HOST=localhost:8081
export GOOGLE_CLOUD_PROJECT=vamosjuntos-local
# Contra el proyecto real: GOOGLE_CLOUD_PROJECT=vamosjuntos-club y FIRESTORE_DATABASE=vamosjuntos-club-db
export EVENT_ID=paseo-familiar-diciembre-2026
export SESSION_KEY=$(openssl rand -base64 32)

# Sembrar el evento directamente en el emulador (la app nunca crea eventos, Principio I)
curl -s -X POST \
  "http://localhost:8081/v1/projects/vamosjuntos-local/databases/(default)/documents/events?documentId=paseo-familiar-diciembre-2026" \
  -H "Content-Type: application/json" -d '{"fields":{
    "name":{"stringValue":"Paseo Familiar Diciembre 2026"},
    "startsAt":{"timestampValue":"2026-12-23T19:00:00Z"},
    "endsAt":{"timestampValue":"2026-12-26T16:00:00Z"},
    "destination":{"stringValue":"Acacías, Meta, Colombia"},
    "departurePlaces":{"arrayValue":{"values":[{"stringValue":"Bogotá"},{"stringValue":"Tocancipá"}]}},
    "capacity":{"integerValue":"25"}}}'

# Invitados de PRUEBA en events/{EVENT_ID}/users (campos Nombre y código, como en producción)
for g in "carmencita|Carmencita|prueba01" "liam|Liam|prueba02" "papa-de-lore|papá de lore|prueba03"; do
  IFS='|' read -r id nombre codigo <<< "$g"
  curl -s -X POST     "http://localhost:8081/v1/projects/vamosjuntos-local/databases/(default)/documents/events/paseo-familiar-diciembre-2026/users?documentId=$id"     -H "Content-Type: application/json; charset=utf-8"     -d "{\"fields\":{\"Nombre\":{\"stringValue\":\"$nombre\"},\"código\":{\"stringValue\":\"$codigo\"}}}"
done

go run .   # escucha en :8080

# Terminal 3: frontend (el proxy de Vite manda /api a :8080)
cd vamosjuntos-frontend
npm install && npm run dev
```

> En local la cookie `Secure` funciona porque los navegadores tratan `localhost` como contexto
> seguro.

## 2. Escenarios

| # | Pasos | Resultado esperado | Cubre |
|---|-------|--------------------|-------|
| 1 | Abrir la app sin sesión | Lista: Carmencita, Liam, papá de lore, más la opción "Ya confirmé" | US1-1, FR-002 |
| 2 | Elegir Carmencita y escribir `PRUEBA01 ` (mayúsculas y espacio) | Muestra evento, fechas en hora Colombia y la pregunta | FR-003, FR-005 |
| 3 | Confirmar | Pantalla principal; al recargar en otra ventana privada, Carmencita no está en la lista | US1-4, SC-004 |
| 4 | Ventana privada: abrir directo una ruta interna (`/inicio`) | Se muestra el onboarding | US2-1 |
| 5 | `curl -i localhost:8080/api/session` | `401 unauthenticated` | US2-2, SC-003 |
| 6 | Con la sesión de Carmencita: "Confirmar a otro invitado", elegir Liam y escribir `prueba02` | Liam confirmado; la sesión sigue siendo de Carmencita | US4 |
| 7 | Elegir papá de lore, escribir su código y responder "No" | Mensaje "¡Te vamos a extrañar!..."; sale de la lista y no hay sesión | US5, FR-007 |
| 8 | "Ya confirmé" + `prueba03` | Vuelve a preguntar; al confirmar, entra | US3-5, FR-011 |
| 9 | "Ya confirmé" + `prueba01` en una ventana privada nueva | Entra como Carmencita | US3-1 |
| 10 | 5 códigos incorrectos seguidos | Mensaje de bloqueo; el 6.º intento, aun con el código correcto, se rechaza | FR-010 |
| 11 | "Cerrar sesión" | Vuelve al onboarding; `/api/session` responde `401` | US6 |
| 12 | Borrar el documento de Carmencita en `users` y recargar con su sesión, sin reiniciar el backend | La app vuelve al onboarding (`403 not_invited`) | US2-3, FR-001 |
| 13 | Dejar en `users` un código repetido, uno de 5 caracteres o 26 invitados | Al arrancar, el backend termina con un error claro en el log. Con el backend en marcha, la app muestra el mensaje genérico y el log nombra a los invitados, sin códigos | FR-013, FR-016 |
| 15 | Agregar un documento a `users` con el backend en marcha | El invitado aparece en la lista de pendientes al recargar | FR-001 |
| 14 | Borrar el evento del emulador y recargar | "El paseo aún no está disponible..." | FR-004 |

## 3. Chequeos automáticos

```bash
cd vamosjuntos-backend && gofmt -l . && go vet ./... && golangci-lint run && go test ./...
cd vamosjuntos-frontend && npm run build   # incluye el chequeo de tipos
```

Todos deben terminar sin errores; son las mismas compuertas del pipeline (spec 002).

## 4. Crear el evento real en producción (una sola vez, organizador)

En la consola de Firestore, en la base `vamosjuntos-club-db`, crea la colección `events` y en ella el documento
`paseo-familiar-diciembre-2026` con los campos de [data-model.md](data-model.md#evento--firestore-eventsevent_id).
Ojo: la consola captura las fechas en la zona horaria del navegador. Verifica que `startsAt`
quede como 2026-12-23 19:00 UTC y `endsAt` como 2026-12-26 16:00 UTC.

## 5. Invitados reales en producción (organizador)

En la misma base, dentro del documento del evento, la subcolección `users` tiene un documento por
invitado, con id automático y los campos `Nombre` y `código`
([data-model.md](data-model.md#invitado--firestore-eventsevent_idusersguestid)).

- **Cambios**: agregar, corregir o borrar un invitado aplica de inmediato, sin desplegar.
- **Ids**: no cambies el id de un documento; es el `guestId` de las sesiones y respuestas.
- **Códigos**: los lee el organizador en la consola para entregarlos. Las reglas de Firestore
  impiden leerlos desde fuera del backend.
