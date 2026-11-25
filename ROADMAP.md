# ROADMAP — construir E-Com-Saas-backend archivo por archivo

Guía para reconstruir el proyecto desde cero, entendiendo cada pieza antes de pasar a la siguiente.
Las decisiones de arquitectura están en [CLAUDE.md](CLAUDE.md) y no cambian. Lo que cambia es el **orden**.

## Cómo usar este roadmap

- El árbol de carpetas ya existe y el monorepo ya está inicializado en la raíz. El `.gitattributes` es uno solo, en la raíz, y cada `ecom-*` ya tiene `README.md` y `.gitignore`. Esos archivos no se repiten en las fases.
- Cada fase termina en un **checkpoint** que puedes ejecutar: tests en verde, un `curl` que responde o un contenedor sano. Si no pasa, no avances.
- Dentro de cada fase, los archivos van **en el orden en que conviene escribirlos**. Marca `[x]` al terminar cada uno.
- **ecom-common crece justo a tiempo.** No se escribe completo al principio: cada módulo se agrega en la fase donde un servicio lo necesita por primera vez. Así cada pieza llega con un caso real que la justifica.
- El scaffold anterior está en `_referencia/ecom-scaffold.tar.gz`. Consúltalo **después** de intentar escribir el archivo tú, no antes. Para ver un archivo sin extraer todo:
  ```powershell
  tar -xzOf _referencia/ecom-scaffold.tar.gz ./ecom-common/src/ecom_common/tenancy/context.py
  ```
- El `.gitattributes` de la raíz fuerza LF, así que los `.sh` no se rompen con CRLF.
- Haz un commit por cada archivo o grupo pequeño. El historial de git se vuelve tu bitácora de aprendizaje.

## Mejoras respecto al scaffold anterior

| Problema en el scaffold | Cómo se corrige en este roadmap |
|---|---|
| Se generó todo de golpe, sin validar con Docker | Cada fase se levanta en Docker antes de continuar |
| `ecom-common` completo desde el día 1 (mucho código sin uso) | Se construye por módulos, en la fase que los necesita |
| `requires-python >=3.11` pero ruff con `target-version = py312` | Python **3.12** en todo: pyproject, ruff y Dockerfile |
| `axllent/mailpit:latest` sin versión fija | Todas las imágenes con tag fijo |
| `git init` quedó pendiente al final, y eran 10 repos separados | Un monorepo inicializado desde el día 1 |
| Los TODO quedaron sueltos en el código | Cada TODO tiene su fase y su checkpoint (fases 8, 10 y 11) |
| Traefik, forwardAuth y RabbitMQ entraron juntos (difícil de depurar) | Cada uno entra solo, en su propia fase |

## Vista general

| Fase | Qué construyes | Concepto principal | Checkpoint |
|---|---|---|---|
| 0 | ecom-infra mínimo | Docker Compose, una base por servicio | Postgres y Redis `healthy` |
| 1 | ecom-common: tenancy | ContextVar, manager *fail closed* | pytest de common en verde |
| 2 | identity: tenants y usuarios | Modelos globales vs. de tenant | `create_tenant demo` funciona |
| 3 | identity: JWT + gateway | RS256, JWKS, cookies httpOnly, forwardAuth | Login en `demo.localhost` deja cookie |
| 4 | catalog (servicio de referencia) | Capas views/services/selectors, caché por tenant | CRUD admin + storefront + test de aislamiento |
| 5 | Eventos: outbox + RabbitMQ | Outbox transaccional, envelope versionado | `variant.created` llega a RabbitMQ |
| 6 | inventory | Inbox idempotente, locks ordenados, kardex | Crear variante => aparece su StockLevel |
| 7 | order: carrito | Redis por tenant, client credentials | Carrito con precios traídos de catalog |
| 8 | order: saga sin pago | Saga orquestada, compensaciones, state machine | Checkout reserva y libera stock |
| 9 | payment | Stripe test mode, webhooks firmados, idempotencia | Pago de prueba => pedido `PAID` |
| 10 | promotion + notification | UPDATE condicional atómico, correos por eventos | Cupón aplicado + correo en Mailpit |
| 11 | Endurecimiento | DLQ, concurrencia, contratos, OTel, RLS | Suite de concurrencia en verde |

