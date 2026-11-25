# E-commerce SaaS Backend

Backend de un E-commerce SaaS **multi-tenant lógico por `tenant_id`**, construido en microservicios Django.
Es un **monorepo**: cada `ecom-*` es un servicio (o librería) independiente dentro de este mismo repo.

> Proyecto en **reconstrucción incremental**. El orden de construcción está en [ROADMAP.md](ROADMAP.md)
> y el contexto para Claude Code en [CLAUDE.md](CLAUDE.md).

## Servicios

| Carpeta | Responsabilidad | Fase del roadmap |
|---|---|---|
| [ecom-infra](ecom-infra/) | docker-compose, Traefik, Postgres init, `tasks.ps1` | 0 |
| [ecom-common](ecom-common/) | Librería compartida: tenancy, auth, outbox, inbox, messaging, http, observability, testing | 1 (crece en cada fase) |
| [ecom-identity-svc](ecom-identity-svc/) | Tenants, dominios, usuarios staff, membresías, compradores, JWT, forwardAuth | 2–3 |
| [ecom-catalog-svc](ecom-catalog-svc/) | Categorías, productos, variantes (**servicio de referencia**) | 4–5 |
| [ecom-contracts](ecom-contracts/) | JSON Schemas de eventos y OpenAPI | 5 (crece en cada fase) |
| [ecom-inventory-svc](ecom-inventory-svc/) | Stock, reservas, kardex | 6 |
| [ecom-order-svc](ecom-order-svc/) | Carrito, pedidos, orquestador de la saga de checkout | 7–8 |
| [ecom-payment-svc](ecom-payment-svc/) | Stripe (test mode), webhooks firmados, reembolsos | 9 |
| [ecom-promotion-svc](ecom-promotion-svc/) | Cupones y descuentos | 10 |
| [ecom-notification-svc](ecom-notification-svc/) | Correos por eventos (solo worker + consumer) | 10 |

`_referencia/ecom-scaffold.tar.gz` es el scaffold anterior, que se guarda solo para consulta.

## Stack

Python 3.12 · Django 5.2 LTS + DRF · PostgreSQL 16 (una base por servicio) · Redis 7 · Celery + beat ·
RabbitMQ 3.13 · Traefik v3 · Stripe (test mode) · Docker Desktop.

## Cómo viaja el tenant

```
navegador (demo.localhost)
  -> Traefik: borra X-Tenant-ID del cliente
  -> forwardAuth identity: Host -> tenant_id, valida cookie JWT, ¿coincide el tenant?
  -> servicio: JWTClaimsMiddleware -> TenantMiddleware -> contextvar
       Model.objects       filtra WHERE tenant_id = <contexto>   (sin contexto = TenantNotSet)
       publish_event()     guarda tenant_id en el envelope
       TenantTask          restaura el tenant en Celery
       tkey()              t:<tenant_id>:... en Redis
  -> consumer RabbitMQ: envelope.tenant_id -> tenant_context -> handler
```

## Reglas de oro

1. Todo modelo de negocio hereda de `TenantModel`.
2. Las restricciones únicas y los índices siempre llevan `tenant_id` primero.
3. Sin tenant en contexto se lanza un error (fail closed). `all_tenants` solo se usa en tareas de sistema.
4. Ningún servicio lee la base de otro: se comunican por REST interno (`clients/`) o por eventos.
5. Publicar un evento = escribir en el outbox en la misma transacción. Consumirlo = pasar por el inbox idempotente.
6. Cada endpoint nuevo agrega su caso en `tests/test_tenant_isolation.py`.
7. El dinero siempre va en centavos (`int`).

## Convenciones de git

- Rama principal: `main`.
- Commits con el servicio como scope: `feat(catalog): crea Category`, `chore(infra): agrega postgres`.
- Ver solo el historial de un servicio:

```powershell
git log --oneline -- ecom-catalog-svc
```
