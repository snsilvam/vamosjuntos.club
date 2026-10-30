# Research: Autenticación con Firebase

**Feature**: `008-autenticacion-firebase` | **Date**: 2026-10-30

No quedan puntos `NEEDS CLARIFICATION`. Las dos decisiones de producto (código una vez para
vincular; Google + correo) las tomó el responsable antes de la spec.

## R1. Frontend: el SDK web de Firebase, como en kaizen

- **Decision**: paquete `firebase` (^12.17, modular).
  - `src/auth/firebase.ts` es el único archivo que importa el SDK, igual que
    `front-kaizensnsilvam.com/src/auth/firebase.ts`.
  - Usa `signInWithPopup` con `GoogleAuthProvider`, `signInWithEmailAndPassword`,
    `createUserWithEmailAndPassword`, `sendPasswordResetEmail`, `signOut`, `onAuthStateChanged` y
    `getIdToken`.
  - La persistencia es la que trae el SDK por defecto (IndexedDB): la sesión dura hasta cerrarla
    (FR-011).
- **Rationale**: es lo que ya funciona en kaizen. Con el popup, el inicio de sesión con Google no
  depende de redirecciones ni de cookies de terceros. Correo y contraseña no dependen de popups.
- **Alternatives considered**:
  - **FirebaseUI**: sumaría otra dependencia y estilos que chocan con Tailwind.
  - **`signInWithRedirect`**: los navegadores que bloquean almacenamiento de terceros lo rompen
    si `authDomain` no es el dominio de la app.

## R2. Configuración web de Firebase

- **Decision**:
  - Registrar una app web en el proyecto `vamosjuntos-club`.
  - `apiKey`, `authDomain` (`vamosjuntos-club.firebaseapp.com`), `projectId` y `appId` llegan
    como `VITE_FIREBASE_*`:
    - en local, desde `.env.local`, que está ignorado (`.env.example` documenta los nombres);
    - en el pipeline, desde variables de repositorio de GitHub (`vars.*`), como en kaizen.
  - El frontend falla al arrancar con un mensaje claro si falta alguna, como en kaizen.
- **Rationale**: la constitución v3.0.0 dice que esta configuración es pública por diseño. Usar
  `vars` y no `secrets` lo deja explícito, y es el patrón que el responsable ya usa.
- **Alternatives considered**:
  - **Leer `/__/firebase/init.json` de Hosting**: no funciona con el servidor de Vite en local.
  - **Escribir los valores en el código**: obligaría a cambiar código si se registra otra app.

## R3. Backend: verificar el ID token con el Admin SDK

- **Decision**:
  - `firebase.NewApp(ctx, &firebase.Config{ProjectID})` y `app.Auth(ctx)`, como
    `kaizensnsilvam-backend/internal/infrastructure/firebase/auth.go`.
  - `GOOGLE_CLOUD_PROJECT` pasa a ser obligatoria, porque el `aud` del token se compara con el
    proyecto.
  - Un middleware `requireAccount` lee `Authorization: Bearer <token>` y llama a
    `VerifyIDToken`.
  - El `Account{UID, Email}` resultante pasa al handler. Sin token o con token inválido →
    `401 unauthenticated`.
- **Rationale**:
  - El Admin SDK maneja firma, rotación de llaves y validación de `aud`, `iss` y `exp`, y guarda
    las llaves públicas en caché.
  - Corre con las credenciales por defecto de Cloud Run, sin permisos de IAM extra para
    verificar.
- **Alternatives considered**:
  - **Verificar el JWT a mano**: es código de seguridad propio.
  - **`VerifyIDTokenAndCheckRevoked`**: hace una llamada de red por request, y la revocación no
    está en la spec.

## R4. Cómo viaja el token: encabezado `Authorization` a través de Hosting

- **Decision**: `Authorization: Bearer <ID token>` en cada `fetch` a `/api/...`.
  - El token lo da `getIdToken()`, que lo renueva solo cuando está por vencer (1 hora).
  - Al recibir `401`, el cliente reintenta una vez con `getIdToken(true)` antes de mandar al
    usuario a iniciar sesión.
- **Rationale**: es como kaizen manda el token (`src/services/api.ts`).
- **Riesgo**: ninguna fuente oficial confirma que Hosting reenvíe `Authorization` a Cloud Run en
  un rewrite. Guías de terceros dicen que sí y que solo se filtran las cookies (salvo
  `__session`). Kaizen no lo prueba, porque llama a la API directo.
  - **Validación**: es la primera tarea de implementación. Se despliega el backend con `/api/me`
    y se llama a través de `vamosjuntos-club.web.app` con un token real: `200` confirma el
    reenvío.
  - **Plan B** si no llega: el frontend copia el ID token a la cookie `__session` (la única que
    Hosting reenvía) en cada `onIdTokenChanged`, y el middleware lee el encabezado o, si falta,
    esa cookie. El resto del diseño no cambia.

## R5. El vínculo cuenta ↔ invitado

