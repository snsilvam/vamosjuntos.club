# Data Model: Autenticación con Firebase

**Feature**: `008-autenticacion-firebase` | **Date**: 2026-10-30

> Base de Firestore: `vamosjuntos-club-db`. Sus reglas de seguridad siguen negando todo acceso
> desde clientes; solo el backend lee y escribe.

## Cuenta: Firebase Authentication (no se guarda en Firestore)

| Atributo | Origen | Uso |
|----------|--------|-----|
| `uid` | ID token verificado (`sub`) | Identifica la cuenta en `links` y `loginAttempts` |
| `email` | ID token (`email`) | Se copia en el vínculo para que el organizador sepa quién es |
| Método | Google o correo/contraseña | No lo usa el backend |

La cuenta la crea y la guarda Firebase Authentication. No se exige correo verificado (spec,
Assumptions).

## Vínculo: Firestore `events/{EVENT_ID}/links/{guestId}`

El id del documento es el `guestId`, es decir, el id del documento en `users`.

| Campo | Tipo | Regla |
|-------|------|-------|
| `uid` | string | La cuenta vinculada. Una cuenta no puede aparecer en dos documentos (FR-005) |
| `email` | string | Correo de la cuenta al vincular; solo informativo |
| `linkedAt` | timestamp | Hora del servidor |

**Creación**: transacción con este orden:
1. consultar `links where uid == X`; si hay alguno → `account_already_linked`;
2. `Create(links/{guestId})`; si ya existe → `guest_already_linked`.

**Borrado**: solo el organizador, en la consola (deshacer un vínculo). La app nunca edita ni borra
vínculos (FR-006).

## Estado de la cuenta (calculado, no se guarda)

```text
sin token válido ─────────────────────────────────────► 401 unauthenticated
token válido, sin vínculo ────────────────────────────► unlinked   → lista / "entrar con mi código"
vínculo a un guestId que ya no está en el catálogo ───► 403 not_invited
vínculo + attendance pendiente ───────────────────────► pending    → pregunta de asistencia
vínculo + attendance "declined" ──────────────────────► declined   → pregunta de asistencia
vínculo + attendance "confirmed" ─────────────────────► confirmed  → pantalla principal
```

Las rutas protegidas exigen `confirmed`. Si no, responden `403`: `not_linked` cuando no hay vínculo
y `not_invited` en los demás casos.

## Sin cambios (spec 001)

- **Invitado**: `events/{EVENT_ID}/users/{guestId}`, con `Nombre` y `código`.
- **Respuesta de asistencia**: `events/{EVENT_ID}/attendance/{guestId}`, con `status`,
  `respondedAt` y `respondedBy`. `respondedBy` sigue siendo un `guestId`: el propio al responder
  y el de quien confirma a otro.
- **Transiciones de asistencia**: `nextStatus` de la 001.

## Cambia: intentos de código, Firestore `loginAttempts/{uid}`

Mismos campos (`failures`, `blockedUntil`) y misma regla (5 fallos → 15 minutos). Solo cambia el
id del documento: antes era el `X-Device-Id` y ahora es el `uid` (FR-014). Cuentan como intento
los códigos de `link` y los de confirmar a otro.

## Sale

- La cookie `__session` con el token HMAC y `SESSION_KEY`.
- `X-Device-Id`.