---

## Fase 0 — Plataforma mínima (`ecom-infra`)

**Objetivo:** levantar solo Postgres y Redis. Ni Traefik ni RabbitMQ todavía.

- [x] `ecom-infra/.gitignore`: `.env`, `secrets/*` (salvo `.gitkeep`).
- [ ] `ecom-infra/.env.example`: credenciales de Postgres y la contraseña de cada base (`IDENTITY_DB_PASSWORD`, …).
- [ ] `ecom-infra/postgres/init/01-create-databases.sh`: crea una base y un usuario por servicio. Debe quedar en **LF**.
- [ ] `ecom-infra/docker-compose.yml`: servicios `postgres:16-alpine` y `redis:7-alpine`, con healthchecks y volúmenes nombrados.
- [ ] `ecom-infra/tasks.ps1`: comandos `up`, `down`, `logs`, `ps` y `psql <servicio>`. Se irá ampliando.
- [x] `ecom-infra/README.md`: cómo arrancar.

**Checkpoint:** `./tasks.ps1 up`, luego `docker compose ps` muestra ambos `healthy`, y `./tasks.ps1 psql identity` conecta con el usuario `identity`.

---

## Fase 1 — `ecom-common`: núcleo de tenancy

**Objetivo:** la regla central del proyecto, probada antes de que exista ningún servicio.

- [ ] `ecom-common/pyproject.toml`: dependencias mínimas (Django 5.2, DRF), extras `dev` (pytest, pytest-django, ruff) y Python 3.12.
- [x] `ecom-common/.gitignore`.
- [ ] `src/ecom_common/__init__.py`
- [ ] `src/ecom_common/models.py`: `BaseModel` con `id` UUID, `created_at` y `updated_at`.
- [ ] `src/ecom_common/money.py`: helpers para centavos `int` (formatear, validar que no sea negativo).
- [ ] `src/ecom_common/tenancy/exceptions.py`: `TenantNotSet`, `TenantMismatch`.
- [ ] `src/ecom_common/tenancy/context.py`: `ContextVar`, `get_current_tenant()`, `tenant_context()`.
- [ ] `src/ecom_common/tenancy/models.py`: `TenantManager` (filtra siempre), `TenantModel`, `all_tenants`.
- [ ] `src/ecom_common/tenancy/middleware.py`: `TenantMiddleware`. Por ahora lee el header `X-Tenant-ID`; en la fase 3 pasará a leer los claims del JWT.
- [ ] `src/ecom_common/tenancy/permissions.py`: permiso DRF que exige tenant en contexto.
- [ ] `tests/settings.py`: SQLite en memoria y una app de prueba con un modelo `TenantModel`.
- [ ] `tests/test_tenancy_context.py`: sin contexto lanza `TenantNotSet`; con el tenant A no ve las filas de B; `tenant_context` restaura el valor anterior; `save()` con otro tenant lanza `TenantMismatch`.
- [ ] `src/ecom_common/testing/fixtures.py`: fixtures `tenant_a` y `tenant_b` (el resto llega después).
- [x] `README.md`

**Checkpoint:** `pip install -e .[dev]`, luego `pytest` en verde.
**Pregunta para comprobar que lo entendiste:** ¿por qué `ContextVar` y no `threading.local`?

---

## Fase 2 — `ecom-identity-svc`: esqueleto, tenants y usuarios

**Objetivo:** el primer servicio Django y la plantilla de servicio que copiarás 7 veces.

