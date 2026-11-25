# ecom-payment-svc

Pagos con Stripe en test mode y webhooks firmados.

**Estado:** carpetas creadas, sin código. Se construye en la **fase 9** de
[ROADMAP.md](../ROADMAP.md).

## Responsabilidad

- `providers/`: interfaz `PaymentProvider` e implementación de Stripe (PaymentIntent con `idempotency_key`).
- `Payment`, `Refund`.
- `WebhookEvent`: webhook con firma verificada, deduplicado por `provider_event_id` y procesado en Celery.
- Pendientes que cubre la fase 9: validar que el pedido pertenece al cliente y reembolsar el pago tardío.

## Estructura

```
apps/            una app Django por agregado: models, services, selectors, api/, handlers, tests/
clients/         clientes HTTP hacia /internal/* de otros servicios
config/          settings (base, dev, test, prod), urls, celery, wsgi/asgi
messaging/       routes.py (routing_key -> handler) y el comando consume_events
requirements/    base.txt y dev.txt
tests/           conftest.py y test_tenant_isolation.py
```

## Desarrollo local

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -e ../ecom-common -r requirements/dev.txt
pytest
```

Con Docker, desde `ecom-infra`: `./tasks.ps1 up` y `./tasks.ps1 test <servicio>`.

## Reglas

- Los modelos de negocio heredan de `TenantModel`; los únicos e índices empiezan por `tenant_id`.
- Las views son delgadas; las escrituras van en `services` (con `transaction.atomic` + `publish_event`) y las lecturas en `selectors`.
- Cada endpoint tiene su caso en `tests/test_tenant_isolation.py`; un objeto de otro tenant responde 404.
