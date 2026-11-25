# ecom-common

Librería compartida (paquete `ecom_common`) que instalan todos los servicios.

**Estado:** carpetas creadas, sin código. Se construye en la **fase 1 (crece en cada fase)** de
[ROADMAP.md](../ROADMAP.md).

## Responsabilidad

Infraestructura transversal; **nada de lógica de negocio**.

| Módulo | Qué hace | Fase |
|---|---|---|
| `tenancy` | ContextVar del tenant, `TenantModel`, middleware, `TenantTask`, `tkey()` | 1, 4, 5 |
| `observability` | health checks, correlation id, logs JSON | 2, 4 |
| `auth` | JWKS, autenticación DRF, `TokenUser`, permisos, token de servicio | 3, 7 |
| `testing` | fixtures de pytest (`tenant_a`, `api_client_for`, `make_jwt`, …) | 1, 3, 7 |
| `events` / `outbox` / `messaging` | envelope, publicación confiable, conexión y topología de RabbitMQ | 5 |
| `inbox` | consumo idempotente de eventos | 6 |
| `http` | cliente httpx entre servicios | 7 |

## Desarrollo

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -e .[dev]
pytest
```

Los servicios la instalan en modo editable: `pip install -e ../ecom-common`.