### 2a. Esqueleto de servicio (plantilla reutilizable)
- [ ] `.env.example` (el `.gitignore` ya existe)
- [ ] `pyproject.toml`: configuración de pytest (`DJANGO_SETTINGS_MODULE=config.settings.test`) y de ruff.
- [ ] `requirements/base.txt`, `requirements/dev.txt`
- [ ] `manage.py`
- [ ] `config/__init__.py`, `config/settings/__init__.py`
- [ ] `config/settings/base.py`: django-environ, DRF, la base del servicio y `ecom_common` en `INSTALLED_APPS`.
- [ ] `config/settings/dev.py`, `config/settings/test.py` (SQLite), `config/settings/prod.py`
- [ ] `config/urls.py`, `config/wsgi.py`, `config/asgi.py`
- [ ] `src/ecom_common/observability/health.py` + `observability/urls.py` (en common): `/healthz` y `/readyz`.
- [ ] `Dockerfile`: python:3.12-slim, usuario sin root. Se construye con la raíz del monorepo como contexto para copiar `ecom-common`.
- [ ] `docker-entrypoint.sh`: `migrate` y después `gunicorn` (en LF).
- [ ] `tests/__init__.py`, `tests/conftest.py`
- [ ] Agregar `identity-api` a `ecom-infra/docker-compose.yml` (puerto publicado temporalmente, sin Traefik).

**Checkpoint:** `curl localhost:8001/healthz` responde 200 desde Docker.

### 2b. Dominio de identity
- [ ] `apps/tenants/models.py`: `Tenant` y `TenantDomain` (modelos **globales**, no heredan de `TenantModel`).
- [ ] `apps/tenants/migrations/0001_initial.py`: generada con `makemigrations`, no escrita a mano.
- [ ] `apps/tenants/resolver.py`: Host → tenant, con caché.
- [ ] `apps/tenants/services.py`: `create_tenant()` crea el tenant, su dominio y el owner en una sola transacción.
- [ ] `apps/tenants/management/commands/create_tenant.py`
- [ ] `apps/accounts/models.py`: `User` global (staff), `AUTH_USER_MODEL`.
- [ ] `apps/memberships/models.py` + `selectors.py`: User ↔ Tenant con rol.
- [ ] `apps/customers/models.py` + `services.py`: `Customer` (comprador), **sí** hereda de `TenantModel`.
- [ ] `tests/test_tenant_isolation.py`: primer caso (los customers del tenant A no se ven desde B).
- [ ] `ecom-infra/tasks.ps1`: agregar `bootstrap` (migraciones + tienda demo).

**Checkpoint:** `./tasks.ps1 bootstrap` crea `demo` con `owner@demo.com`.
**Pregunta:** ¿por qué `User` es global y `Customer` es por tenant?

---

## Fase 3 — Identity: JWT, JWKS y gateway

**Objetivo:** autenticación completa. Traefik entra aquí, solo.

- [ ] `apps/tokens/keys.py`: carga la llave privada RS256 y expone la pública como JWK.
- [ ] `apps/tokens/management/commands/generate_jwt_keys.py`
- [ ] `apps/tokens/services.py`: emitir access y refresh con claims `sub`, `tid`, `roles`, `typ`.
- [ ] `apps/tokens/cookies.py`: cookies `httpOnly`, `Secure` (en prod) y `SameSite=Lax`.
- [ ] `apps/tokens/api/serializers.py`, `views.py`, `urls.py`: login de staff y de cliente, refresh, logout y `/.well-known/jwks.json`.
- [ ] `apps/tenants/api/*`: signup de una tienda nueva.
- [ ] **ecom-common** `auth/jwks.py`: descarga y cachea el JWKS.
- [ ] **ecom-common** `auth/user.py`: `TokenUser` (usuario sin base de datos, construido desde los claims).
- [ ] **ecom-common** `auth/authentication.py`: autenticación DRF desde la cookie o el header.
- [ ] **ecom-common** `auth/middleware.py`: `JWTClaimsMiddleware`. `TenantMiddleware` pasa a leer `tid` del token.
- [ ] **ecom-common** `auth/permissions.py`: `IsStaff`, `IsCustomer`, `HasRole("owner")`.
- [ ] **ecom-common** `testing/jwt.py` + fixtures `make_jwt`, `api_client_for` y `public_client`.
- [ ] `apps/gateway/views.py`: `/internal/gateway/authorize` (Host → tenant, valida la cookie y comprueba que el tenant coincide; devuelve el header `X-Tenant-ID`).
- [ ] `ecom-infra/traefik/traefik.yml`
- [ ] `ecom-infra/traefik/dynamic/middlewares.yml`: borra el `X-Tenant-ID` que mande el cliente, forwardAuth y rate limit por tenant.
- [ ] `ecom-infra/traefik/dynamic/routes.yml`: `/api/v1/identity/*`. **Nunca** expone `/internal/*`.
- [ ] Quitar el puerto publicado de `identity-api`; ahora todo pasa por Traefik.

