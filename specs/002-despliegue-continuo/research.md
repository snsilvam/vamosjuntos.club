# Research: Despliegue Continuo

**Feature**: `002-despliegue-continuo` | **Date**: 2026-10-30

No quedaron puntos `NEEDS CLARIFICATION`. Cada decisión sigue el formato Decisión, Razón y
Alternativas.

**Decisión del responsable (2026-10-30)**: seguir la misma estructura de `kaizen-app`. Eso implica
la rama `production` y los secretos en GitHub, sin Secret Manager. La constitución se enmendó a la
v1.2.0 para permitirlo.

## R1. Base: los workflows de `kaizen-app`

**Decisión**: copiar la estructura de los ejemplos del responsable:
`kaizensnsilvam-backend/.github/workflows/backend-deploy.yml`,
`front-kaizensnsilvam.com/.github/workflows/front-deploy.yml` y el `Dockerfile` del backend.
Solo se cambia lo que exige la spec o la constitución.

| De kaizen | En este plan | Por qué |
|-----------|--------------|---------|
| Disparo con `push` a `production` | Igual, más `workflow_dispatch` | FR-003 |
| Un job `deploy` en `ubuntu-latest` con `contents: read` e `id-token: write` | Igual | |
| Bloque `env` con `PROJECT_ID`, `REGION`, `SERVICE` e `IMAGE` | Igual | |
| `auth` con `workload_identity_provider` y `service_account` en `with:` | Igual (`auth@v3`) | |
| Un proveedor de WIF por repo dentro de `github-pool` | Igual: `github-provider` y `github-frontend-provider` | R3 |
| Cuentas `github-actions-cloud-run` y `github-actions-frontend` | Igual | R3 |
| `setup-gcloud` + `configure-docker` + `docker build/push` | Igual | R4 |
| Repositorio de Artifact Registry `containers` | Igual | |
| Paso que valida que existan los secrets (`test -n ... \|\| exit 1`) | Igual, para `GUESTS_JSON` y `SESSION_KEY` | Caso borde "falta un secreto" |
| `gcloud run deploy` con `--set-env-vars "^;^..."` desde `secrets.*` | Igual | R6 |
| `--region us-central1 --platform managed --allow-unauthenticated --port 8080` | Igual | R5 |
| Dockerfile multi-etapa golang-alpine → distroless | Igual, con `static-debian12:nonroot` | El binario es estático y corre sin root |
| — | Compuertas antes de autenticarse | FR-004 a FR-006 (no está en kaizen) |
| — | `concurrency` | FR-011 (no está en kaizen) |
| Imagen `:latest` | Imagen `:<commit>` | FR-012: cada revisión dice qué commit corre |
| Cuenta de Compute por defecto en Cloud Run | Cuenta de ejecución propia, `vamosjuntos-backend-run` | R5 |
| Frontend: Docker + nginx en Cloud Run con variables `VITE_*` | `npm run build` + Firebase Hosting, sin variables de build | Constitución; la API está en el mismo origen (R8 y R9) |
| `checkout@v4`, `setup-node@v4`, `auth@v2`, `setup-gcloud@v2` | Versiones actuales (R13) | Las v3 de Google corren en Node 24 |

**Alternativas**: los ejemplos oficiales de Google. Se descartaron por decisión del responsable;
kaizen ya le funciona.

## R2. Región: `us-central1`

**Decisión**: Cloud Run y Artifact Registry en `us-central1`, igual que en kaizen.

**Razón**:
- Está en la lista de regiones que admiten rewrites de Hosting hacia Cloud Run, y es una de las
  recomendadas para mejor rendimiento.
- Está junto a Firestore `nam5`.

**Alternativas**:
- `southamerica-east1`: cada request iría y volvería a Firestore en EE. UU.
- `northamerica-south1`: no admite rewrites de Hosting.

## R3. Autenticación: WIF como en kaizen

**Decisión**:

| Proveedor (en `github-pool`) | Condición | Cuenta que puede suplantar |
|------------------------------|-----------|----------------------------|
| `github-provider` | `assertion.repository == 'snsilvam/vamosjuntos-backend' && assertion.ref == 'refs/heads/production'` | `github-actions-cloud-run` |
| `github-frontend-provider` | `assertion.repository == 'snsilvam/vamosjuntos-frontend' && assertion.ref == 'refs/heads/production'` | `github-actions-frontend` |

- **Emisor**: `https://token.actions.githubusercontent.com`.
- **Mapeo**: `google.subject=assertion.sub`, `attribute.repository=assertion.repository` y
  `attribute.ref=assertion.ref`.
- **Vínculo**: cada cuenta da `roles/iam.workloadIdentityUser` solo a
  `principalSet://…/attribute.repository/snsilvam/<repo>`.
