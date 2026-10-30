# Feature Specification: Despliegue Continuo

**Feature Branch**: `002-despliegue-continuo`

**Created**: 2026-10-30

**Status**: Draft

**Input**: Derivado de la constitución v1.1.0 ("Despliegue Continuo"): "Pipelines de despliegue
de GitHub: el backend a Cloud Run y el frontend a Firebase Hosting. Un push a `main` despliega a
producción; las compuertas de calidad deben pasar antes de desplegar."

## User Scenarios & Testing *(mandatory)*

El actor de esta feature es el responsable del proyecto (quien desarrolla y publica la app).

### User Story 1 - Publicar el backend al integrar en main (Priority: P1)

El responsable integra un cambio en la rama `main` del repositorio del backend y, sin pasos
manuales, la nueva versión queda publicada en producción.

**Why this priority**: sin backend publicado ninguna feature funciona para la familia.

**Independent Test**: un cambio visible del backend (por ejemplo, un texto de respuesta) integrado
en `main` aparece en producción sin intervención manual.

**Acceptance Scenarios**:

1. **Given** un cambio que pasa las validaciones, **When** se integra en `main` del backend,
   **Then** la nueva versión queda en producción y el registro de la ejecución muestra éxito.
2. **Given** un cambio integrado en otra rama, **When** se sube, **Then** no se publica nada.

---

### User Story 2 - Publicar el frontend al integrar en main (Priority: P1)

El responsable integra un cambio en `main` del repositorio del frontend y la nueva versión del
sitio queda publicada en `vamosjuntos.club`.

**Why this priority**: es lo que la familia ve y usa.

**Independent Test**: un cambio visible del sitio integrado en `main` aparece al recargar
`vamosjuntos.club`.

**Acceptance Scenarios**:

1. **Given** un cambio que pasa las validaciones, **When** se integra en `main` del frontend,
   **Then** el sitio publicado muestra el cambio.
2. **Given** el sitio publicado, **When** el frontend llama al backend, **Then** lo hace en el
   mismo dominio, sin errores de origen cruzado.

---

### User Story 3 - Bloquear despliegues que no pasan las validaciones (Priority: P2)

Si un cambio no cumple las compuertas de calidad de la constitución, no llega a producción y el
responsable se entera.

**Why this priority**: protege a la familia de versiones rotas.

**Independent Test**: integrar un cambio con un error de formato o de compilación deja en
producción la versión anterior y marca la ejecución como fallida.

**Acceptance Scenarios**:

1. **Given** un cambio de backend con formato incorrecto, error de análisis estático o del linter,
   **When** se integra en `main`, **Then** el despliegue no ocurre y la ejecución queda fallida.
2. **Given** un cambio de frontend que no compila o tiene errores de tipos, **When** se integra en
   `main`, **Then** el despliegue no ocurre y la ejecución queda fallida.
3. **Given** una ejecución fallida, **When** termina, **Then** el responsable recibe la
   notificación estándar de la plataforma.

---

### User Story 4 - Volver a una versión anterior (Priority: P3)

Una versión publicada tiene un problema. El responsable revierte el cambio en `main` y la versión
anterior vuelve a producción por el mismo pipeline.

**Why this priority**: es poco frecuente, pero en diciembre no hay tiempo para arreglos largos.

**Independent Test**: revertir el último commit de `main` vuelve a dejar en producción el
comportamiento anterior.

**Acceptance Scenarios**:

1. **Given** una versión con problema en producción, **When** se revierte el commit en `main`,
   **Then** el pipeline publica la versión anterior.

---

### Edge Cases

- Dos cambios se integran en `main` casi al mismo tiempo: los despliegues no se solapan y queda
  publicada la versión del último commit.
- Falla la conexión con la nube a mitad del despliegue: la versión anterior sigue sirviendo y la
  ejecución queda fallida; se puede relanzar manualmente.
- Falta un secreto requerido (por ejemplo, el catálogo de invitados): el despliegue falla antes de
  publicar y la versión anterior sigue en producción.
- Se despliega el frontend antes que el backend con un cambio que depende de ambos: la
  coordinación es responsabilidad del desarrollador; cada pipeline es independiente.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Cada repositorio de código (backend y frontend) MUST tener su propio pipeline
  independiente.
- **FR-002**: Un push a `main` MUST disparar el pipeline del repositorio correspondiente; pushes a
  otras ramas MUST NOT publicar nada.
- **FR-003**: Cada pipeline MUST poder relanzarse manualmente sin nuevos commits.
- **FR-004**: El pipeline del backend MUST verificar formato, análisis estático y linter según el
  Principio IV, y ejecutar las pruebas automatizadas si existen, antes de publicar.
- **FR-005**: El pipeline del frontend MUST construir la versión de producción, con chequeo de
  tipos, antes de publicar.
- **FR-006**: Si cualquier validación falla, el pipeline MUST detenerse sin publicar y la versión
  en producción MUST seguir intacta.
- **FR-007**: El backend MUST publicarse en Cloud Run y el frontend en Firebase Hosting, según la
  constitución.
- **FR-008**: El sitio publicado MUST enviar las solicitudes del backend por el mismo dominio
  (`vamosjuntos.club`).
- **FR-009**: Los pipelines MUST autenticarse en la nube sin llaves de larga duración guardadas
  en el repositorio ni en los secretos de la plataforma.
- **FR-010**: El catálogo de invitados y sus códigos MUST llegar al backend desde un almacén de
  secretos; MUST NOT aparecer en los repositorios ni en los archivos de pipeline.
- **FR-011**: Los despliegues de un mismo repositorio MUST ejecutarse de uno en uno; si llega un
  commit más nuevo, la versión final publicada MUST ser la del commit más reciente.
- **FR-012**: Cada ejecución MUST dejar un registro consultable con su resultado y el commit
  publicado.

### Key Entities

- **Ejecución de pipeline**: un intento de validar y publicar un commit; tiene repositorio, commit,
  resultado (exitosa o fallida) y fecha.
- **Ambiente de producción**: el único ambiente de la V1, donde la familia usa la app.
- **Secreto**: un dato sensible (catálogo de invitados, códigos) que se inyecta al publicar y nunca
  se versiona.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un cambio integrado en `main` está en producción en menos de 10 minutos.
- **SC-002**: El 100 % de los commits con alguna validación fallida no llegan a producción.
- **SC-003**: Una búsqueda de códigos de invitación o llaves de nube en ambos repositorios devuelve
  0 resultados.
- **SC-004**: Revertir una versión con problema la saca de producción en menos de 15 minutos.
- **SC-005**: Publicar no requiere ningún paso manual después del push.

## Assumptions

- Los repositorios `vamosjuntos-backend` y `vamosjuntos-frontend` están en GitHub y el responsable
  tiene permisos de administrador.
- Todo vive en un único proyecto de nube, compartido por Firestore, el backend y el sitio.
- La configuración inicial de la nube (proyecto, permisos de despliegue, registro de imágenes,
  dominio) se hace una sola vez a mano, siguiendo instrucciones documentadas en el plan; no se
  automatiza con herramientas de infraestructura como código.
- Los ejemplos de pipelines de Go y React que tiene el responsable se usan como base en el plan.
- La notificación de fallos es la que ofrece la plataforma por defecto (correo); no se agregan
  otros canales.
