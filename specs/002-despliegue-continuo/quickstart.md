# Quickstart: configurar y validar el despliegue continuo

**Feature**: `002-despliegue-continuo` | **Date**: 2026-10-30

Esta guía tiene tres partes:
- la configuración inicial, que se hace una sola vez;
- el primer despliegue;
- los escenarios que prueban la spec.

Recursos e identidades en [data-model.md](data-model.md) y contratos en
[contracts/pipelines.md](contracts/pipelines.md). La estructura es la de `kaizen-app`.

## 0. Prerrequisitos

- `gcloud` autenticado como owner de `vamosjuntos-club`. La facturación ya está activa.
- Permisos de administrador en `snsilvam/vamosjuntos-backend` y `snsilvam/vamosjuntos-frontend`.
- Acceso al panel DNS del registrador de `vamosjuntos.club`.
- Los comandos son de Bash (en Windows, Git Bash). Variables comunes:

```bash
PROJECT_ID=vamosjuntos-club
PROJECT_NUMBER=54833988307
REGION=us-central1
POOL="projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool"
gcloud config set project "$PROJECT_ID"
```

## 1. Configuración inicial (una sola vez)

### 1.1 APIs

```bash
gcloud services enable run.googleapis.com artifactregistry.googleapis.com \
  iam.googleapis.com iamcredentials.googleapis.com sts.googleapis.com \
  cloudresourcemanager.googleapis.com firebase.googleapis.com firebasehosting.googleapis.com
```

### 1.2 Firebase y el sitio de Hosting

1. En <https://console.firebase.google.com>, usa *Agregar proyecto* y elige el proyecto de Google
   Cloud existente `vamosjuntos-club`. Analytics no hace falta.
2. En *Hosting*, pulsa *Comenzar* para crear el sitio por defecto `vamosjuntos-club`. No necesitas
   seguir los pasos de la CLI que muestra el asistente.

### 1.3 Repositorio de imágenes

```bash
gcloud artifacts repositories create containers --repository-format=docker --location="$REGION"
```

### 1.4 Cuenta de ejecución del backend

```bash
gcloud iam service-accounts create vamosjuntos-backend-run --display-name="Cloud Run: vamosjuntos-backend"
RUN_SA="vamosjuntos-backend-run@$PROJECT_ID.iam.gserviceaccount.com"
gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$RUN_SA" --role=roles/datastore.user
```

### 1.5 Workload Identity Federation (pool y dos proveedores, como en kaizen)

```bash
gcloud iam workload-identity-pools create github-pool --location=global --display-name="GitHub Actions"

gcloud iam workload-identity-pools providers create-oidc github-provider \
  --location=global --workload-identity-pool=github-pool \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.ref=assertion.ref" \
  --attribute-condition="assertion.repository == 'snsilvam/vamosjuntos-backend' && assertion.ref == 'refs/heads/production'"

gcloud iam workload-identity-pools providers create-oidc github-frontend-provider \
  --location=global --workload-identity-pool=github-pool \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.ref=assertion.ref" \
  --attribute-condition="assertion.repository == 'snsilvam/vamosjuntos-frontend' && assertion.ref == 'refs/heads/production'"
```

### 1.6 Cuentas de despliegue

```bash
# Backend
gcloud iam service-accounts create github-actions-cloud-run --display-name="GitHub Actions: vamosjuntos-backend"
DEPLOY_BE="github-actions-cloud-run@$PROJECT_ID.iam.gserviceaccount.com"
gcloud iam service-accounts add-iam-policy-binding "$DEPLOY_BE" --role=roles/iam.workloadIdentityUser \
  --member="principalSet://iam.googleapis.com/$POOL/attribute.repository/snsilvam/vamosjuntos-backend"
gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$DEPLOY_BE" --role=roles/run.admin
gcloud artifacts repositories add-iam-policy-binding containers --location="$REGION" \
  --member="serviceAccount:$DEPLOY_BE" --role=roles/artifactregistry.writer
gcloud iam service-accounts add-iam-policy-binding "$RUN_SA" \
  --member="serviceAccount:$DEPLOY_BE" --role=roles/iam.serviceAccountUser

# Frontend
gcloud iam service-accounts create github-actions-frontend --display-name="GitHub Actions: vamosjuntos-frontend"
DEPLOY_FE="github-actions-frontend@$PROJECT_ID.iam.gserviceaccount.com"
gcloud iam service-accounts add-iam-policy-binding "$DEPLOY_FE" --role=roles/iam.workloadIdentityUser \
  --member="principalSet://iam.googleapis.com/$POOL/attribute.repository/snsilvam/vamosjuntos-frontend"
for role in roles/firebasehosting.admin roles/run.viewer \
            roles/serviceusage.apiKeysViewer roles/serviceusage.serviceUsageConsumer; do
  gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$DEPLOY_FE" --role="$role"
done
```

### 1.7 Secret en GitHub

En `snsilvam/vamosjuntos-backend`, entra a *Settings → Secrets and variables → Actions → New
repository secret* y crea `SESSION_KEY` con la salida de `openssl rand -base64 32`. Genérala en tu
terminal y no la pegues en otro lado.