- **Sin secretos para autenticarse**: el proveedor y la cuenta van en `with:` del paso `auth`, como
  en kaizen. Son identificadores, no secretos.

**Razón**:
- Cumple FR-009, sin llaves JSON.
- La condición sobre `ref` refuerza FR-002 del lado de la nube: desde `main` u otra rama, GCP no
  entrega credenciales.

**Alternativas**:
- **Llaves JSON**: prohibidas por la constitución.
- **WIF directo, sin cuenta de servicio**: no está probado con `firebase-tools` y se aleja de
  kaizen.

## R4. Imagen del backend

**Decisión**:
- **Construcción**: etapa `golang:1.26-alpine` con
  `CGO_ENABLED=0 go build -trimpath -ldflags="-s -w"`.
- **Imagen final**: `gcr.io/distroless/static-debian12:nonroot`, que trae los certificados CA.
- **Dónde se construye**: Docker del runner, con push a
  `us-central1-docker.pkg.dev/vamosjuntos-club/containers/vamosjuntos-backend:<github.sha>`.
- **`.dockerignore`**: excluye `.git`, `.github`, `.env*` y `*.exe`.

**Razón**:
- Es el mismo flujo de kaizen.
- Etiquetar con el commit hace cada despliegue trazable y reproducible.
- En este proyecto el binario es uno solo y estático, así que la imagen `static` alcanza (kaizen
  usa `base`).

**Alternativas**:
- `gcloud run deploy --source`: necesita Cloud Build y más permisos.
- `distroless/base`: trae glibc, que no hace falta.

## R5. Despliegue en Cloud Run

**Decisión**: `gcloud run deploy vamosjuntos-backend` con los flags de kaizen, más:
- `--service-account vamosjuntos-backend-run@…`;
- `--max-instances 2`.

