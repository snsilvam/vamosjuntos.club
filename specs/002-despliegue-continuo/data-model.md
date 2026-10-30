# Data Model: Despliegue Continuo

**Feature**: `002-despliegue-continuo` | **Date**: 2026-10-30

Esta feature no agrega colecciones a Firestore. Sus "entidades" son recursos de la nube, secrets de
GitHub y registros de las plataformas. Todo vive en el proyecto `vamosjuntos-club` (número
`54833988307`), región `us-central1`, con la misma estructura de `kaizen-app`.

## Ambiente de producción: recursos

| Recurso | Nombre | Tipo | Para qué |
|---------|--------|------|----------|
| Servicio | `vamosjuntos-backend` | Cloud Run (`us-central1`) | La API |
| Repositorio de imágenes | `containers` | Artifact Registry, Docker (`us-central1`) | Imágenes `vamosjuntos-backend:<commit>` |
| Sitio | `vamosjuntos-club` | Firebase Hosting | SPA en `vamosjuntos-club.web.app` y `vamosjuntos.club` |
| Base de datos | `vamosjuntos-club-db` | Firestore nativo (`nam5`) | Ya existe (spec 001); no cambia |
| Pool de identidades | `github-pool`, con los proveedores `github-provider` y `github-frontend-provider` | Workload Identity Federation | Ver "Federación" |

Hay un solo ambiente. La rama `production` es la única que despliega; `main` es la de integración.

## Identidades: cuentas de servicio

| Cuenta | La usa | Roles | Alcance |
|--------|--------|-------|---------|
| `vamosjuntos-backend-run@vamosjuntos-club.iam.gserviceaccount.com` | Cloud Run, en ejecución | `roles/datastore.user` | Proyecto |
| `github-actions-cloud-run@vamosjuntos-club.iam.gserviceaccount.com` | Pipeline del backend | `roles/run.admin` (desplegar y permitir acceso público) | Proyecto |
| | | `roles/artifactregistry.writer` | Solo el repositorio `containers` |
| | | `roles/iam.serviceAccountUser` | Solo `vamosjuntos-backend-run` |
| `github-actions-frontend@vamosjuntos-club.iam.gserviceaccount.com` | Pipeline del frontend | `roles/firebasehosting.admin`, `roles/run.viewer`, `roles/serviceusage.apiKeysViewer`, `roles/serviceusage.serviceUsageConsumer` | Proyecto |

Ninguna cuenta tiene llaves JSON. Las de despliegue solo se usan por suplantación desde WIF.

## Federación: GitHub → GCP

| Proveedor | Condición | Cuenta que puede suplantar |
|-----------|-----------|----------------------------|
| `github-provider` | `assertion.repository == 'snsilvam/vamosjuntos-backend' && assertion.ref == 'refs/heads/production'` | `github-actions-cloud-run` |
| `github-frontend-provider` | `assertion.repository == 'snsilvam/vamosjuntos-frontend' && assertion.ref == 'refs/heads/production'` | `github-actions-frontend` |

- **Emisor**: `https://token.actions.githubusercontent.com`.
- **Mapeo**: `google.subject=assertion.sub`, `attribute.repository=assertion.repository` y
  `attribute.ref=assertion.ref`.
- **Vínculo**: `roles/iam.workloadIdentityUser` para
  `principalSet://iam.googleapis.com/projects/54833988307/locations/global/workloadIdentityPools/github-pool/attribute.repository/snsilvam/<repo>`.

**Regla**: solo un workflow de `production` en el repo correcto obtiene credenciales, y solo para
su propia cuenta.

## Secreto

| Secret de GitHub (repo `vamosjuntos-backend`) | Variable en Cloud Run | Contenido | Lo crea |
|-----------------------------------------------|-----------------------|-----------|---------|
| — | — | Desde la spec 008 no hay secrets: la sesión la firma Firebase Authentication. `SESSION_KEY` se borra (tarea T049 de la 008) | — |

Variables públicas del frontend (repositorio `vamosjuntos-frontend`, *Variables* de Actions):
`VITE_FIREBASE_API_KEY`, `VITE_FIREBASE_AUTH_DOMAIN`, `VITE_FIREBASE_PROJECT_ID` y
`VITE_FIREBASE_APP_ID` (spec 008).

El catálogo de invitados ya no es un secret (enmienda v2.0.0). Vive en Firestore,
`events/{EVENT_ID}/users`, con las reglas de
[data-model de 001](../001-onboarding-sesion/data-model.md#invitado--firestore-eventsevent_idusersguestid);
lo lee el backend con `vamosjuntos-backend-run`.

**Reglas**:
- Los secretos nunca se escriben en archivos versionados ni en el YAML del workflow, que solo los
  referencia como `${{ secrets.X }}`. En los logs, GitHub los muestra como `***`.
- Llegan a Cloud Run como variables de entorno y se ven en la configuración de la revisión. Solo
  los ve quien tiene acceso al proyecto.
- Cambiar un secret en GitHub no afecta a producción hasta la siguiente ejecución del pipeline.

## Configuración de ejecución del backend

| Variable | Origen | Valor |
|----------|--------|-------|
| `EVENT_ID` | `env` del workflow | `paseo-familiar-diciembre-2026` |
| `FIRESTORE_DATABASE` | `env` del workflow | `vamosjuntos-club-db` |
| `PORT` | Cloud Run | `8080` |
| `GOOGLE_CLOUD_PROJECT` | `env` del workflow (`$PROJECT_ID`) | `vamosjuntos-club`; obligatoria desde la spec 008 (`aud` del ID token) |

## Ejecución de pipeline

Es una ejecución de GitHub Actions. No se guarda en nada propio.

| Campo | Dónde se ve |
|-------|-------------|
| Repositorio y commit | La ejecución en GitHub Actions (`github.sha`) |
| Resultado | Estado de la ejecución: exitosa, fallida o cancelada |
| Fecha y logs | La ejecución en GitHub Actions (retención de 90 días) |
| Qué se publicó | Backend: la etiqueta `:<commit>` de la imagen de la revisión activa. Frontend: el mensaje `<commit>` de la versión de Hosting |

### Estados

```text
en cola ──► en curso ──► exitosa   (producción = este commit)
   │            └──────► fallida   (producción intacta; correo de GitHub a quien hizo push)
   └──► cancelada                  (otra ejecución más nueva la reemplazó antes de empezar, FR-011)
```

Una ejecución en curso nunca se cancela (`cancel-in-progress: false`). Si falla cualquier paso
antes del despliegue, producción queda intacta. Si falla durante el despliegue, Cloud Run y Hosting
siguen sirviendo la versión anterior.
