# Quickstart: validar la autenticación con Firebase

**Feature**: `008-autenticacion-firebase` | **Date**: 2026-10-30

Configuración de una sola vez, primer despliegue y escenarios que prueban la spec. Contrato en
[contracts/api.md](contracts/api.md) y modelo en [data-model.md](data-model.md).

## 1. Configuración de Firebase Authentication (una vez, responsable)

En <https://console.firebase.google.com>, proyecto `vamosjuntos-club`:

1. **App web**: en *Configuración del proyecto → Tus apps → Agregar app → Web*, regístrala como
   `vamosjuntos-web`, sin configurar Hosting. Copia `apiKey`, `authDomain`, `projectId` y `appId`.
2. **Proveedores**: en *Authentication → Método de acceso*, habilita:
   - **Correo electrónico/contraseña**, sin el vínculo por correo;
   - **Google**, eligiendo tu correo como correo de soporte.
3. **Dominios autorizados**: en *Authentication → Configuración → Dominios autorizados*, agrega
   `vamosjuntos.club`. Los demás ya vienen autorizados.
4. **Variables en GitHub**: en `snsilvam/vamosjuntos-frontend`, ve a *Settings → Secrets and
   variables → Actions → **Variables*** (no Secrets) y crea `VITE_FIREBASE_API_KEY`,
   `VITE_FIREBASE_AUTH_DOMAIN`, `VITE_FIREBASE_PROJECT_ID` y `VITE_FIREBASE_APP_ID` con los
   valores del paso 1. Son públicos (constitución v3.0.0).
5. **En local**: copia `.env.example` a `.env.local` en el frontend con los mismos valores.

## 2. Primer despliegue

1. **Validación del encabezado (research R4)**:
   1. Publica primero el backend: `main` → `production`.
   2. Con un ID token real (DevTools, después de iniciar sesión en local contra el backend
      publicado, o con `curl`), llama
      `https://vamosjuntos-club.web.app/api/me` con `Authorization: Bearer <token>`.
   - Si responde `200` (o `403`), el encabezado llega.
   - Si responde `401`, se aplica el plan B de R4 antes de seguir.
2. **Frontend**: publícalo enseguida (`main` → `production`). Entre los dos despliegues, la app
   publicada no funciona unos minutos.
3. **Limpieza**: cuando todo funcione, borra el secret `SESSION_KEY` del repo del backend; ya no se
   usa.

## 3. Escenarios

| # | Pasos | Resultado esperado | Cubre |
|---|-------|--------------------|-------|
| 1 | Abrir la app sin sesión, incluso en una ruta interna (`/inicio`) | Pantalla de inicio de sesión con Google y con correo | US1-1, US3-1, FR-002 |
| 2 | `curl -i …/api/me` sin encabezado | `401 unauthenticated` | US3-1 |
| 3 | Crear una cuenta de correo de prueba | Entra y ve la lista de pendientes | US1-2, FR-001 |
| 4 | Elegir un invitado de prueba y escribir un código equivocado | Mensaje amigable; `links` no cambia | US1-6 |
| 5 | Escribir el código correcto (en minúsculas y con espacios) | Se crea `links/{guestId}` con el `uid` y el correo; aparece la pregunta con fechas en hora Colombia | US1-3, FR-004 |
| 6 | Confirmar | Entra a la pantalla principal con el nombre del invitado; sale de la lista | US1-4 |
| 7 | En otro navegador, entrar con la misma cuenta | Va directo a la pantalla principal, sin código | US2-1, SC-002 |
| 8 | Recargar la app | Sigue dentro | US2-2, FR-011 |
| 9 | Con otra cuenta nueva, escribir el código del invitado ya vinculado | `guest_already_linked` con el mensaje para el organizador | FR-005 |
| 10 | Con la cuenta vinculada, intentar vincular otro código | `account_already_linked` | FR-005 |
| 11 | Desde la cuenta vinculada, "Confirmar a otro invitado" con su código | Ese invitado queda confirmado sin vínculo; la sesión no cambia | US6 |
| 12 | Con una cuenta nueva, "Ya me confirmaron, entrar con mi código" con el código del escenario 11 | Se vincula y entra directo | US4-1 |
| 13 | Con otra cuenta, elegir un pendiente, vincular y responder "No" | Despedida; no entra. Al volver a entrar con esa cuenta, la app pregunta otra vez | US1-5, US5 |
| 14 | 5 códigos incorrectos seguidos con una cuenta | Bloqueo de 15 min para esa cuenta, aun con el código correcto | FR-014 |
| 15 | Borrar en la consola el documento `links/{guestId}` de una cuenta y recargar | Vuelve al onboarding (`unlinked`) | FR-006 |
| 16 | Quitar del catálogo (`users`) al invitado de una cuenta vinculada | `403 not_invited` con su mensaje | US3-3 |
| 17 | Cerrar sesión | Vuelve al inicio de sesión; `/api/me` con el token anterior ya no se usa | US7 |
| 18 | "Olvidé mi contraseña" con el correo de prueba | Llega el correo de Firebase para restablecerla | FR-001 |
| 19 | Cerrar la ventana de Google a mitad del inicio de sesión | No aparece ningún error técnico | FR-016 |
| 20 | En el dominio `vamosjuntos.club` (cuando esté conectado), entrar con Google | Funciona (dominio autorizado) | Assumptions |

**Limpieza al terminar**:
- borrar en *Authentication → Usuarios* las cuentas de prueba;
- borrar sus documentos en `links`, `attendance` y `loginAttempts`;
- restaurar cualquier invitado que se haya quitado en el escenario 16.

## 4. Chequeos automáticos

```bash
cd vamosjuntos-backend && gofmt -l . && go vet ./... && golangci-lint run && go test ./...
cd vamosjuntos-frontend && npm run build && npm run lint
```
