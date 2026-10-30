# Constitución de Vamos Juntos .club

Herramienta para la gestión de paseos familiares y eventos. Su propósito es permitir que los
usuarios se autentiquen y gestionen su participación en un evento con tiempos y espacios
específicos. La V1 cubre exclusivamente el "Paseo Familiar Diciembre 2026".

## Core Principles

### I. Alcance Cerrado de la V1

- El sistema MUST manejar un único evento fijo: **"Paseo Familiar Diciembre 2026"**. No se
  construyen catálogos, CRUD ni pantallas de administración de eventos en la V1.
- El evento MUST leerse desde la colección `events` de Firestore, y su lista de invitados desde
  la subcolección `events/{EVENT_ID}/users`. El responsable del proyecto crea y mantiene ambos a
  mano; la aplicación MUST NOT crear ni editar eventos ni invitados.
- Cada evento MUST admitir como máximo **25 usuarios**. Ningún flujo puede superar ese tope.
- Los módulos permitidos en la V1 son exactamente los listados en "Dominio de la V1". Cualquier
  otro módulo requiere enmendar esta constitución.

**Razón:** la V1 tiene una fecha dura (23 de diciembre de 2026) y un público conocido; un
alcance cerrado es lo que permite llegar a tiempo.

### II. Acceso Solo por Invitación

- Todo flujo de la aplicación, salvo el propio onboarding, MUST exigir un usuario con sesión
  activa que esté invitado al evento. El backend MUST validarlo en cada request; ocultar
  pantallas en el frontend no cuenta como control de acceso.
- El onboarding MUST seguir este orden: (1) el usuario selecciona su nombre de la lista de
  invitados, (2) ingresa su código de invitación, (3) el sistema le pregunta si confirma
  asistencia. Si confirma, su nombre MUST dejar de aparecer en la lista de onboarding.
- La lista de invitados y sus códigos MUST provenir de Firestore (`events/{EVENT_ID}/users`, un
  documento por invitado). Solo el backend los lee, con su cuenta de servicio. Las reglas de
  seguridad de Firestore MUST negar todo acceso desde clientes. Los códigos reales MUST NOT
  versionarse en ningún repositorio (incluido este) ni salir del backend en respuestas o logs.

**Razón:** la app maneja datos de una familia; nadie fuera de la lista debe poder entrar.

### III. Simplicidad y Estructura Plana

- Se MUST priorizar simplicidad y legibilidad sobre la abstracción. Prohibido "Clean
  Architecture", capas hexagonales o patrones complejos sin un problema real que resolver.
- Se MUST preferir composición sobre herencia y funciones simples sobre jerarquías complejas.
- Estructura de proyectos (directorios hermanos):
  - `/vamosjuntos-frontend`: aplicación React + TypeScript, con pipeline de despliegue.
  - `/vamosjuntos-backend`: API en Go, con pipeline de despliegue.
  - `/vamosjuntos.club`: este repositorio; arquitectura y artefactos SDD, sin pipeline.
- Firestore se usa tal como lo documenta Google oficialmente (cliente oficial, colecciones y
  documentos simples), sin ORMs ni capas de repositorio genéricas.

**Razón:** es un proyecto pequeño con un único evento; cada abstracción extra es costo puro.

### IV. Código Idiomático por Capa

**Backend (Go):**
- Idiomático ante todo: se siguen Effective Go y Go Code Review Comments. Ante un conflicto
  entre cualquier paradigma (p. ej. programación funcional) y el estilo idiomático de Go,
  prevalece Go.
- `struct` con métodos para entidades y servicios con estado o dependencias; funciones simples
  para lógica sin estado.
- La lógica de negocio MUST quedar aislada de efectos secundarios (I/O, base de datos, red) en
  funciones puras donde sea natural, para que sea testeable sin Firestore.
- Composición con embedding e interfaces; no simular herencia.
- Interfaces pequeñas (idealmente 1-3 métodos), definidas donde se consumen. "Acepta
  interfaces, devuelve structs".
- Errores explícitos: devolver y envolver (`fmt.Errorf("...: %w", err)`). `panic` MUST NOT
  usarse para flujo de control.
- `context.Context` como primer parámetro en toda operación de I/O; toda goroutine MUST tener
  un ciclo de vida claro (sin fugas).