Los flags exactos están en [contracts/pipelines.md](contracts/pipelines.md#cloud-run-vamosjuntos-backend).

**Razón**:
- **Si el arranque falla, no se publica**: Cloud Run solo mueve el tráfico cuando la revisión
  nueva arranca. Si `GUESTS_JSON` o `SESSION_KEY` son inválidos, el proceso termina con su mensaje
  (spec 001), el despliegue falla y la revisión anterior sigue sirviendo (FR-006).
- **Acceso público**: `--allow-unauthenticated` hace falta porque Hosting invoca al servicio como
  cliente público.
- **Cuenta de ejecución propia**: solo tiene `roles/datastore.user`. La cuenta de Compute por
  defecto, la que usa kaizen, tiene rol de Editor en todo el proyecto. Es la única diferencia de
  identidad con kaizen, y cuesta un flag y dos comandos.
- **Tope de costo**: `--max-instances 2`. Una instancia atiende de sobra a 25 personas.

**Alternativas**:
- La cuenta de Compute por defecto, como en kaizen: tiene permisos de más.
- `deploy-cloudrun@v3`: hace lo mismo, pero no es el comando que ya conoce el responsable.

## R6. Secretos: GitHub → variables de entorno de Cloud Run

**Decisión**:
- **Dónde viven**: `GUESTS_JSON` y `SESSION_KEY` son repository secrets de
  `snsilvam/vamosjuntos-backend`, que el responsable crea en *Settings → Secrets and variables →
  Actions*.
- **Validación previa**: un paso, como el de `DATABASE_URL` en kaizen, falla si alguno está vacío.
- **Cómo llegan a Cloud Run**: en el paso de despliegue, los secrets entran como variables de
  entorno del paso, y `gcloud run deploy` los pasa con el separador `^;^`:

  ```text
  --set-env-vars "^;^EVENT_ID=$EVENT_ID;FIRESTORE_DATABASE=$FIRESTORE_DATABASE;GUESTS_JSON=$GUESTS_JSON;SESSION_KEY=$SESSION_KEY"
  ```

- **Variables no secretas**: `EVENT_ID=paseo-familiar-diciembre-2026` y
  `FIRESTORE_DATABASE=vamosjuntos-club-db` van en el `env` del workflow. `GOOGLE_CLOUD_PROJECT` no
  hace falta, porque el backend lo detecta en Cloud Run.

**Razón**:
- Es el patrón de kaizen y cumple la constitución v1.2.0 y FR-010.
- Los códigos nunca están en archivos versionados, y GitHub los reemplaza por `***` en los logs.
- El separador `^;^` evita que las comas del JSON se tomen como separadores de variables.

**Consecuencias que el responsable acepta**:
- **Los códigos no pueden contener `;`**: los partiría el separador. Los códigos alfanuméricos
  bastan.
- **Las variables se ven en texto plano en la consola de Cloud Run**: cualquiera con acceso al
  proyecto (hoy, solo el responsable) puede verlas en la revisión.
- **Cambiar el catálogo**: se edita el secret en GitHub y se usa "Run workflow" (FR-003).
- **Rotar `SESSION_KEY`**: se hace igual, y se cierran todas las sesiones.
- **Nunca usar `set -x`** en los pasos que reciben secretos.

**Alternativas**:
- **Secret Manager con `--set-secrets`**: el responsable lo descartó para mantener la estructura de
  kaizen.
- **`--env-vars-file` generado en el runner**: evita la restricción del `;`, pero escribe los
  secretos en un archivo y se aleja de kaizen.

## R7. Compuertas de calidad

**Decisión**: en el mismo job y antes del paso `auth`.

- **Backend**:
  1. `test -z "$(gofmt -l .)"`.
  2. `go vet ./...`.
  3. `golangci-lint` v2.15.0 con la configuración por defecto.
  4. `go test ./...`.
- **Frontend**: `npm ci` y `npm run build` (`tsc -b && vite build`). Ocupa el lugar del paso
  `Typecheck` de kaizen.
- **Lint del frontend**: `oxlint` no es compuerta, porque la spec no la pide.

**Razón**:
- Cumple FR-004 a FR-006.
- Un commit roto nunca llega a autenticarse en GCP.
- golangci-lint v2.15.0 es la misma versión que se usa en local; Go 1.26 lo exige desde la v2.9.

**Alternativas**:
- **Un job de compuertas aparte con `needs`**: se aleja del job único de kaizen.
- **Compuertas en pull requests**: la spec no las pide.

## R8. Frontend a Firebase Hosting con `firebase-tools`

**Decisión**: esta parte es distinta de kaizen, porque la constitución exige Hosting.

1. Los mismos `checkout` y `setup-node` de kaizen, más `npm ci` y `npm run build`.
2. `auth@v3` con `github-frontend-provider` y `github-actions-frontend`, igual que en kaizen. Crea
   el archivo de credenciales y exporta `GOOGLE_APPLICATION_CREDENTIALS`.
3. `npx --yes firebase-tools@15.33.0 deploy --only hosting --project vamosjuntos-club
   --message "<github.sha>" --non-interactive`.

Roles de la cuenta:
- `roles/firebasehosting.admin`;
- `roles/run.viewer`, porque el despliegue valida el rewrite;
- `roles/serviceusage.apiKeysViewer` y `roles/serviceusage.serviceUsageConsumer`.

**Razón**:
- Cumple FR-009 sin llaves.
- La versión fija de la CLI (publicada el 2026-10-08) evita sorpresas.
- `--message` deja el commit en el historial de Hosting (FR-012).

**Alternativas**:
- **`FirebaseExtended/action-hosting-deploy`**: exige una llave JSON.
- **`firebase login:ci`**: está obsoleto y es una credencial de larga duración.

**Riesgo**: que `firebase-tools` funcione con credenciales federadas lo documentan terceros. Se
valida en el primer despliegue ([quickstart.md §3](quickstart.md), escenario 3). Si falla, se
detiene la implementación y se revisa este punto.

## R9. Enrutamiento y caché en Hosting (`firebase.json`)

**Decisión**: la misma base del `firebase.json` de kaizen (`public: dist`, `ignore` y SPA), con dos
agregados:

1. **Rewrite de la API**: `/api/**` → `run: { serviceId: "vamosjuntos-backend",
   region: "us-central1" }`. Va antes del rewrite `**` → `/index.html`. No usa `pinTag`.
2. **Encabezado**: `Cache-Control: no-cache` para todo menos `/assets/**` y `/api/**`, con el glob
   `!/@(assets|api)/**`.

**Razón**:
- **Mismo dominio (FR-008)**: la API comparte origen con el sitio, sin CORS. Hosting reenvía la
  cookie `__session`.
- **Sin `pinTag`**: acoplaría los dos pipelines, que la spec quiere independientes.
- **Por qué `no-cache` en el HTML**: por defecto, el navegador guarda el HTML 1 hora. Con
  `no-cache` se revalida, y el usuario ve la versión nueva al recargar (US2) sin pedir archivos de
  `/assets` que ya no existen.
- **Orden del primer despliegue**: Hosting valida que el servicio del rewrite exista, así que el
  backend se despliega primero.

**Alternativas**: un subdominio para la API obliga a CORS y a cookies entre sitios.

## R10. Disparadores y concurrencia

**Decisión**:
- **Disparadores**: `push` a `production` y `workflow_dispatch`.
- **Concurrencia**: `concurrency: { group: deploy-production, cancel-in-progress: false }`.
- **Guarda del job**: `if: github.ref == 'refs/heads/production'`.

**Razón**:
- **Uno a la vez (FR-011)**: hay una sola ejecución por grupo, y de las pendientes solo queda la
  más reciente. La versión final es la del último commit, y nunca se corta un despliegue a la
  mitad.
- **Relanzar (FR-003)**: "Run workflow" o "Re-run jobs".
- **Otras ramas (FR-002)**: `main` y las demás ramas no disparan nada. Un "Run workflow" desde
  otra rama se salta por la guarda, y además WIF negaría las credenciales.

**Alternativas**: `cancel-in-progress: true` puede cortar un despliegue a la mitad.

## R11. Registro y notificaciones

**Decisión**: se usa lo que ya traen las plataformas.

- **GitHub Actions**: guarda el commit, el resultado y los logs, con retención de 90 días.
- **Cloud Run**: la imagen de cada revisión lleva el commit en su etiqueta.
- **Hosting**: cada versión lleva el commit como mensaje.
- **Fallos (US3-3)**: GitHub avisa por correo.

**Razón**: cumple FR-012 y US3-3 sin canales nuevos.

**Alternativas**: avisos a Slack o a un chat. La spec los excluye.

## R12. Volver a una versión anterior

**Decisión**:
1. `git revert` del commit con problema en `main`.
2. `git push origin main:production`.
3. El pipeline publica la versión anterior en menos de 15 minutos (SC-004).

**Razón**: `production` y `main` siguen describiendo lo que hay en producción.

**Alternativas**: el rollback desde las consolas de Cloud Run y Hosting es más rápido, pero deja
las ramas distintas de producción. Solo se usa en una emergencia y después se hace el revert
([quickstart.md §4](quickstart.md)).

## R13. Versiones de herramientas

| Herramienta | Versión | Nota |
|-------------|---------|------|
| `actions/checkout` | `v7` | kaizen usa `v4` |
| `actions/setup-go` | `v7` | `go-version: "1.26"`, igual que la imagen `golang:1.26-alpine` |
| `golangci/golangci-lint-action` | `v9` | `version: v2.15.0`, la misma versión que en local |
| `actions/setup-node` | `v7` | `node-version: 24` y `cache: npm` (kaizen: `v4` y Node 22) |
| `google-github-actions/auth` | `v3` | kaizen usa `v2`; la v3 corre en Node 24 |
| `google-github-actions/setup-gcloud` | `v3` | kaizen usa `v2`; la v3 corre en Node 24 |
| `firebase-tools` | `15.33.0` | Exacta, vía `npx` |

**Razón**: se usan las versiones mayores actuales de las mismas acciones de kaizen.

**Alternativas**:
- Dejar las versiones de kaizen: siguen corriendo, pero quedan en Node 20.
- Fijar las acciones por SHA: es más estricto de lo que este proyecto necesita.

## R14. Configuración inicial: manual, una sola vez

**Decisión**: el responsable, como owner del proyecto, ejecuta con `gcloud` los pasos de
[quickstart.md §1](quickstart.md):
- APIs;
- Firebase en el proyecto y su sitio de Hosting;
- Artifact Registry;
- cuentas de servicio y roles;
- WIF.

Los secrets se crean en GitHub y el dominio se conecta en la consola de Firebase.

**Razón**: la spec excluye la infraestructura como código.

## R15. Dominio `vamosjuntos.club`

**Decisión**: conectar el dominio raíz en *Hosting → Agregar dominio personalizado* y crear en el
registrador los registros que indique Firebase. Hoy el dominio apunta a `3.33.130.190` y
`15.197.148.33`. Mientras tanto, el sitio queda en `https://vamosjuntos-club.web.app`.

**Razón**: es lo que pide FR-008.

## Fuentes

- [Firebase Hosting + Cloud Run: regiones admitidas](https://firebase.google.com/docs/hosting/cloud-run)
- [Firebase Hosting: configuración completa](https://firebase.google.com/docs/hosting/full-config)
- [Firebase Hosting: caché y cookie `__session`](https://firebase.google.com/docs/hosting/manage-cache)
- [google-github-actions/auth](https://github.com/google-github-actions/auth) y [releases](https://github.com/google-github-actions/auth/releases)
- [google-github-actions/setup-gcloud releases](https://github.com/google-github-actions/setup-gcloud/releases)
- [action-hosting-deploy: roles de la cuenta de servicio](https://github.com/FirebaseExtended/action-hosting-deploy/blob/main/docs/service-account.md)
- [golangci-lint-action releases](https://github.com/golangci/golangci-lint-action/releases)
- [Fallas de `firebase deploy` con WIF (guía de terceros)](https://oneuptime.com/blog/post/2026-02-17-how-to-fix-firebase-deploy-failures-caused-by-workload-identity-federation-issues/view)
- Ejemplos del responsable: `C:\golang\kaizen-app\kaizensnsilvam-backend\.github\workflows\backend-deploy.yml`,
  `C:\golang\kaizen-app\front-kaizensnsilvam.com\.github\workflows\front-deploy.yml` y
  `C:\golang\kaizen-app\front-kaizensnsilvam.com\firebase.json`
