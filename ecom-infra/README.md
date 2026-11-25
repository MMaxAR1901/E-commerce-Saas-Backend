# ecom-infra

Infraestructura local del E-commerce SaaS: Docker Compose, gateway y bases de datos.

**Estado:** carpetas creadas, sin código. Se construye en la **fase 0 (y se amplía en cada fase)** de
[ROADMAP.md](../ROADMAP.md).

## Responsabilidad

- `docker-compose.yml`: Postgres 16, Redis 7, RabbitMQ 3.13, Traefik v3, Mailpit y los contenedores de cada servicio.
- `postgres/init/`: crea una base y un usuario por servicio.
- `traefik/`: rutas públicas, forwardAuth contra identity, borrado de `X-Tenant-ID` y rate limit por tenant.
- `tasks.ps1`: comandos del día a día en PowerShell (`up`, `down`, `logs`, `test`, `bootstrap`).
- `scripts/`: utilidades del workspace (clonar todos los repos, etc.).

## Estructura

```
postgres/init/       scripts de inicialización (siempre en LF)
traefik/             traefik.yml + dynamic/ (routes.yml, middlewares.yml)
scripts/             utilidades bash (Git Bash)
secrets/             llaves locales, NO se versionan (solo .gitkeep)
```

## Uso

```powershell
Copy-Item .env.example .env
./tasks.ps1 up
./tasks.ps1 ps
```

Todas las imágenes van con tag fijo; nunca `:latest`.