- Inyección de dependencias mediante constructores `NewXxx`; sin estado global mutable.
- No introducir abstracciones genéricas ni utilidades funcionales propias (map/filter/reduce,
  monads, etc.) salvo que eliminen duplicación real.
- `gofmt`, `go vet` y `golangci-lint` MUST pasar sin errores en el pipeline de
  `/vamosjuntos-backend`; si alguno falla, no se despliega.

**Frontend (React + TypeScript):**
- Solo componentes funcionales y Hooks; componentes de clase MUST NOT usarse.
- Estilos MUST hacerse con Tailwind CSS.
- Nomenclatura: `camelCase` para funciones y variables; `PascalCase` para componentes,
  interfaces y tipos.
- El build de producción (incluido el chequeo de tipos de TypeScript) MUST pasar en el
  pipeline de `/vamosjuntos-frontend`; si falla, no se despliega.

### V. Errores Amigables y HTTP Semántico

- **UI:** el usuario final MUST NOT ver errores crudos, códigos internos ni *stack traces*.
  Todo error técnico MUST traducirse a un mensaje amigable en español
  (ej.: "El evento aún no inicia según su fecha de inicio.").
- **Backend:** toda respuesta MUST usar códigos HTTP semánticos, como mínimo: `400` petición
  inválida, `401` no autenticado, `403` autenticado pero no invitado/no autorizado,
  `404` recurso inexistente, `409` conflicto (p. ej. cupo lleno, usuario ya confirmado).
- El backend MUST NOT filtrar detalles internos (mensajes de Firestore, rutas, trazas) en el
  cuerpo de la respuesta; esos detalles van a los logs.

### VI. Cero Código Sombra: la Spec es la Fuente de la Verdad

- Se construye estrictamente lo documentado en `spec.md` de cada feature. Prohibido agregar
  características "por si acaso" (pasarelas de pago, perfiles complejos, notificaciones,
  roles, etc.) que no estén en la spec.
- Si una instrucción contradice esta constitución, o se detecta una falla lógica, el agente de
  IA MUST detenerse, advertir el problema y solicitar actualizar `spec.md` (o enmendar esta
  constitución) antes de tocar código fuente.

**Razón:** en SDD el valor está en que spec, plan y código digan lo mismo.

## Stack Tecnológico e Infraestructura

| Capa | Decisión |
|------|----------|
| Frontend / UI | React + TypeScript, estilos con Tailwind CSS |
| Backend | Go (biblioteca estándar como base; dependencias solo si se justifican) |
| Base de datos | Google Cloud Firestore (modo nativo) |
| Hosting backend | Cloud Run |
| Hosting frontend | Firebase Hosting (mismo proyecto de GCP) |
| Despliegue | GitHub Actions: backend → Cloud Run, frontend → Firebase Hosting |

**Hosting del frontend: Firebase Hosting.** Elegido sobre Cloud Storage + Load Balancer y sobre
un contenedor estático en Cloud Run porque:
- Sirve una SPA estática con CDN global, HTTPS y dominio propio (`vamosjuntos.club`) sin
  configurar balanceadores, y su capa gratuita cubre de sobra 25 usuarios.
- Vive en el mismo proyecto de GCP que Firestore y Cloud Run.
- Permite reescribir `/api/**` hacia el servicio de Cloud Run, así frontend y backend comparten
  origen: sin CORS y con cookies de primera parte.

Restricción derivada: Firebase Hosting solo reenvía a Cloud Run la cookie llamada `__session`.
Si la sesión se implementa con cookie, MUST usar ese nombre.

**Configuración y secretos:** la lista de invitados y sus códigos viven en Firestore
(Principio II). La configuración del backend y la llave de sesión llegan por variables de
entorno de Cloud Run. Ningún secreto se versiona.

### Despliegue Continuo

- Cada repositorio de código tiene su propio pipeline en **GitHub Actions**
  (`.github/workflows/` del repo correspondiente):
  - **Backend** (`/vamosjuntos-backend`): GitHub → imagen de contenedor en Artifact Registry →
    **Cloud Run**.
  - **Frontend** (`/vamosjuntos-frontend`): GitHub → build estático → **Firebase Hosting**.
