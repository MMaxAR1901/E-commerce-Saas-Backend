# ecom-contracts

Contratos entre servicios: JSON Schemas de eventos y especificaciones OpenAPI.

**Estado:** carpetas creadas, sin código. Se construye en la **fase 5 (crece en cada fase)** de
[ROADMAP.md](../ROADMAP.md).

## Responsabilidad

Es la fuente de verdad de lo que viaja entre servicios.

- `events/envelope.v1.json`: sobre común (`id`, `type`, `version`, `tenant_id`, `occurred_at`, `correlation_id`, `data`).
- `events/<dominio>/<evento>.v<N>.json`: esquema del `data` de cada evento.
- `openapi/`: esquemas exportados de cada servicio (fase 11).

## Reglas de versionado

- Agregar un campo **opcional** es compatible: se edita el `.v1`.
- Quitar o renombrar un campo, o cambiar su tipo, **no es compatible**: se crea `.v2` y conviven las dos versiones hasta migrar los consumidores.
- Cada cambio se anota en `CHANGELOG.md`.