- **Decision**: un documento `events/{EVENT_ID}/links/{guestId}` con `{uid, email, linkedAt}`.
  - **Buscar el invitado de una cuenta**: consulta `links where uid == X limit 1`. Es una lectura,
    con el índice simple automático.
  - **Vincular**: una transacción:
    1. consulta si el `uid` ya tiene vínculo → si lo tiene, `409 account_already_linked`;
    2. `tx.Create(links/{guestId})`. Si el documento ya existe, la transacción falla y el
       resultado es `409 guest_already_linked`.
  - **Unicidad**: `Create` garantiza un invitado por cuenta aunque dos cuentas lo intenten a la vez
    (FR-005, SC-004). La consulta por `uid` cubre el otro sentido.
  - **Deshacer un vínculo**: el organizador borra un solo documento.
- **Rationale**: un solo documento por vínculo es lo más fácil de entender y de corregir a mano en
  la consola. Usar el `guestId` como id hace que Firestore imponga la unicidad del invitado.
- **Alternatives considered**:
  - **Escribir el `uid` en `users/{guestId}`**: la app no edita invitados (Principio I).
  - **Dos documentos, `accounts/{uid}` y `links/{guestId}`**: unicidad fuerte en ambos sentidos,
    pero dos documentos que borrar a mano para deshacer un vínculo. El caso de la misma cuenta
    vinculando dos invitados a la vez no ocurre en la práctica.
  - **Custom claims de Firebase**: el vínculo quedaría en el proveedor y cambiarlo exige el Admin
    SDK, no la consola.

## R6. Flujo y API

- **Decision**:
  - **`GET /api/me`** devuelve el estado de la cuenta:
    - `unlinked`: va a la lista de pendientes;
    - `pending` o `declined`: va a la pregunta de asistencia, con el evento;
    - `confirmed`: entra a la app.
  - **`POST /api/onboarding/link {guestId?, code}`**: vincula y devuelve el mismo estado.
    - Con `guestId`, el código debe ser de ese invitado (flujo de la lista).
    - Sin `guestId`, se busca el invitado por código ("Ya me confirmaron, entrar con mi código").
  - **`POST /api/onboarding/respond {attending}`**: ya no pide código, porque la cuenta está
    vinculada.
  - **Cerrar sesión**: es `signOut()` del SDK, sin endpoint.
  - **Un código incorrecto es `400 invalid_code`**, no `401`, porque `401` ahora significa token
    inválido.
- **Rationale**: un solo endpoint de estado evita que el frontend deduzca a dónde ir, y un solo
  endpoint de vínculo cubre la lista y "entrar con mi código" (FR-004).
- **Alternatives considered**: conservar `verify` y `reentry` de la 001 obligaría a mantener dos
  caminos para lo mismo.

## R7. Lo que sale y lo que se descarta

- **Sale**:
  - `SESSION_KEY` (secret de GitHub y variable de Cloud Run);
  - `session.go` (HMAC y cookie `__session`);
  - `X-Device-Id` y `src/api/deviceId.ts`;
  - `verify`, `reentry`, `GET` y `DELETE /api/session`.
- **El bloqueo por intentos** se cuenta por `uid` (FR-014), en el mismo `loginAttempts` con la misma
  regla pura `registerAttempt`.
- **Descartado** (Principio VI):
  - verificación de correo;
  - otros proveedores (Apple, teléfono);
  - nombre o foto de perfil;
  - pantalla para administrar vínculos;
  - revocación de tokens;
  - cookies de sesión de servidor.

## R8. Configuración de Firebase Authentication (una vez)

- **Decision**: lo hace el responsable en la consola, con los pasos de
  [quickstart.md §1](quickstart.md):
  1. registrar la app web y copiar su configuración;
  2. habilitar los proveedores Google (con correo de soporte) y Correo/contraseña;
  3. agregar `vamosjuntos.club` a los dominios autorizados (`localhost`,
     `vamosjuntos-club.web.app` y `vamosjuntos-club.firebaseapp.com` vienen autorizados);
  4. crear en el repo del frontend las variables `VITE_FIREBASE_*`.
- **Rationale**: son cuatro clics que no cambian. La spec 002 ya excluye la infraestructura como
  código.

## R9. Pruebas

- **Decision**:
  - **Backend**: el verificador y el almacén de vínculos son interfaces pequeñas con fakes. Se
    prueban:
    - los handlers: estados de `me`, vincular por lista y por código, conflictos `409`, bloqueo
      `429`, `respond` sin código y confirmar a otro;
    - la regla pura del vínculo.
  - **Frontend**: el build con chequeo de tipos.
  - **De punta a punta**: el quickstart contra el Firebase real, con una cuenta de correo de
    prueba que se borra al final y la cuenta Google del responsable.
- **Rationale**: el emulador de Auth necesita Java, que no está instalado. El resto se cubre igual
  que en la 001.

## Fuentes

- Ejemplo del responsable: `C:\golang\kaizen-app\front-kaizensnsilvam.com\src\auth\firebase.ts`,
  `src/services/api.ts` y `kaizensnsilvam-backend/internal/infrastructure/firebase/auth.go`.
- [Firebase Hosting + Cloud Run (rewrites)](https://firebase.google.com/docs/hosting/cloud-run)
- [Rewrites de Hosting hacia Cloud Run y encabezados (guía de terceros)](https://oneuptime.com/blog/post/2026-02-17-how-to-use-firebase-hosting-rewrites-to-route-traffic-to-cloud-run-services/view)
- [Firebase Hosting: caché y cookie `__session`](https://firebase.google.com/docs/hosting/manage-cache)
