# Vamos Juntos .club: arquitectura y SDD

Repositorio de arquitectura de la app del **Paseo Familiar Diciembre 2026**. Aquí viven la
constitución y los artefactos de Spec-Driven Development (Spec Kit) de cada feature. No tiene
pipeline: el código está en los repos hermanos.

| Repo | Contenido | Despliegue |
|------|-----------|------------|
| `vamosjuntos.club` (este) | Constitución y `specs/` | Ninguno |
| [`vamosjuntos-backend`](https://github.com/snsilvam/vamosjuntos-backend) | API en Go | Push a `production` → Cloud Run |
| [`vamosjuntos-frontend`](https://github.com/snsilvam/vamosjuntos-frontend) | SPA en React + TypeScript | Push a `production` → Firebase Hosting |

- **Constitución vigente**: [.specify/memory/constitution.md](.specify/memory/constitution.md)
  (v3.0.0). Prevalece sobre cualquier spec, plan o tarea.
- **Producción**: <https://vamosjuntos.club>, con `/api/**` reescrito hacia Cloud Run.

## Estado de las specs (2026-10-31)

| Spec | Feature | Estado | Tareas | Pendiente |
|------|---------|--------|--------|-----------|
| [001](specs/001-onboarding-sesion/) | Onboarding y sesión | En producción | 72/72 | — (la sesión la reemplazó la 008) |
| [002](specs/002-despliegue-continuo/) | Despliegue continuo | En producción | 35/42 | Pruebas de robustez T034–T040 |
| [003](specs/003-card-proximo-viaje/) | Card "Próximo viaje" | En producción | 17/17 | — |
| [004](specs/004-banner-cuenta-regresiva/) | Banner de cuenta regresiva (con reloj en vivo) | En producción | 23/23 | — |
| [005](specs/005-modulo-transporte/) | Módulo Transporte | En producción | 22/22 | — |
| [006](specs/006-modulo-itinerario/) | Módulo Itinerario | En producción | 19/19 | — |
| [007](specs/007-modulo-compras/) | Módulo Compras | En producción | 21/21 | — |
| [008](specs/008-autenticacion-firebase/) | Autenticación con Firebase | En producción | 49/50 | T049: borrar el secret `SESSION_KEY` |
| [009](specs/009-confirmar-otro-sin-codigo/) | Confirmar a otro invitado sin su código | Integrada en `main` local, sin publicar | 15/16 | T015: subir, publicar y validar |

Todo el alcance de "Dominio de la V1" está construido y publicado. Cada `tasks.md` tiene sus
notas de implementación y la validación en producción.

**Versiones publicadas**: backend `625a9b3` (revisión `vamosjuntos-backend-00008-m4v`) y
frontend `2e9c2a9`. En los dos repos `main` = `production`.

## Pendientes

**Del responsable** (consola de GitHub):

- 008 T049: borrar el secret `SESSION_KEY` de `snsilvam/vamosjuntos-backend`.
- 002 T036 y T037: correo de ejecución fallida y prueba de la variable `VITE_FIREBASE_APP_ID`.
- 002 T039: los dos "Run workflow" (sobre `production` y sobre `main`).
- 002 T040: `git grep` con 2 o 3 códigos reales en ambos repos (la parte de llaves y `secrets.`
  ya dio 0).

**Con visto bueno** (publican a propósito en `production`): 002 T034, T035, T038 y la primera
parte de T039. El plan está en las notas de
[002/tasks.md](specs/002-despliegue-continuo/tasks.md).

## Cómo se trabaja

1. **Spec Kit**: `/speckit-specify` → `/speckit-clarify` → `/speckit-plan` → `/speckit-tasks` →
   `/speckit-implement`. Toda decisión que cambie el comportamiento se registra antes en
   `spec.md` (Principio VI).
2. **Código**: rama de feature en cada repo → `main` (integración, no despliega) →
   `git push origin main:production` (despliega). Nunca `gcloud run deploy` ni `firebase deploy`
   desde local.
3. **Validar en producción**: la familia ya usa la app. Las pruebas usan cuentas de correo de
   prueba vinculadas a invitados **pendientes**, con datos marcados "[PRUEBA]", y al final borran
   solo lo creado, por id.
