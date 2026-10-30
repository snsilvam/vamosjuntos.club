---

description: "Lista de tareas de la feature 002-despliegue-continuo"
---

# Tasks: Despliegue Continuo

**Input**: Design documents from `/specs/002-despliegue-continuo/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/pipelines.md](contracts/pipelines.md),
[quickstart.md](quickstart.md)

**Tests**: no hay pruebas automáticas nuevas. Las compuertas (`gofmt`, `go vet`, `golangci-lint`,
`go test` y `npm run build`) son parte de los propios pipelines, y cada historia se valida con los
escenarios de [quickstart.md §3](quickstart.md). Los workflows se revisan con `actionlint` antes
del primer push.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

Las rutas son relativas a `C:\golang\vamosjuntos-app\`:

- `vamosjuntos-backend/` y `vamosjuntos-frontend/` reciben los archivos de despliegue.
- `vamosjuntos.club/` solo recibe la documentación (sin pipeline).

Reglas que aplican a todas las tareas:

- **Estructura de kaizen-app**: rama `production`, un job `deploy`, `auth` con
  `workload_identity_provider` y `service_account` en `with:`, Docker hacia Artifact Registry y
  `gcloud run deploy`. Referencia:
  - `C:\golang\kaizen-app\kaizensnsilvam-backend\.github\workflows\backend-deploy.yml`;
  - `C:\golang\kaizen-app\front-kaizensnsilvam.com\.github\workflows\front-deploy.yml`.
- **Secretos**: `SESSION_KEY` solo aparece como `${{ secrets.SESSION_KEY }}` en el `env` del paso
  que la usa (enmienda v2.0.0: el catálogo vive en Firestore). Nunca se imprimen, nunca se usa `set -x` y nunca se escriben en archivos
  versionados.
- **Etiquetas en las tareas**:
  - **(responsable)**: la hace el responsable, porque requiere la consola, GitHub o los códigos
    reales.
  - **(visto bueno)**: crea recursos en GCP o publica en producción, así que Claude la ejecuta solo
    después de que el responsable lo apruebe.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: ramas de trabajo en ambos repos.

- [X] T001 Crear la rama `002-despliegue-continuo` en vamosjuntos-backend a partir de `001-onboarding-sesion`, porque el `main` del backend aún no tiene la feature 001 (plan.md, "Ramas")
- [X] T002 [P] Crear la rama `002-despliegue-continuo` en vamosjuntos-frontend a partir de `main`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: configuración inicial de GCP y GitHub, una sola vez ([quickstart.md §1](quickstart.md)).
Bloquea los despliegues de todas las historias, pero no la escritura de los archivos de las
fases 3 y 4.

**⚠️ CRITICAL**: ningún push a `production` antes de terminar esta fase.

- [X] T003 (visto bueno) Habilitar las APIs de §1.1 en `vamosjuntos-club`: `run`, `artifactregistry`, `iam`, `iamcredentials`, `sts`, `cloudresourcemanager`, `firebase` y `firebasehosting`
- [X] T004 (responsable) Agregar Firebase al proyecto existente `vamosjuntos-club` desde la consola y crear el sitio de Hosting por defecto `vamosjuntos-club` (§1.2) (depende de T003)
- [X] T005 [P] (visto bueno) Crear el repositorio Docker `containers` de Artifact Registry en `us-central1` (§1.3) (depende de T003)
- [X] T006 [P] (visto bueno) Crear la cuenta `vamosjuntos-backend-run` y darle `roles/datastore.user` en el proyecto (§1.4) (depende de T003)
- [X] T007 [P] (visto bueno) Crear el pool `github-pool` y los proveedores OIDC `github-provider` (`assertion.repository == 'snsilvam/vamosjuntos-backend' && assertion.ref == 'refs/heads/production'`) y `github-frontend-provider` (lo mismo para `snsilvam/vamosjuntos-frontend`), con el mapeo de §1.5 (depende de T003)
- [X] T008 (visto bueno) Crear `github-actions-cloud-run` con los roles de §1.6 (depende de T005, T006 y T007):
  - `roles/iam.workloadIdentityUser` para el `principalSet` de `snsilvam/vamosjuntos-backend`;
  - `roles/run.admin` en el proyecto;
  - `roles/artifactregistry.writer` en `containers`;
  - `roles/iam.serviceAccountUser` sobre `vamosjuntos-backend-run`.
- [X] T009 (visto bueno) Crear `github-actions-frontend` con los roles de §1.6 (depende de T004 y T007):
  - `roles/iam.workloadIdentityUser` para el `principalSet` de `snsilvam/vamosjuntos-frontend`;
  - `roles/firebasehosting.admin`, `roles/run.viewer`, `roles/serviceusage.apiKeysViewer` y
    `roles/serviceusage.serviceUsageConsumer` en el proyecto.
- [X] T010 (responsable) Crear en `snsilvam/vamosjuntos-backend` el repository secret `SESSION_KEY` con la salida de `openssl rand -base64 32` (§1.7). Si ya se creó `GUESTS_JSON`, borrarlo: el catálogo vive en Firestore (enmienda v2.0.0)
- [X] T011 (responsable) Crear el documento `events/paseo-familiar-diciembre-2026` en la base `vamosjuntos-club-db` (quickstart de 001, §4)
- [X] T012 Verificar la configuración con `gcloud` (depende de T003 a T009):
  - `artifacts repositories describe containers`;
  - `iam service-accounts list`;
  - `iam workload-identity-pools providers list --workload-identity-pool=github-pool`;
  - las políticas de IAM de las tres cuentas.

**Checkpoint**: la nube y GitHub están listos para recibir los pipelines.

---

## Phase 3: User Story 1 - Publicar el backend al integrar en production (Priority: P1) 🎯 MVP

**Goal**: un push a `production` del backend valida el código y deja la nueva versión en Cloud Run.

**Independent Test**: escenarios 1 y 2 de [quickstart.md §3](quickstart.md).

### Implementation for User Story 1

- [X] T013 [P] [US1] Crear vamosjuntos-backend/Dockerfile (contracts/pipelines.md, "Contenedor"):
  - etapa `golang:1.26-alpine` que copia `go.mod` y `go.sum`, ejecuta `go mod download`, copia el
    código y compila con `CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath -ldflags="-s -w" -o /app/server .`;
  - etapa final `gcr.io/distroless/static-debian12:nonroot` con solo `/app/server`, `EXPOSE 8080` y
    `CMD ["/app/server"]`;
  - misma forma que el Dockerfile de kaizen.
- [X] T014 [P] [US1] Crear vamosjuntos-backend/.dockerignore con `.git`, `.github`, `.env*` y `*.exe`
- [X] T015 [US1] Crear vamosjuntos-backend/.github/workflows/backend-deploy.yml con la cabecera del contrato:
  - `name`;
  - `on: push: branches: [production]` y `workflow_dispatch`;
  - `concurrency: { group: deploy-production, cancel-in-progress: false }`;
  - el bloque `env` (`PROJECT_ID`, `REGION`, `SERVICE`, `IMAGE`, `EVENT_ID=paseo-familiar-diciembre-2026`, `FIRESTORE_DATABASE=vamosjuntos-club-db`);
  - un job `deploy` en `ubuntu-latest` con `if: github.ref == 'refs/heads/production'` y permisos `contents: read` e `id-token: write`.
- [X] T016 [US1] Agregar a vamosjuntos-backend/.github/workflows/backend-deploy.yml las compuertas, en orden (depende de T015):
  1. `actions/checkout@v7`;
  2. `actions/setup-go@v7` con `go-version: "1.26"`;
  3. `test -z "$(gofmt -l .)"`, que lista los archivos si falla;
  4. `go vet ./...`;
  5. `golangci/golangci-lint-action@v9` con `version: v2.15.0`;
  6. `go test ./...`.
- [X] T017 [US1] Agregar a vamosjuntos-backend/.github/workflows/backend-deploy.yml los pasos de publicación (depende de T016):
  1. "Validar secrets": `test -n` sobre `SESSION_KEY` desde `secrets.*`, con un mensaje que solo nombra el secret faltante (antes también `GUESTS_JSON`; lo quitó la enmienda v2.0.0);
  2. `google-github-actions/auth@v3` con `workload_identity_provider: projects/54833988307/locations/global/workloadIdentityPools/github-pool/providers/github-provider` y `service_account: github-actions-cloud-run@vamosjuntos-club.iam.gserviceaccount.com`;
  3. `google-github-actions/setup-gcloud@v3`;
  4. `gcloud auth configure-docker us-central1-docker.pkg.dev --quiet`;
  5. `docker build` y `docker push` de `$IMAGE:${{ github.sha }}`.
- [X] T018 [US1] Agregar a vamosjuntos-backend/.github/workflows/backend-deploy.yml el paso `gcloud run deploy "$SERVICE"` con los flags de contracts/pipelines.md (depende de T017):
  - `--image "$IMAGE:${{ github.sha }}" --region "$REGION" --platform managed --port 8080`;
  - `--allow-unauthenticated`;
  - `--service-account vamosjuntos-backend-run@vamosjuntos-club.iam.gserviceaccount.com`;
  - `--max-instances 2`;
  - `--set-env-vars "^;^EVENT_ID=$EVENT_ID;FIRESTORE_DATABASE=$FIRESTORE_DATABASE;SESSION_KEY=$SESSION_KEY"`, con el secreto en el `env` del paso.
- [X] T019 [US1] Revisar vamosjuntos-backend/.github/workflows/backend-deploy.yml con `go run github.com/rhysd/actionlint/cmd/actionlint@latest` y confirmar que `gofmt -l .`, `go vet ./...`, `golangci-lint run` y `go test ./...` pasan en local (depende de T018)
- [X] T020 [P] [US1] Agregar una sección "Despliegue" a vamosjuntos-backend/README.md que explique:
  - el flujo feature → `main` → `production`;
  - qué hace el pipeline;
  - el secret `SESSION_KEY` (sin valor) y que los invitados viven en Firestore;
  - cómo relanzar, cambiar el catálogo y revertir (quickstart §4).
- [X] T021 [US1] Hacer commit de T013 a T020 en la rama `002-despliegue-continuo` del backend
- [X] T022 [US1] (visto bueno) Primer despliegue del backend (depende de la Fase 2 y de T021):
  1. integrar `001-onboarding-sesion` y `002-despliegue-continuo` en `main`;
  2. `git push origin main` y `git push origin main:production`;
  3. seguir la ejecución en *Actions*;
  4. comprobar con `curl -i "$URL/api/onboarding/guests"` (quickstart §2.1).
- [ ] T023 [US1] Validar los escenarios 1 y 2 de quickstart §3 en el backend:
  - la revisión activa usa la imagen `:<commit>`;
  - un push a `main` no dispara nada.

**Checkpoint**: el backend está en producción y se publica solo desde `production`.

---

## Phase 4: User Story 2 - Publicar el frontend al integrar en production (Priority: P1)

**Goal**: un push a `production` del frontend construye la SPA y la publica en Firebase Hosting, con
`/api/**` hacia Cloud Run en el mismo dominio.

**Independent Test**: escenarios 3 y 4 de [quickstart.md §3](quickstart.md).

### Implementation for User Story 2

- [X] T024 [P] [US2] Crear vamosjuntos-frontend/firebase.json igual al de contracts/pipelines.md:
  - `public: dist` y el `ignore` de kaizen;
  - rewrite `/api/**` → `run: { serviceId: "vamosjuntos-backend", region: "us-central1" }` antes de `**` → `/index.html`, sin `pinTag`;
  - header `Cache-Control: no-cache` para `!/@(assets|api)/**`.
- [X] T025 [P] [US2] Agregar `.firebase/` a vamosjuntos-frontend/.gitignore
- [X] T026 [US2] Crear vamosjuntos-frontend/.github/workflows/front-deploy.yml:
  - cabecera:
    - `on: push: branches: [production]` y `workflow_dispatch`;
    - `concurrency: { group: deploy-production, cancel-in-progress: false }`;
    - `env: PROJECT_ID: vamosjuntos-club`;
    - un job `deploy` en `ubuntu-latest` con `if: github.ref == 'refs/heads/production'` y
      permisos `contents: read` e `id-token: write`;
  - pasos:
    1. `actions/checkout@v7`;
    2. `actions/setup-node@v7` con `node-version: 24` y `cache: npm`;
    3. `npm ci`;
    4. `npm run build`.
- [X] T027 [US2] Agregar a vamosjuntos-frontend/.github/workflows/front-deploy.yml la publicación (depende de T026):
  1. `google-github-actions/auth@v3` con `workload_identity_provider: projects/54833988307/locations/global/workloadIdentityPools/github-pool/providers/github-frontend-provider` y `service_account: github-actions-frontend@vamosjuntos-club.iam.gserviceaccount.com`;
  2. `npx --yes firebase-tools@15.33.0 deploy --only hosting --project "$PROJECT_ID" --message "${{ github.sha }}" --non-interactive`.
- [X] T028 [US2] Revisar vamosjuntos-frontend/.github/workflows/front-deploy.yml con `actionlint` y confirmar que `npm run build` pasa en local (depende de T027)
- [X] T029 [P] [US2] Agregar una sección "Despliegue" a vamosjuntos-frontend/README.md con el flujo feature → `main` → `production`, qué hace el pipeline, el rewrite de `/api/**` y cómo relanzar o revertir
- [X] T030 [US2] Hacer commit de T024 a T029 en la rama `002-despliegue-continuo` del frontend
- [X] T031 [US2] (visto bueno) Primer despliegue del frontend (depende de T022 y T030):
  1. integrar `002-despliegue-continuo` en `main`;
  2. `git push origin main` y `git push origin main:production`;
  3. comprobar con los `curl` de quickstart §2.2: `/api/...` responde `no-store` y `/inicio` responde `no-cache`.
  - Si `firebase-tools` no acepta las credenciales federadas, detenerse y revisar research R8 antes
    de cambiar el diseño.
- [ ] T032 [US2] (responsable) Conectar `vamosjuntos.club` en *Hosting → Agregar dominio personalizado* y cambiar los registros DNS en el registrador (quickstart §2.3)
- [ ] T033 [US2] Validar los escenarios 3 y 4 de quickstart §3:
  - el texto nuevo aparece al recargar;
  - el commit queda en el historial de Hosting;
  - el onboarding completo funciona en el dominio, sin CORS y con `__session` de primera parte.

**Checkpoint**: la app completa está en producción en `vamosjuntos.club`.

---

## Phase 5: User Story 3 - Bloquear despliegues que no pasan las validaciones (Priority: P2)

**Goal**: comprobar que un commit roto no llega a producción y que el responsable se entera. Las
compuertas ya existen desde T016 y T026; esta fase las prueba.

**Independent Test**: escenarios 5, 6, 7 y 12 de [quickstart.md §3](quickstart.md).

- [ ] T034 [US3] (visto bueno) Escenario 5: llevar a `production` un archivo de Go mal formateado y confirmar que la ejecución falla en `gofmt` y la revisión activa no cambia. Revertir después
- [ ] T035 [US3] (visto bueno) Escenario 6: llevar a `production` un error de tipos del frontend y confirmar que falla en `npm run build` y el sitio no cambia. Revertir después
- [ ] T036 [US3] (responsable) Escenario 7: confirmar que llegó el correo de GitHub por las ejecuciones fallidas de T034 y T035
- [ ] T037 [US3] (responsable) Escenario 12, antes de abrir la app a la familia:
  1. borrar el secret `SESSION_KEY` y usar "Run workflow";
  2. confirmar que falla en "Validar secrets" sin desplegar;
  3. crearlo de nuevo con un valor nuevo y relanzar.

**Checkpoint**: probado que un cambio roto no llega a producción.

---

## Phase 6: User Story 4 - Volver a una versión anterior (Priority: P3)

**Goal**: revertir un commit saca su versión de producción por el mismo pipeline.

**Independent Test**: escenario 11 de [quickstart.md §3](quickstart.md).

- [ ] T038 [US4] (visto bueno) Escenario 11: `git revert` en `main` del commit inocuo del escenario 1, `git push origin main:production`, y confirmar que en menos de 15 minutos la revisión activa vuelve al código anterior

**Checkpoint**: probado el camino de rollback.

---

## Phase 7: Polish & Cross-Cutting Concerns

- [ ] T039 (visto bueno) Escenarios 8, 9 y 10 de quickstart §3:
  - dos push seguidos a `production`: el segundo espera y queda publicado el último commit (FR-011);
  - "Run workflow" sobre `production` vuelve a desplegar (FR-003);
  - "Run workflow" desde `main` salta el job (FR-002).
- [ ] T040 [P] Escenarios 13 y 14 de quickstart §3 (SC-003, FR-010):
  - los logs muestran `***` en lugar de los secretos;
  - `git grep` de llaves privadas y de 2 o 3 códigos reales devuelve 0 en ambos repos. Los códigos
    los proporciona el responsable en su terminal; no se escriben en ningún archivo.
- [X] T041 Quitar el Sync Impact Report de vamosjuntos.club/.specify/memory/constitution.md (v1.2.0) y hacer commit en `vamosjuntos.club` de la enmienda y de specs/002-despliegue-continuo/
- [X] T042 Registrar en este tasks.md una sección "Implementation Notes" con los resultados de la validación y cualquier desviación del plan

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Fase 1)**: no depende de nada.
- **Foundational (Fase 2)**: depende de la Fase 1 solo por orden. Bloquea los despliegues (T022 y
  T031), pero no la escritura de archivos.
- **US1 (Fase 3)**: T013 a T021 pueden hacerse en paralelo con la Fase 2. T022 necesita la Fase 2
  completa.
- **US2 (Fase 4)**: T024 a T030 pueden hacerse en paralelo con US1. T031 necesita T022, porque
  Hosting valida que el servicio de Cloud Run exista.
- **US3 (Fase 5)**: depende de T022 y T031.
- **US4 (Fase 6)**: depende de T023 (el commit inocuo del escenario 1).
- **Polish (Fase 7)**: depende de las fases 3 a 6. T041 y T042 pueden hacerse en cualquier momento
  después de la fase en curso.

### Within Each User Story

- Los pasos del mismo workflow son secuenciales, porque es un solo archivo: cabecera → compuertas
  → publicación → revisión con `actionlint`.
- El commit va antes del despliegue, y el despliegue antes de la validación.

### Parallel Opportunities

- T001 y T002.
- T005, T006 y T007, después de T003.
- T013, T014 y T020 (backend) con T024, T025 y T029 (frontend).
- La escritura de archivos de las fases 3 y 4 mientras el responsable hace T004, T010 y T011.

---

## Parallel Example: archivos de ambos repos

```bash
Task: "Crear vamosjuntos-backend/Dockerfile (T013)"
Task: "Crear vamosjuntos-backend/.dockerignore (T014)"
Task: "Crear vamosjuntos-frontend/firebase.json (T024)"
Task: "Agregar .firebase/ a vamosjuntos-frontend/.gitignore (T025)"
```

## Parallel Example: configuración de GCP (después de T003)

```bash
Task: "Crear el repositorio containers (T005)"
Task: "Crear la cuenta vamosjuntos-backend-run (T006)"
Task: "Crear github-pool y sus dos proveedores (T007)"
```

---

## Implementation Strategy

### MVP First (US1 + US2)

1. Fase 1 y archivos de las fases 3 y 4 (T013 a T021 y T024 a T030), todo en ramas y sin
   publicar.
2. Fase 2, con los pasos del responsable en paralelo.
3. T022: backend en producción. Validar con T023.
4. T031 y T032: frontend y dominio. Validar con T033.
5. **STOP**: con esto la familia ya puede usar el onboarding en `vamosjuntos.club`.

### Incremental Delivery

1. Fases 1 y 2 → la nube está lista.
2. US1 → backend publicado desde `production`.
3. US2 → sitio publicado; la app completa está en línea (MVP).
4. US3 → probado que los cambios rotos no pasan.
5. US4 → probado el rollback.
6. Polish → concurrencia, relanzamiento, secretos y commit de la documentación.

---

## Notes

- Las tareas (visto bueno) crean recursos o publican en producción. Claude las ejecuta solo
  después de que el responsable lo apruebe en la conversación.
- Las tareas (responsable) requieren la consola de Firebase, GitHub, el registrador o los códigos
  reales, que Claude nunca recibe.
- Los workflows no deben imprimir secretos. Si un paso necesita depurarse, se usa `echo` de
  variables no secretas, nunca `set -x`.
- Cada escenario que rompe algo a propósito (T034, T035 y T037) termina revirtiendo, para dejar
  producción como estaba.

## Implementation Notes (2026-10-30)

**Hecho, sin publicar nada** (commits locales en las ramas `002-despliegue-continuo`; no hay
push):

- **vamosjuntos-backend** `cf4939d` (sale de `001-onboarding-sesion`): `Dockerfile`,
  `.dockerignore`, `.github/workflows/backend-deploy.yml` y la sección "Despliegue" del README.
- **vamosjuntos-frontend** `a36ce6b` (sale de `main`): `firebase.json`, `.gitignore` (+
  `.firebase/`), `.github/workflows/front-deploy.yml` y un README del proyecto en lugar del de la
  plantilla de Vite.

**Validación local**:

- **Backend**: `gofmt -l .` vacío, `go vet` sin hallazgos, `golangci-lint` v2.15.0 con 0 issues y
  `go test` en verde. El binario compila para Linux estático con los mismos flags del Dockerfile
  (`CGO_ENABLED=0 GOOS=linux GOARCH=amd64 -trimpath -ldflags="-s -w"`).
- **Frontend**: `npm run build` en verde. Vite deja los archivos con hash en `dist/assets/`, que es
  lo que excluye la regla de caché, y `firebase.json` es JSON válido.

**Desviaciones**:

- **T019 y T028: `actionlint` no corrió.** El modo automático negó
  `go run github.com/rhysd/actionlint/cmd/actionlint@latest` porque descarga y ejecuta código
  externo. Los workflows se revisaron a mano contra contracts/pipelines.md. Para correrlo:
  `go run github.com/rhysd/actionlint/cmd/actionlint@latest` en cada repo, o permitir el comando.
  Si el YAML tuviera un error, aparecería en la primera ejecución en GitHub, antes de publicar
  nada.
- **No se probó la imagen Docker en local**, porque Docker no está instalado. La primera ejecución
  del pipeline la construye.

**Pendiente**:

- **Requiere visto bueno** (crea recursos en GCP o publica): T003, T005 a T009, T022, T031, T034,
  T035, T038 y T039.
- **Responsable**: T004 (Firebase), T010 (secrets), T011 (evento), T032 (dominio), T036 y T037.
- **Después de lo anterior**: T012, T023, T033 y T040. T041 se hizo junto con estas notas.
