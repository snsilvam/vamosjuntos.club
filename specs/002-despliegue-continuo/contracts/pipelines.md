# Contract: Pipelines, Cloud Run y enrutamiento

**Feature**: `002-despliegue-continuo`

Qué debe cumplir cada artefacto de despliegue. La estructura es la de `kaizen-app`. Los nombres de
recursos e identidades están en [data-model.md](../data-model.md) y el porqué de cada decisión, en
[research.md](../research.md).

## Reglas comunes a los dos workflows

| Aspecto | Contrato |
|---------|----------|
| Disparadores | `push` a `production` y `workflow_dispatch`, sin entradas (FR-002, FR-003) |
| Concurrencia | `group: deploy-production`, `cancel-in-progress: false` (FR-011) |
| Jobs | Uno solo, `deploy`, en `ubuntu-latest`, con `if: github.ref == 'refs/heads/production'` |
| Permisos | `contents: read` e `id-token: write`; nada más |
| Autenticación | `google-github-actions/auth@v3` con `workload_identity_provider` y `service_account` escritos en `with:`, como en kaizen. Sin llaves JSON (FR-009) |
| Secretos | Ninguno desde la spec 008. El frontend usa `vars.VITE_FIREBASE_*` (públicas). El catálogo vive en Firestore y no pasa por el pipeline. Nunca se imprimen ni se usa `set -x` (FR-010) |
| Orden | Compuertas → autenticación → publicación. Si un paso falla, la ejecución se detiene y no publica (FR-006) |
| Rastro | El commit (`github.sha`) queda en lo publicado (FR-012) |

## Pipeline del backend: `vamosjuntos-backend/.github/workflows/backend-deploy.yml`

**`env` del workflow**:

| Variable | Valor |
|----------|-------|
| `PROJECT_ID` | `vamosjuntos-club` |
| `REGION` | `us-central1` |
| `SERVICE` | `vamosjuntos-backend` |
| `IMAGE` | `us-central1-docker.pkg.dev/vamosjuntos-club/containers/vamosjuntos-backend` |
| `EVENT_ID` | `paseo-familiar-diciembre-2026` |
| `FIRESTORE_DATABASE` | `vamosjuntos-club-db` |

**Pasos**:

| # | Paso | Falla si |
|---|------|----------|
| 1 | `actions/checkout@v7` | |
| 2 | `actions/setup-go@v7` con `go-version: "1.26"` | |
| 3 | `test -z "$(gofmt -l .)"`, que lista los archivos sin formato | Algún archivo no tiene formato |
| 4 | `go vet ./...` | Hay hallazgos |
| 5 | `golangci/golangci-lint-action@v9` con `version: v2.15.0` | Hay hallazgos |
| 6 | `go test ./...` | Falla una prueba |
| 7 | (Quitado por la spec 008: el backend ya no tiene secrets) | |
| 8 | `auth@v3` → `projects/54833988307/locations/global/workloadIdentityPools/github-pool/providers/github-provider` con `github-actions-cloud-run@vamosjuntos-club.iam.gserviceaccount.com` | La rama no es `production` o falta un permiso |
| 9 | `google-github-actions/setup-gcloud@v3` | |
| 10 | `gcloud auth configure-docker us-central1-docker.pkg.dev --quiet` | |
| 11 | `docker build -t "$IMAGE:${{ github.sha }}" .` | No compila |
| 12 | `docker push "$IMAGE:${{ github.sha }}"` | |
| 13 | `gcloud run deploy`, con los flags de abajo | La revisión no arranca o falta un permiso |

**Resultado**: una revisión nueva de `vamosjuntos-backend`, con el 100 % del tráfico y la imagen
`…/vamosjuntos-backend:<commit>`.

### Cloud Run: `vamosjuntos-backend`

| Flag | Valor | Por qué |
|------|-------|---------|
| `--image` | `$IMAGE:${{ github.sha }}` | Una imagen por commit |
| `--region` | `us-central1` | Como en kaizen |
| `--platform` | `managed` | Como en kaizen |
| `--port` | `8080` | Como en kaizen |
| `--allow-unauthenticated` | | Hosting la invoca como cliente público |
| `--service-account` | `vamosjuntos-backend-run@vamosjuntos-club.iam.gserviceaccount.com` | Solo con `datastore.user` (R5) |
| `--max-instances` | `2` | Tope de costo |
| `--set-env-vars` | `"^;^GOOGLE_CLOUD_PROJECT=$PROJECT_ID;EVENT_ID=$EVENT_ID;FIRESTORE_DATABASE=$FIRESTORE_DATABASE"` | Separador `;` como en kaizen (R6); `GOOGLE_CLOUD_PROJECT` es obligatoria desde la spec 008 |

### Contenedor

| Aspecto | Contrato |
|---------|----------|
| Construcción | `golang:1.26-alpine`, `CGO_ENABLED=0 go build -trimpath -ldflags="-s -w"` desde la raíz del módulo |
| Imagen final | `gcr.io/distroless/static-debian12:nonroot`, con solo el binario; `EXPOSE 8080` |
| Arranque | Si la configuración es inválida, el proceso termina con código 1 y un mensaje en el log que no incluye valores secretos (spec 001). Cloud Run no le pasa tráfico |
| `.dockerignore` | `.git`, `.github`, `.env*` y `*.exe` |

## Pipeline del frontend: `vamosjuntos-frontend/.github/workflows/front-deploy.yml`

**`env` del workflow**: `PROJECT_ID: vamosjuntos-club`.

**Pasos**:

| # | Paso | Falla si |
|---|------|----------|
| 1 | `actions/checkout@v7` | |
| 2 | `actions/setup-node@v7` con `node-version: 24` y `cache: npm` | |
| 3 | `npm ci` | El lockfile no coincide |
| 3b | Validar `vars.VITE_FIREBASE_*` (spec 008) | Falta alguna variable |
| 4 | `npm run build` (`tsc -b && vite build`), con `VITE_FIREBASE_*` en `env` | Error de tipos o de compilación |
| 5 | `auth@v3` → `…/github-pool/providers/github-frontend-provider` con `github-actions-frontend@vamosjuntos-club.iam.gserviceaccount.com` | La rama no es `production` o falta un permiso |
| 6 | `npx --yes firebase-tools@15.33.0 deploy --only hosting --project "$PROJECT_ID" --message "${{ github.sha }}" --non-interactive` | Falta un permiso o no existe el servicio de Cloud Run del rewrite |

**Resultado**: una versión nueva del sitio `vamosjuntos-club`, con el commit como mensaje.

## Enrutamiento: `firebase.json` del frontend

Es la base del `firebase.json` de kaizen, más el rewrite de la API y el encabezado de caché:

```json
{
  "hosting": {
    "public": "dist",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [
      { "source": "/api/**", "run": { "serviceId": "vamosjuntos-backend", "region": "us-central1" } },
      { "source": "**", "destination": "/index.html" }
    ],
    "headers": [
      { "source": "!/@(assets|api)/**", "headers": [{ "key": "Cache-Control", "value": "no-cache" }] }
    ]
  }
}
```

**Garantías que la app necesita** (spec 001 y FR-008):
- El navegador solo habla con `vamosjuntos.club` (o `vamosjuntos-club.web.app`). No hay CORS.
- Hosting reenvía a Cloud Run solo la cookie `__session`, que es la de la sesión.
- Las respuestas de `/api/**` traen `Cache-Control: no-store` desde el backend, así que el CDN no
  las guarda.
- El rewrite de `/api/**` va antes que el de la SPA, para que nunca caiga en `index.html`.