Los invitados ya no son un secret (enmienda v2.0.0): viven en Firestore. Ver el
[quickstart de 001, §5](../001-onboarding-sesion/quickstart.md).

### 1.8 Evento real

Crea el documento del evento en `vamosjuntos-club-db` siguiendo
[quickstart de 001, §4](../001-onboarding-sesion/quickstart.md). Sin él, la app muestra "El paseo
aún no está disponible".

## 2. Primer despliegue

El flujo de ramas es feature → `main` → `production`. El backend va primero, porque Hosting valida
que exista el servicio de Cloud Run del rewrite.

1. **Backend**:
   1. Integra `001-onboarding-sesion` y luego `002-despliegue-continuo` en `main`.
   2. Publica: `git push origin main:production`.
   3. Cuando la ejecución en *Actions* termine en verde, comprueba el servicio:

   ```bash
   URL=$(gcloud run services describe vamosjuntos-backend --region="$REGION" --format="value(status.url)")
   curl -i "$URL/api/onboarding/guests"   # 200 con la lista (o 404 event_not_found sin evento)
   ```

2. **Frontend**: integra `002-despliegue-continuo` en `main` y haz `git push origin main:production`.
   Cuando termine:

   ```bash
   curl -i https://vamosjuntos-club.web.app/api/onboarding/guests   # 200 y cache-control: no-store
   curl -sI https://vamosjuntos-club.web.app/inicio | grep -i cache-control   # no-cache
   ```

3. **Dominio**:
   1. En *Hosting → Agregar dominio personalizado*, escribe `vamosjuntos.club`.
   2. En el registrador, reemplaza los registros A actuales (`3.33.130.190` y `15.197.148.33`) por
      los que indique Firebase, y agrega el TXT de verificación.
   3. El certificado tarda de minutos a 24 horas.

## 3. Escenarios de validación

| # | Pasos | Resultado esperado | Cubre |
|---|-------|--------------------|-------|
| 1 | Cambio inocuo en el backend, integrado en `main` y llevado a `production` | Ejecución verde en menos de 10 min; la revisión activa usa la imagen `:<commit>` | US1-1, SC-001, FR-012 |
| 2 | Push a `main` y a otra rama en cada repo | No se dispara ningún workflow | US1-2, FR-002 |
| 3 | Cambio de texto visible en el frontend llevado a `production` | Ejecución verde. Al recargar `vamosjuntos.club`, aparece el texto, y el historial de Hosting muestra el commit | US2-1, FR-012, R8 (WIF + CLI) |
| 4 | En el sitio publicado: onboarding completo, recarga y "Cerrar sesión" | Funciona igual que en local; las llamadas van a `vamosjuntos.club/api/...` sin errores de CORS y la cookie `__session` es del mismo dominio | US2-2, FR-008 |
| 5 | Archivo mal formateado en el backend, llevado a `production` | Falla en el paso `gofmt` y la revisión activa no cambia; luego revierte | US3-1, FR-006, SC-002 |
| 6 | Error de tipos en el frontend, llevado a `production` | Falla en `npm run build` y el sitio no cambia; luego revierte | US3-2, FR-006 |
| 7 | Revisa el correo tras los escenarios 5 y 6 | Llega el aviso de GitHub de ejecución fallida | US3-3 |
| 8 | Dos push seguidos a `production` (p. ej. dos `git commit --allow-empty`) | La segunda ejecución espera a la primera; la revisión final usa el commit más reciente | FR-011 |
| 9 | En *Actions*, "Run workflow" sobre `production` | Se vuelve a desplegar el mismo commit | FR-003 |
| 10 | "Run workflow" eligiendo `main` | El job se salta (`if`) y no se publica nada | FR-002 |
| 11 | `git revert` en `main` del commit del escenario 1 y `git push origin main:production` | En menos de 15 min la revisión activa vuelve al código anterior | US4, SC-004 |
| 12 | Antes de abrir la app a la familia: borra el secret `SESSION_KEY` y usa "Run workflow" | Falla en "Validar secrets" sin desplegar y la revisión anterior sigue sirviendo. Luego créalo con un valor nuevo y relanza (eso cierra las sesiones de prueba) | Caso borde: falta un secreto |
| 13 | En el log del paso de despliegue | El valor de `SESSION_KEY` aparece como `***` | FR-010 |
| 14 | En ambos repos: `git grep -nIiE "private_key\|BEGIN .*PRIVATE KEY"` y `git grep -nI "<un código real>"` (prueba 2 o 3) | 0 resultados | SC-003, FR-010 |

## 4. Operación

- **Cambiar el catálogo**: edita `events/paseo-familiar-diciembre-2026/users` en la consola de
  Firestore. Aplica de inmediato, sin desplegar.
- **Rotar `SESSION_KEY`**: pon un valor nuevo de `openssl rand -base64 32` y usa "Run workflow".
  Todas las sesiones se cierran.
- **Emergencia, si no hay tiempo para esperar el pipeline**:
  - Cloud Run: *Manage traffic* hacia la revisión anterior.
  - Hosting: *Rollback* a la versión anterior.
  - Después, siempre el revert en `main` y el push a `production`.