**Checkpoint:** `POST http://demo.localhost/api/v1/identity/auth/login` devuelve la cookie, y una petición con `X-Tenant-ID` falso es ignorada.
**Pregunta:** ¿qué ataque evita borrar `X-Tenant-ID` en Traefik?

---

## Fase 4 — `ecom-catalog-svc` (servicio de referencia)

**Objetivo:** el patrón de capas que todos los demás copian. Todavía sin eventos.

- [ ] Esqueleto (copiar 2a desde identity y ajustar nombres).
- [ ] `apps/categories/models.py`: `Category(TenantModel)` con `UniqueConstraint(tenant_id, slug)`.
- [ ] `apps/categories/selectors.py`, `services.py`
- [ ] `apps/categories/api/admin/{serializers,views,urls}.py`: `queryset=Category.objects`, nunca `.all()` a nivel de clase.
- [ ] `apps/categories/api/storefront/{serializers,views,urls}.py`
- [ ] `apps/products/models.py`: `Product`, `ProductVariant` (`price_cents: int`) y `ProductImage`.
- [ ] `apps/products/selectors.py`, `services.py`
- [ ] **ecom-common** `tenancy/cache.py`: `tkey()` genera `t:<tenant_id>:...`.
- [ ] `apps/products/cache.py`: caché del storefront por tenant, con invalidación al escribir.
- [ ] `apps/products/api/admin/*`, `apps/products/api/storefront/*`
- [ ] `apps/products/api/internal/{views,urls}.py`: precios por lote para order-svc (lo usará en la fase 7).
- [ ] `apps/products/tests/factories.py`, `test_services.py`, `test_storefront_api.py`
- [ ] `tests/test_tenant_isolation.py`: un caso por endpoint; un objeto de otro tenant da **404**.
- [ ] **ecom-common** `observability/correlation.py` + `logging.py`: `X-Request-ID` y logs JSON con `tenant_id`.
- [ ] Ruta de catalog en `routes.py` de Traefik.

**Checkpoint:** crear un producto como owner y verlo en `http://demo.localhost/api/v1/catalog/store/products/`.

---

## Fase 5 — Eventos: outbox, contratos y RabbitMQ

**Objetivo:** publicar eventos de forma confiable. Todavía nadie los consume.

- [ ] `ecom-contracts/CHANGELOG.md` (el README ya existe)
- [ ] `ecom-contracts/events/envelope.v1.json`: `id`, `type`, `version`, `tenant_id`, `occurred_at`, `correlation_id`, `data`.
- [ ] `ecom-contracts/events/catalog/variant.created.v1.json`, `variant.price_changed.v1.json`
- [ ] **ecom-common** `events/envelope.py`: `build_envelope()`.
- [ ] **ecom-common** `outbox/models.py` + migración, `outbox/publisher.py` (`publish_event()`).
- [ ] **ecom-common** `messaging/connection.py`, `messaging/topology.py`: exchange `topic`, colas por servicio.
- [ ] **ecom-common** `outbox/relay.py` + `outbox/tasks.py`: tarea de Celery que publica los pendientes.
- [ ] **ecom-common** `tenancy/celery.py`: `TenantTask` (recibe `tenant_id=` y restaura el contexto).
- [ ] **ecom-common** `tests/test_outbox.py`: si la transacción hace rollback, no queda ningún evento.
- [ ] `ecom-catalog-svc/config/celery.py` + `config/__init__.py` (carga la app de Celery).
- [ ] `ecom-catalog-svc/apps/products/services.py`: `publish_event("catalog.variant.created", …)` dentro del `atomic`.
- [ ] `ecom-catalog-svc/apps/products/tasks.py`
- [ ] Compose: `rabbitmq:3.13-management-alpine` y `catalog-worker` (Celery worker + beat).

