# ecom-inventory-svc

Inventario: stock por variante, reservas para el checkout y kardex.

**Estado:** carpetas creadas, sin código. Se construye en la **fase 6** de
[ROADMAP.md](../ROADMAP.md).

## Responsabilidad

- `StockLevel` (`on_hand`, `reserved`, nunca negativos).
- `Reservation` + `ReservationLine`: se reservan, confirman o liberan desde la saga, con `select_for_update()` ordenado por `variant_id`.
- `StockMovement`: kardex de solo inserción.
- Consume `catalog.variant.created` y los eventos del pedido; publica `stock.reserved`, `stock.rejected` y `stock.low`.

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