- Un push a la rama `production` MUST disparar el despliegue a producción; pushes a cualquier
  otra rama (incluida `main`, que es la rama de integración) MUST NOT desplegar. No hay otros
  ambientes en la V1.
- El pipeline MUST ejecutar las compuertas de calidad del Principio IV antes de desplegar; si
  alguna falla, el despliegue MUST NOT ocurrir.
- Los pipelines MUST autenticarse en GCP con Workload Identity Federation. Prohibido guardar
  llaves JSON de cuentas de servicio en los secrets de GitHub.
- Los workflows MUST NOT contener en texto plano los códigos de invitación ni otros secretos;
  se guardan en los secrets del repositorio en GitHub (o en Secret Manager) y llegan al backend
  como variables de entorno de Cloud Run.
- El detalle de cada workflow (pasos, acciones, Dockerfile) se define en el `plan.md` del spec
  de infraestructura, no en esta constitución.

**Tiempo:** todas las fechas del dominio están en hora de Colombia (`America/Bogota`, UTC-5,
sin horario de verano). Se almacenan como `Timestamp` de Firestore (UTC) y se muestran en
`America/Bogota`.

## Dominio de la V1

Estas reglas son inmutables para la V1; cambiarlas requiere enmendar esta constitución.

**Evento: "Paseo Familiar Diciembre 2026"**
- Inicio: 2026-12-23 14:00 (hora Colombia). Fin: 2026-12-26 11:00 (hora Colombia).
- Destino: Acacías, Meta, Colombia.
- Lugares de salida: Bogotá y Tocancipá, Colombia.
- Capacidad: 25 usuarios.

**Catálogo de invitados (22 + hasta 3 nuevos = 25):** Carmencita, Guillermito, Maidecita,
Alisson, Estiben, Liam, Mafe, Janneth, Orlando, Melissa, Samuel, Sergio, Heidy, Maria, Luis,
Sebastián, Alejandro, El Fercho, Lorena, papá de lore, mamá de lore, Nicolás López.
Cada invitado es un documento de `events/{EVENT_ID}/users` con los campos `Nombre` y `código`
(su código de confirmación propio). Este documento no contiene los códigos reales.

**Funcionalidades permitidas en la V1 (y solo estas):**
- **Onboarding:** según el Principio II.
- **Card "Próximo viaje":** nombre del evento, lugar(es) de salida, lugar de llegada, fecha de
  inicio, fecha de fin, cantidad de invitados, confirmados y pendientes por confirmar.
- **Banner de cuenta regresiva:** un banner MUY EXPRESIVO que le dice al usuario cuántos días
  faltan para el próximo evento al que está invitado (días calculados en `America/Bogota`).
- **Módulo Transporte:** vehículos y cupos.
- **Módulo Itinerario:** actividades y horarios durante el evento.
- **Módulo Compras:** comida y suministros.

El detalle de cada funcionalidad se define en su propio `spec.md`; esta sección solo fija el
alcance.

## Gobernanza

- Esta constitución prevalece sobre cualquier otra práctica, convención o instrucción puntual.
  `spec.md`, `plan.md` y `tasks.md` MUST ser coherentes con ella.
- Todo `plan.md` MUST pasar el "Constitution Check" contra los Principios I-VI. Cualquier
  violación MUST quedar justificada en la tabla de complejidad del plan o el plan se rechaza.
- **Enmiendas:** se proponen con `/speckit-constitution`, quedan registradas en el Sync Impact
  Report y las aprueba el responsable del proyecto antes de aplicarse a specs o código.
- **Versionado semántico:**
  - MAJOR: eliminar o redefinir un principio o una regla de dominio de forma incompatible.
  - MINOR: agregar un principio, sección o módulo, o ampliar materialmente una guía.
  - PATCH: aclaraciones, redacción y erratas sin cambio de significado.
- **Revisión de cumplimiento:** en cada revisión de cambios (y en `/speckit-analyze`) se verifica
  que no haya código sombra, que los errores sigan el Principio V, que el backend pase
  `gofmt`, `go vet` y `golangci-lint`, y que los pipelines cumplan "Despliegue Continuo".

**Version**: 2.0.0 | **Ratified**: 2026-10-30 | **Last Amended**: 2026-10-30
