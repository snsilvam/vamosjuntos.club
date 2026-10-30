# Implementation Plan: Autenticación con Firebase

**Branch**: `008-autenticacion-firebase` | **Date**: 2026-10-30 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/008-autenticacion-firebase/spec.md`

## Summary

Cada familiar inicia sesión con Firebase Authentication (Google o correo y contraseña), como en
kaizen-app. La primera vez vincula su cuenta con un invitado escribiendo su código; después entra
solo con su cuenta.

Enfoque técnico:

- **Frontend**: SDK web de Firebase con la misma forma que kaizen. Un único archivo,
  `src/auth/firebase.ts`, conoce el SDK. `signInWithPopup` para Google, correo y contraseña,
  creación de cuenta y restablecimiento de contraseña. Cada llamada a `/api` lleva
  `Authorization: Bearer <ID token>`.
- **Backend**: Admin SDK de Go (`firebase.google.com/go/v4`, como en kaizen). `VerifyIDToken` en
  un middleware reemplaza la cookie HMAC.
- **Vínculo**: el vínculo cuenta ↔ invitado es un documento `events/{EVENT_ID}/links/{guestId}`
  con el `uid`.
  - Se crea en una transacción con `Create`, que falla si ya existe: así un invitado nunca queda
    con dos cuentas.
  - Antes de crearlo se busca que la cuenta no esté vinculada a otro invitado.
- **API**: aparecen `GET /api/me` y `POST /api/onboarding/link`. `respond` deja de pedir código.
  `verify`, `reentry` y `GET/DELETE /api/session` desaparecen, porque cerrar sesión ahora es del
  SDK.
- **Lo que sale**: `SESSION_KEY`, `X-Device-Id` y la cookie `__session`. El límite de intentos
  pasa a contarse por `uid`.

Detalle de las decisiones en [research.md](research.md).

## Technical Context

**Language/Version**: Go 1.26 (backend) y TypeScript + React 19 (frontend), sin cambios.

**Primary Dependencies**:
- Backend: se agrega `firebase.google.com/go/v4` (Admin SDK, v4.21.0 como en kaizen), que se
  justifica porque verificar tokens a mano es código de seguridad delicado. Sigue
  `cloud.google.com/go/firestore`.
- Frontend: se agrega `firebase` (SDK web modular, ^12.17 como en kaizen).

**Storage**: Firestore `vamosjuntos-club-db`.
- **Nuevo**: `events/{EVENT_ID}/links/{guestId}`.
- **Cambia**: `loginAttempts/{uid}` pasa a usar el `uid` como id en lugar del dispositivo.
- **Sin cambios**: `users`, `attendance` y el evento.

**Testing**:
- **Backend**: `go test` con fakes del verificador de tokens y del almacén de vínculos; reglas
  puras de vínculo y de estado.
- **Frontend**: build con chequeo de tipos.
- **De punta a punta**: manual, con [quickstart.md](quickstart.md), contra el Firebase real. El
  emulador de Auth necesita Java, que no está instalado.

**Target Platform**: Cloud Run (backend) y Firebase Hosting (frontend), con el celular como cliente
principal.

**Project Type**: aplicación web (SPA + API REST), en dos repositorios.

**Performance Goals**:
- Primer ingreso en menos de 2 minutos (SC-001).
- Reingreso en menos de 15 segundos (SC-002).
- `VerifyIDToken` valida en memoria con las llaves públicas en caché, sin una llamada por request.

**Constraints**:
- La configuración web de Firebase es pública: `apiKey`, `authDomain`, `projectId` y `appId`.
- Códigos fuera del cliente y del proveedor (FR-018).
- Un invitado por cuenta y una cuenta por invitado (FR-005).
- Mensajes amigables también para los errores del SDK (FR-016).

**Scale/Scope**: 25 invitados y unas 6 pantallas. La API queda en 5 endpoints.

**Estado actual de Firebase** (verificado el 2026-10-30):
- El proyecto ya es de Firebase y la API `identitytoolkit` está habilitada.
- Falta registrar una app web y habilitar los proveedores Google y Correo/Contraseña.
- Falta autorizar `vamosjuntos.club`.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Se evalúa contra la constitución **v3.0.0**.

| Principio | Verificación | Estado |
|-----------|--------------|--------|
| I. Alcance cerrado | La app sigue sin crear ni editar eventos ni invitados. Solo escribe `links`, `attendance` y `loginAttempts`. | ✅ |
| II. Acceso por invitación | Firebase Authentication con Google y correo. El backend verifica el ID token en cada request protegido. El orden es: iniciar sesión → nombre → código (vincula una vez) → confirmación. El catálogo sigue en Firestore, cerrado a clientes. | ✅ |
| III. Simplicidad | Paquete `main` plano: se agregan `auth.go` y `links.go`. Un solo archivo del SDK en el frontend, como en kaizen. Sin librerías de estado ni de formularios. | ✅ |
| IV. Código idiomático | Interfaces pequeñas en los handlers (`tokenVerifier`, `linkStore`). Regla pura `linkDecision`. `context.Context` en el I/O. Errores envueltos. | ✅ |
| V. Errores y HTTP | `401` sin token válido; `403` (`not_linked` o `not_invited`); `400` código inválido; `409` vínculos en conflicto; `429` bloqueo. La UI traduce también los códigos `auth/*` del SDK. | ✅ |
| VI. Cero código sombra | Solo lo de la spec. Se descartan perfiles, foto, verificación de correo, más proveedores, administración de vínculos en la app y cookies de sesión de servidor (R7). | ✅ |
| Stack | Firebase Authentication en la fila de autenticación; Hosting reenvía `/api/**` a Cloud Run. | ✅ |
| Despliegue continuo | El backend deja de necesitar secrets. El frontend recibe la configuración web de Firebase como variables de GitHub, como kaizen. | ✅ |

**Re-check post-diseño**: ✅ El diseño de la Fase 1 no agrega colecciones ni endpoints fuera de los
anteriores.

## Project Structure

### Documentation (this feature)

```text
specs/008-autenticacion-firebase/
├── plan.md              # Este archivo
├── research.md          # Fase 0
├── data-model.md        # Fase 1
├── quickstart.md        # Fase 1
├── contracts/
│   └── api.md           # Fase 1: API después de la 008 (reemplaza contracts/api.md de la 001)
└── tasks.md             # Fase 2 (/speckit-tasks)
```

### Source Code

```text
vamosjuntos-backend/
├── auth.go              # NUEVO: Account, tokenVerifier (Admin SDK), middleware requireAccount/requireGuest
├── auth_test.go         # NUEVO
├── links.go             # NUEVO: LinkStore (links/{guestId}): GuestOf(uid), Link(uid, email, guestId)
├── links_test.go        # NUEVO: regla pura del vínculo
├── handlers.go          # /api/me, /api/onboarding/{guests,link,respond}, /api/guests/{id}/confirmation
├── handlers_test.go     # fakes de tokenVerifier y linkStore
├── attempts.go          # el id del documento pasa a ser el uid
├── config.go            # sin SESSION_KEY; GOOGLE_CLOUD_PROJECT obligatorio (aud del token)
├── main.go              # crea el cliente de Auth del Admin SDK
├── session.go           # SE BORRA (HMAC y cookie __session), junto con session_test.go
└── .github/workflows/backend-deploy.yml   # sin SESSION_KEY; agrega GOOGLE_CLOUD_PROJECT

vamosjuntos-frontend/
├── src/auth/firebase.ts         # NUEVO: único archivo que conoce el SDK (igual que kaizen)
├── src/auth/AuthProvider.tsx    # NUEVO: contexto con user, signIn*, signOut, getIdToken
├── src/auth/useAuth.ts          # NUEVO
├── src/pages/SignInPage.tsx     # NUEVO: Google, correo (entrar/crear) y "olvidé mi contraseña"
├── src/pages/LinkByCodePage.tsx # REEMPLAZA ReentryPage: "Ya me confirmaron, entrar con mi código"
├── src/api/client.ts            # Authorization: Bearer; sin X-Device-Id (se borra deviceId.ts)
├── src/session/*                # "me": status unlinked|pending|confirmed|declined guía las rutas
├── src/pages/{GuestList,Code,Confirm,Home,ConfirmOther}Page.tsx  # ajustes al nuevo contrato
├── src/messages.ts              # nuevos códigos de la API y códigos auth/* del SDK
├── .env.example                 # NUEVO: VITE_FIREBASE_* (valores públicos, para desarrollo)
└── .github/workflows/front-deploy.yml    # build con vars.VITE_FIREBASE_* (como kaizen)
```

**Structure Decision**: se siguen los nombres de kaizen (`src/auth/firebase.ts`, `AuthProvider`)
para que el responsable reconozca el código. En el backend todo sigue en el paquete `main` plano.

**Ramas**: `008-autenticacion-firebase` en ambos repos, desde `main`. El backend y el frontend
deben publicarse juntos, porque el contrato cambia: primero el backend y enseguida el frontend.
Entre los dos despliegues la app publicada no funciona unos minutos. Antes del primer usuario real
eso no importa.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| Nueva dependencia del backend (`firebase.google.com/go/v4`) | Verificar el ID token: firma RS256, rotación de llaves públicas, `aud`, `iss` y `exp` | Hacerlo a mano con `crypto` de la biblioteca estándar es código de seguridad propio y sin necesidad; kaizen ya usa el Admin SDK |