**Checkpoint:** crear una variante y ver el mensaje en la UI de RabbitMQ (`localhost:15672`).
**Pregunta:** ¿qué falla si publicas directo a RabbitMQ en lugar de usar el outbox?

---

## Fase 6 — `ecom-inventory-svc`

**Objetivo:** consumir eventos de forma idempotente y controlar stock bajo concurrencia.

- [ ] Esqueleto (copiar desde catalog).
- [ ] **ecom-common** `inbox/models.py` + migración, `inbox/idempotency.py` (`handle_idempotent`).
- [ ] **ecom-common** `inbox/consumer.py`: loop de kombu; `envelope.tenant_id` → `tenant_context` → handler.
- [ ] `messaging/apps.py`, `messaging/routes.py` (routing_key → handler)
- [ ] `messaging/management/commands/consume_events.py`
- [ ] `apps/stock/models.py`: `StockLevel(on_hand, reserved)` con `CheckConstraint` de no negativos.
- [ ] `apps/stock/services.py`, `handlers.py` (`variant.created` crea un StockLevel en 0)
- [ ] `apps/stock/api/admin/*`: ajustes manuales.
- [ ] `apps/movements/models.py`: `StockMovement` (kardex, solo inserción) + `api/admin/*` (consulta).
- [ ] `apps/reservations/models.py`: `Reservation` + `ReservationLine`.
- [ ] `apps/reservations/services.py`: reservar, confirmar y liberar con `select_for_update()` **ordenado por `variant_id`**.
- [ ] `apps/reservations/handlers.py` (se conecta en la fase 8)
- [ ] `apps/reservations/tests/test_reservations.py`, `tests/test_tenant_isolation.py`
- [ ] Contratos `inventory/stock.reserved|rejected|low.v1.json`
- [ ] Compose: `inventory-api`, `inventory-worker` y `inventory-consumer`.

**Checkpoint:** crear una variante en catalog crea su StockLevel en inventory, y reenviar el mismo mensaje no lo duplica.

---

## Fase 7 — `ecom-order-svc`: carrito

**Objetivo:** comunicación síncrona entre servicios, autenticada.

- [ ] Esqueleto.
- [ ] **identity** endpoint `client_credentials` (`/internal/oauth/token`) + clientes de servicio.
- [ ] **ecom-common** `auth/service_token.py`: obtiene y cachea el token de servicio.
- [ ] **ecom-common** `http/client.py`: httpx con timeouts, reintentos, `X-Request-ID` y el token de servicio.
- [ ] **ecom-common** fixture `service_client_for`.
- [ ] `clients/catalog.py`: precios por lote desde `/internal/` de catalog.
- [ ] `apps/cart/repository.py`: hash de Redis en `tkey("cart", customer_id)` con TTL.
- [ ] `apps/cart/services.py`: agregar, quitar y calcular el total con los precios **actuales** de catalog.
- [ ] `apps/cart/api/{views,urls}.py`
- [ ] `tests/test_tenant_isolation.py`

**Checkpoint:** un cliente con sesión arma un carrito y ve los totales en centavos.

---

## Fase 8 — `ecom-order-svc`: pedidos y saga (sin pago)

**Objetivo:** transacción distribuida con compensaciones.

- [ ] `apps/orders/models.py`: `Order` y `OrderItem` con snapshot de nombre, SKU y precio.
- [ ] `apps/orders/state_machine.py`: transiciones permitidas; cualquier otra lanza error.
- [ ] `apps/orders/selectors.py`, `api/serializers.py`, `api/storefront/*`, `api/admin/*`
- [ ] Contratos `order/order.created|cancelled|paid|shipped.v1.json`, `payment.requested.v1.json`
- [ ] `apps/checkout/models.py`: `CheckoutSaga` (paso actual, `deadline`).
- [ ] `apps/checkout/saga.py`: orquestador (reservar stock → esperar → confirmar, o compensar).
- [ ] `apps/checkout/services.py`, `handlers.py` (`stock.reserved`, `stock.rejected`)
- [ ] `apps/checkout/tasks.py`: `expire_stale_sagas` (uso legítimo de `all_tenants`).
- [ ] `apps/checkout/api/*`
- [ ] Conectar `inventory/apps/reservations/handlers.py` a `order.created` y `order.cancelled`.
- [ ] `apps/checkout/tests/test_saga.py`: camino feliz, sin stock y timeout.

