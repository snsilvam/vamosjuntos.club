# Implementation Plan: Despliegue Continuo

**Branch**: `002-despliegue-continuo` | **Date**: 2026-10-30 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/002-despliegue-continuo/spec.md`

## Summary

Cada repositorio de código tiene un workflow de GitHub Actions que, con un push a `production`,
valida el cambio y lo publica:

- **Backend**: compuertas de Go → imagen en Artifact Registry → Cloud Run.
- **Frontend**: build con chequeo de tipos → Firebase Hosting, que sirve la SPA y reenvía `/api/**`
  al servicio de Cloud Run en el mismo dominio.

Enfoque técnico: **la misma estructura de `kaizen-app`**, por decisión del responsable.

- Rama `production`.
- Un solo job por workflow.
- WIF con una cuenta de servicio y un proveedor por repo.
- Docker hacia Artifact Registry y `gcloud run deploy`.
- Secretos en los secrets del repo de GitHub, pasados a Cloud Run como variables de entorno.

Sobre esa base se agrega solo lo que la spec exige:

| Agregado | Por qué |
|----------|---------|
| Compuertas antes de desplegar | FR-004 a FR-006 |
| Relanzamiento manual | FR-003 |
| Despliegues de uno en uno | FR-011 |
| Imagen etiquetada con el commit | FR-012 |
| Frontend en Firebase Hosting y no en Cloud Run | Constitución |

Detalle en [research.md](research.md).

## Technical Context

**Language/Version**: YAML de GitHub Actions y Dockerfile. Los runners usan Go 1.26 y Node 24,
las mismas versiones del desarrollo local.

**Primary Dependencies**:
- Acciones: `actions/checkout@v7`, `actions/setup-go@v7`, `golangci/golangci-lint-action@v9`
  (golangci-lint v2.15.0), `actions/setup-node@v7`, `google-github-actions/auth@v3` y
  `google-github-actions/setup-gcloud@v3`.
- CLI: `gcloud`, Docker (preinstalado en el runner) y `firebase-tools` 15.33.0 vía `npx`.

**Storage**:
- Artifact Registry guarda las imágenes del backend.
- Los secrets del repo `vamosjuntos-backend` guardan `GUESTS_JSON` y `SESSION_KEY`.
- Firestore (`vamosjuntos-club-db`) no cambia.

**Testing**: el pipeline del backend ejecuta `go test` y el del frontend, el build. La validación
de los pipelines es manual con [quickstart.md](quickstart.md).

**Target Platform**:
- Runners `ubuntu-latest` de GitHub.
- Cloud Run en `us-central1` y Firebase Hosting, en el proyecto `vamosjuntos-club` (número
  `54833988307`).

**Project Type**: infraestructura de despliegue de la app web (dos repositorios).

**Performance Goals**: de push a producción en menos de 10 minutos (SC-001); un revert, en menos de
15 minutos (SC-004).

**Constraints**:
- Sin llaves JSON de cuentas de servicio.
- Ningún código de invitación en repos, archivos de workflow ni logs.
- Los códigos no pueden contener `;`, que es el separador de `--set-env-vars` (R6).
- Un solo ambiente y la cookie `__session`.
- Una región que admita los rewrites de Hosting hacia Cloud Run.

**Scale/Scope**: 2 workflows, 1 Dockerfile, 1 `firebase.json`, 2 secrets de GitHub y unos 12
recursos de GCP creados una vez.

**Estado actual de GCP** (verificado el 2026-10-30):
- Solo existe Firestore (`vamosjuntos-club-db`, `nam5`), con la facturación activa.
- Las APIs de Cloud Run, Artifact Registry, IAM y Firebase Hosting están deshabilitadas.
- No hay cuentas de servicio ni pools de WIF, y Firebase no está agregado al proyecto.
- `vamosjuntos.club` resuelve a `3.33.130.190` y `15.197.148.33`, que no son de Firebase.

Todo eso lo resuelve la configuración inicial ([quickstart.md §1](quickstart.md)).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Se evalúa contra la constitución **v1.2.0**, enmendada hoy: la rama de despliegue es `production`
y los secretos se guardan en los secrets de GitHub.

| Principio | Verificación | Estado |
|-----------|--------------|--------|
| I. Alcance cerrado | No agrega funciones a la app; un solo ambiente: producción. | ✅ |
| II. Acceso por invitación | El catálogo y los códigos viven solo en los secrets del repo del backend y en la configuración de Cloud Run. Nunca están en archivos versionados, y GitHub los enmascara en los logs. Hosting y la API comparten dominio, así que la cookie `__session` sigue siendo de primera parte. | ✅ |
| III. Simplicidad | La estructura de kaizen: un workflow y un job por repo, `gcloud` y `firebase-tools` directos, sin IaC y sin Secret Manager. | ✅ |
| IV. Compuertas | Backend: `gofmt`, `go vet`, `golangci-lint` y `go test` antes de construir la imagen. Frontend: `npm run build` (`tsc -b && vite build`). Si algo falla, el job se detiene antes de pedir credenciales de nube. | ✅ |
| V. Errores y HTTP | No cambia contratos. El `no-store` de la spec 001 evita que el CDN guarde respuestas de la API. | ✅ |
| VI. Cero código sombra | Solo FR-001 a FR-012. Se descartan vistas previas por PR, ambiente de pruebas, notificaciones extra, IaC y compuertas en PR. | ✅ |
| Despliegue continuo | Un push a `production` despliega y `main` no. WIF sin llaves JSON. Ningún secreto en texto plano en los workflows: llegan como variables de entorno de Cloud Run. | ✅ |
| Stack | Cloud Run y Firebase Hosting en el mismo proyecto, con rewrite de `/api/**` y región soportada. | ✅ |

**Re-check post-diseño**: ✅ El diseño de la Fase 1 no agrega recursos, roles ni pasos fuera de los
anteriores ([data-model.md](data-model.md)).

## Project Structure

### Documentation (this feature)

```text
specs/002-despliegue-continuo/
├── plan.md              # Este archivo
├── research.md          # Fase 0
├── data-model.md        # Fase 1: recursos, identidades, secretos y ejecuciones
├── quickstart.md        # Fase 1: configuración inicial, primer despliegue y validación
├── contracts/
│   └── pipelines.md     # Fase 1: contrato de cada pipeline, de Cloud Run y del enrutamiento
└── tasks.md             # Fase 2 (/speckit-tasks)
```

### Source Code

```text
vamosjuntos-backend/
├── .github/workflows/backend-deploy.yml   # Compuertas → imagen → Cloud Run (como en kaizen)
├── Dockerfile                             # golang:1.26-alpine → distroless static (nonroot)
├── .dockerignore
└── README.md                              # + sección "Despliegue"

vamosjuntos-frontend/
├── .github/workflows/front-deploy.yml     # Build → Firebase Hosting (como en kaizen)
├── firebase.json                          # dist, rewrite /api/** → Cloud Run, SPA y caché del HTML
├── .gitignore                             # + .firebase/ (caché local de la CLI)
└── README.md                              # + sección "Despliegue"
```

**Structure Decision**: los workflows llevan los mismos nombres de archivo que en kaizen
(`backend-deploy.yml` y `front-deploy.yml`). El Dockerfile y `firebase.json` van en la raíz de su
repo. `vamosjuntos.club` no recibe nada, porque no tiene pipeline (Principio III).

**Ramas**: el flujo es rama de feature → `main` (integración) → `production` (despliegue). El
trabajo de esta feature va en ramas `002-despliegue-continuo`:
- en el backend sale de `001-onboarding-sesion`, porque su `main` aún no tiene la feature 001;
- en el frontend sale de `main`.

Integrar en `main` no despliega. El primer push a `production` sí, así que solo se hace después de
la configuración inicial ([quickstart.md §1](quickstart.md)).

## Complexity Tracking

No hay violaciones de la constitución que justificar.
