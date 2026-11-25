# CLAUDE.md — E-Com-Saas-backend

Contexto para Claude Code. Léelo completo antes de proponer cambios.

## Qué es

Backend de un **E-commerce SaaS con panel administrativo** (proyecto de aprendizaje intermedio-avanzado).
Temas que se están aprendiendo: autenticación, procesos de compra, webhooks, seguridad y transacciones distribuidas.

Funcionalidades: catálogo, carrito, gestión de pedidos, inventario automático, cupones y descuentos,
y pagos con pasarela en sandbox (Stripe en test mode).

**Modo de trabajo actual: reconstrucción incremental.** El proyecto se construye archivo por archivo siguiendo
[ROADMAP.md](ROADMAP.md). Lo que dice el Mapa de abajo es el **estado objetivo**, no lo que existe hoy.

## Decisiones ya tomadas (no reabrir sin preguntar)

- **Microservicios en un monorepo.** Esta carpeta es un solo repo git ([MMaxAR1901/E-commerce-Saas-Backend](https://github.com/MMaxAR1901/E-commerce-Saas-Backend)); cada `ecom-*` es un servicio o librería independiente dentro de él. Aunque compartan repo, la regla 6 sigue vigente: ningún servicio importa código de otro, salvo `ecom-common`. (Antes era polyrepo; se cambió el 2026-11-24.)
- **Multi-tenant lógico por `tenant_id`**: esquema compartido, con `tenant_id` en cada tabla de negocio. No se usa schema-per-tenant.
- **Stack backend:** Python 3.12, Django 5.2 LTS + DRF, PostgreSQL (una base y un usuario por servicio), Redis (caché, carrito y broker de Celery), Celery + beat, RabbitMQ para los eventos de dominio entre servicios y Traefik como gateway.
- **Frontends (repos aparte, aún no creados):** storefront en Next.js + Tailwind (SEO, ISR) y panel admin en Vite + React + Tailwind (SPA).
- **Auth:** identity emite JWT RS256 en cookies httpOnly y los demás servicios validan localmente con JWKS. Los tokens entre servicios usan client credentials.
- **Checkout:** saga orquestada por order-svc, con outbox (publicación confiable) e inbox (consumo idempotente).
- **Dinero:** siempre en centavos (`int`), nunca `float`.
- **Imágenes Docker con tag fijo** (nada de `:latest`).

## Mapa (estado objetivo)

```
ROADMAP.md             orden de construcción, fase por fase y archivo por archivo
_referencia/           ecom-scaffold.tar.gz: scaffold anterior, solo para consulta (borrar al terminar)
ecom-infra/            docker-compose, Traefik (forwardAuth + rate limit por tenant), Postgres init, tasks.ps1
ecom-contracts/        JSON Schemas de eventos (envelope + data) y OpenAPI
ecom-common/           librería compartida: tenancy, auth, outbox, inbox, messaging, http, observability, testing
ecom-identity-svc/     Tenant, TenantDomain, User global, Membership, Customer (por tenant), JWT, /internal/gateway/authorize
ecom-catalog-svc/      Category, Product, ProductVariant, ProductImage  ← SERVICIO DE REFERENCIA (copiar sus patrones)
ecom-inventory-svc/    StockLevel, Reservation(+Line), StockMovement (kardex)
ecom-order-svc/        carrito Redis, Order/OrderItem (snapshot), state_machine, CheckoutSaga (orquestador)
ecom-promotion-svc/    Coupon, CouponRedemption (UPDATE condicional atómico)
ecom-payment-svc/      Payment, Refund, providers/ (Stripe), WebhookEvent + webhook firmado
ecom-notification-svc/ EmailLog, correos por eventos (solo worker + consumer, sin API)
```

## Cómo viaja el tenant (regla central)

```
cliente -> Traefik: strip X-Tenant-ID -> forwardAuth identity (Host -> tenant, valida cookie JWT, tenant coincide)
        -> servicio: JWTClaimsMiddleware -> TenantMiddleware -> contextvar
             Model.objects filtra por tenant (sin contexto => TenantNotSet, fail closed)
             publish_event() mete tenant_id en el envelope
             TenantTask (Celery) recibe tenant_id=... y restaura el contexto
             tkey() => claves Redis t:<tenant_id>:...
consumer RabbitMQ: envelope.tenant_id -> tenant_context -> handler (dentro de handle_idempotent)
```

## Reglas obligatorias

1. Todo modelo de negocio hereda de `ecom_common.tenancy.models.TenantModel`. La excepción es identity, donde `Tenant`, `TenantDomain`, `User` y `Membership` son modelos globales.
2. `UniqueConstraint` e índices siempre llevan `tenant_id` como primer campo.
3. `Model.all_tenants` solo se usa en tareas de sistema explícitas (por ejemplo `expire_stale_sagas`).
4. En serializers se pasa el **manager** (`queryset=Category.objects`), nunca `.all()` a nivel de clase.
5. `bulk_create` no llama a `save()`, así que hay que asignar `tenant_id` a mano.
6. Ningún servicio lee la base de otro. La comunicación va por `clients/` (REST interno, `/internal/*`, que no se expone en Traefik) o por eventos.
7. Para publicar un evento se usa `publish_event()` DENTRO de la transacción del cambio de negocio. Si un evento cambia de forma incompatible, se crea un `.v2` en ecom-contracts.
8. Capas: las `views` son delgadas, `services` concentra las escrituras (con `transaction.atomic` y eventos), `selectors` las lecturas, y `handlers` reutiliza los services.
9. Cada endpoint nuevo agrega su caso a `tests/test_tenant_isolation.py`. Un objeto de otro tenant responde 404 (no 403).
10. Locks: `select_for_update()` siempre ordenado (por ejemplo por `variant_id`) para evitar deadlocks.
11. El `.gitattributes` de la raíz fuerza LF (`* text=auto eol=lf`; los `.sh` siempre en LF, los `.ps1` en CRLF). Cada `ecom-*` puede tener su propio `.gitignore`.
12. Las migraciones se generan con `makemigrations`, nunca se escriben a mano.
13. Los Dockerfile se construyen con la **raíz del repo como contexto** (`context: ..` en compose), para poder copiar `ecom-common`.
14. Commits con prefijo del servicio: `feat(catalog): ...`, `chore(infra): ...`.

## Comandos

Están disponibles a medida que avanzan las fases (por ejemplo, `bootstrap` aparece en la fase 2).

```powershell
cd ecom-infra
./tasks.ps1 up
./tasks.ps1 bootstrap      # (fase 2+) .env + llave JWT + migraciones + tienda demo (owner@demo.com / demo12345)
./tasks.ps1 test catalog
./tasks.ps1 logs order-consumer
```

Fuera de Docker, desde un servicio:

```powershell
pip install -e ../ecom-common -r requirements/dev.txt
pytest
```

Los tests usan SQLite en memoria (`config.settings.test`) y fixtures de `ecom_common.testing.fixtures`: `tenant_a`, `tenant_b`, `api_client_for`, `public_client`, `service_client_for` y `make_jwt` (cada una llega en la fase donde se necesita).

Para consultar un archivo del scaffold anterior sin extraerlo todo:

```powershell
tar -xzOf _referencia/ecom-scaffold.tar.gz ./ecom-common/src/ecom_common/tenancy/context.py
```

## Estado (2026-11-24)

- Se borró el scaffold generado de golpe. Queda respaldado en `_referencia/ecom-scaffold.tar.gz`.
- Ya está creado el árbol de carpetas completo de los 10 repos `ecom-*` (178 carpetas), sin código todavía.
- Monorepo inicializado en la raíz (rama `main`, autor MMaxAR1901) y subido a GitHub. Cada `ecom-*` tiene su `README.md` y su `.gitignore`.
- Hay un `README.md` del workspace en la raíz.
- **Fase actual: 0 (plataforma mínima, `ecom-infra`).** El siguiente archivo es `ecom-infra/.env.example`.
- Los TODO que tenía el scaffold ya tienen fase asignada en ROADMAP.md:
  - payment: validar que el pedido pertenece al cliente en `OrderIntentView` y reembolsar el pago tardío (fase 9).
  - notification: alerta `stock.low` al dueño, con un endpoint interno en identity (fase 10).
  - common/inbox: cola de retraso con TTL + DLQ en lugar de `sleep` (fase 11).

## Roadmap (resumen; el detalle está en ROADMAP.md)

0 Infra mínima → 1 common/tenancy → 2 Identity (tenants y usuarios) → 3 JWT + Traefik → 4 Catalog → 5 Outbox + RabbitMQ
→ 6 Inventory → 7 Carrito → 8 Saga sin pago → 9 Payment → 10 Promotion + notification → 11 Endurecimiento.

## Cómo trabajar con Max

- Muestra el **análisis/plan antes de aplicar código** y espera su visto bueno.
- Prefiere **archivos completos listos para copiar** en lugar de diffs parciales.
- **Avanza de a un archivo o grupo pequeño** del roadmap. Explica qué concepto enseña el archivo antes de escribirlo, y no te adelantes a fases futuras.
- Al terminar una fase, verifica su checkpoint y actualiza la sección **Estado** de este archivo y las casillas de ROADMAP.md.
- Señala por iniciativa propia los problemas relacionados que veas, aunque no los haya pedido.
- Responde en español. Su entorno es Windows (PowerShell y Docker Desktop).