**Checkpoint:** un checkout reserva el stock; si la saga expira, el stock se libera y el pedido queda `CANCELLED`.

---

## Fase 9 — `ecom-payment-svc`

**Objetivo:** pasarela externa, webhooks firmados e idempotencia de extremo a extremo.

- [ ] Esqueleto.
- [ ] `apps/providers/base.py`: interfaz `PaymentProvider`.
- [ ] `apps/providers/stripe_provider.py`: PaymentIntent con `idempotency_key`.
- [ ] `apps/payments/models.py`: `Payment`, `Refund`.
- [ ] `apps/payments/services.py`, `handlers.py` (`payment.requested`)
- [ ] `apps/payments/api/views.py`: `OrderIntentView`. **Validar que el pedido pertenece al cliente** (TODO pendiente del scaffold).
- [ ] `apps/webhooks/models.py`: `WebhookEvent` (único por `provider_event_id`).
- [ ] `apps/webhooks/views.py`: verifica la firma, guarda y responde 200 rápido; procesa en `tasks.py`.
- [ ] Contratos `payment/payment.succeeded|failed.v1.json`, `payment.refund_requested.v1.json`
- [ ] Saga: `payment.succeeded` → confirmar reserva → `PAID`. **Pago tardío** sobre un pedido `CANCELLED` → `payment.refund_requested` (TODO pendiente).
- [ ] `apps/payments/tests/test_payments.py`: firma inválida, webhook duplicado y pago tardío.
- [ ] Ruta pública del webhook en Traefik (sin forwardAuth).

**Checkpoint:** `stripe listen --forward-to demo.localhost/...` y la tarjeta de prueba `4242 4242 4242 4242` dejan el pedido en `PAID`.

---

## Fase 10 — `ecom-promotion-svc` + `ecom-notification-svc`

### Promotion
- [ ] Esqueleto, `apps/coupons/models.py`: `Coupon`, `CouponRedemption`.
- [ ] `apps/coupons/services.py`: canje con `UPDATE ... WHERE used < max_uses` (atómico y sin lock explícito).
- [ ] `apps/coupons/handlers.py`, `api/admin/*`, `api/storefront/*` (validar un código)
- [ ] Contratos `promotion/coupon.reserved|rejected.v1.json` y un paso de cupón en la saga.
- [ ] `apps/coupons/tests/test_coupons.py`

### Notification
- [ ] Esqueleto (solo worker + consumer, sin API).
- [ ] `apps/dispatch/models.py`: `EmailLog`.
- [ ] `apps/dispatch/services.py`, `tasks.py`, `handlers.py`
- [ ] `apps/dispatch/templates/emails/order_paid.txt`, `order_shipped.txt`
- [ ] **identity** endpoint interno: email del owner por tenant → alerta `stock.low` (TODO pendiente).
- [ ] Compose: `axllent/mailpit` con versión fija.
- [ ] `apps/dispatch/tests/test_dispatch.py`

**Checkpoint:** compra con cupón, el correo llega a Mailpit (`localhost:8025`) y el cupón agotado se rechaza.

---

## Fase 11 — Endurecimiento

- [ ] **ecom-common** inbox: cola de retraso con TTL + DLQ, en lugar de `sleep` (TODO pendiente).
- [ ] Tests de concurrencia en Postgres real: 50 reservas simultáneas de la última unidad terminan en exactamente 1 éxito.
- [ ] Tests de contrato: cada `publish_event` se valida contra su JSON Schema en `ecom-contracts`.
- [ ] `ecom-contracts/openapi/`: exportar el esquema de cada servicio con drf-spectacular.
- [ ] OpenTelemetry: trazas a través de HTTP, Celery y RabbitMQ.
- [ ] Row Level Security de Postgres como segunda barrera de tenancy.
- [ ] `ecom-infra/Makefile` (para Linux/Mac).
