# Plan de implementación — Despliegue de LuminaUL con Docker Compose

> Repositorio: `luminaul-docker` · Responsable: Tech Lead / Integrador · Fecha: 2026-11-07
>
> Objetivo: que `docker compose up -d --build` levante PostgreSQL 16, el backend NestJS y el frontend React (Caddy) desde los repos oficiales, con una sola URL de entrada, sesión por cookie y chat WebSocket funcionando.

---

## 0. Diagnóstico del estado actual

| # | Problema | Impacto | Dónde se corrige |
|---|----------|---------|------------------|
| 1 | `luminaul-new-backend` y `luminaul-frontend` **no tienen `Dockerfile`** ni `.dockerignore`. El compose hace `build: https://github.com/...git#main`, que necesita un Dockerfile en la raíz del repo remoto. | **Bloqueante**: `docker compose build` falla. | Repos de backend y frontend (zona roja → Tech Lead) |
| 2 | Con `NODE_ENV=production`, `src/config/database.config.ts` activa `ssl: { rejectUnauthorized }`. La imagen `postgres:16-alpine` no tiene SSL activado. | **Bloqueante**: el backend no conecta a la BD. | Compose (`?sslmode=disable` en `DATABASE_URL`) o una variable `DATABASE_SSL` en el backend |
| 3 | El gateway de chat valida el `Origin` con `WEB_ORIGIN`/`FRONTEND_URL` (por defecto `http://localhost:5173`). El frontend se sirve en `http://localhost`, que no está en la lista. | **Bloqueante para el chat**: el handshake se rechaza. | Compose (`WEB_ORIGIN`) |
| 4 | El frontend usa `VITE_API_URL=""` (llamadas al mismo origen). El proxy de rutas (`/auth`, `/posts`, `/socket.io`, …) solo existe en `vite.config.mjs` (servidor de desarrollo). | **Bloqueante**: la SPA en producción no llega a la API. | `Caddyfile` del frontend |
| 5 | El healthcheck del backend usa `curl`, que no viene en `node:22-alpine`. | El backend nunca pasa a `healthy` y el frontend no arranca (`depends_on: service_healthy`). | Dockerfile del backend / compose |
| 6 | El `init-db.sql` del repo Docker está desactualizado (572 líneas contra 693 en el backend; le falta, por ejemplo, `moderation_tasks`). | Esquema incompleto y errores 500 en moderación. | Repo Docker (sincronizar) |
| 7 | La cookie de sesión lleva `secure: true` en producción. | Funciona en `http://localhost`, pero **no** en `http://<IP-servidor>`: hace falta HTTPS fuera de localhost. | Proxy TLS (fase 5) |
| 8 | Faltan variables en el compose: `OPENAI_API_KEY`, `MAIL_TRANSPORT`, `SMTP_*`, `MAIL_FROM`, `WEB_ORIGIN`. `JWT_SECRET` tiene un valor por defecto inseguro. | Correos OTP solo por log y secreto predecible. | Compose + `.env.production.example` |
| 9 | Postgres (`5432`) y backend (`8000`) se publican en todas las interfaces. | Exposición innecesaria en un servidor. | Compose |
| 10 | El README promete "cero tiempo de inactividad". Con `container_name` fijo y recreación, eso no ocurre. | Documentación incorrecta. | README |
| 11 | `init-db.sql` hace `DROP TABLE` y solo corre la **primera vez** que se crea el volumen. No hay migraciones (`synchronize: false`). | Los cambios de esquema posteriores no se aplican solos. | Procedimiento de actualización (fase 6) |

---

## 1. Arquitectura objetivo

```
                 navegador
                     │  :80 (y :443 en la fase 5)
              ┌──────▼───────┐
              │  frontend    │  caddy:2-alpine — SPA + reverse proxy + TLS
              │  (Caddy)     │  /  → index.html (fallback SPA)
              └──┬───────┬───┘  /auth,/posts,... y /socket.io → backend:8000
                 │ red interna luminaul-net
          ┌──────▼──────┐   ┌──────────────┐
          │  backend    │──▶│  postgres    │  volumen luminaul-postgres-data
          │  NestJS:8000│   │  16-alpine   │  init-db.sql (primer arranque)
          └─────────────┘   └──────────────┘
```

- Hay **un solo origen público** (el frontend). La cookie JWT y Socket.io van al mismo origen, así que CORS no interviene.
- Backend y Postgres solo se exponen a `127.0.0.1` (para depurar o usar Swagger desde el host).

---

## 2. Fases y tareas

### Fase 1 — Dockerfiles en los repos de aplicación (bloqueante)

> Archivos de raíz de repo = zona roja: los crea el Tech Lead mediante un PR en cada repo.

**T1.1 `luminaul-new-backend/Dockerfile`** (multi-stage)

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --omit=dev

FROM node:22-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json ./
USER node
EXPOSE 8000
HEALTHCHECK --interval=10s --timeout=5s --retries=5 \
  CMD node -e "fetch('http://localhost:8000/api/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["node", "dist/main"]
```

- Comprobar que `nest build` deja `dist/main.js` (con `tsconfig.build.json`) y que no hay dependencias de runtime en `devDependencies`.
- `.dockerignore`: `node_modules`, `dist`, `.env`, `*.tsbuildinfo`, `test`, `coverage`, `.git`.

**T1.2 `luminaul-frontend/Dockerfile`**

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ARG VITE_API_URL=""
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

FROM caddy:2-alpine
COPY Caddyfile /etc/caddy/Caddyfile
COPY --from=build /app/dist /srv
EXPOSE 80 443
```

- `.dockerignore`: `node_modules`, `dist`, `tests`, `playwright-report`, `test-results`, `.git`.
- `npm run build` ejecuta `tsc --noEmit`: un error de tipos rompe la imagen. Es intencional, porque obliga a tener `main` compilable.
- Caddy es a la vez servidor estático, reverse proxy y terminador TLS: **no hay Nginx**.

**T1.3 `luminaul-frontend/Caddyfile`**: replica el proxy de `vite.config.mjs`.

```caddyfile
# SITE_ADDRESS=":80" en local (HTTP) | "luminaul.midominio.pe" en servidor (HTTPS automático)
{$SITE_ADDRESS::80} {
	encode gzip

	# API REST + Swagger (/api/docs) → backend, sin reescribir la ruta
	handle /api/* {
		reverse_proxy backend:8000
	}

	# WebSocket del chat (Caddy pasa el Upgrade sin configuración extra)
	handle /socket.io/* {
		reverse_proxy backend:8000
	}

	# Todo lo demás es la SPA: archivo estático o fallback a index.html
	handle {
		root * /srv
		try_files {path} /index.html
		file_server
	}
}
```

- **Prefijo `/api` (implementado):** el backend monta toda la API REST bajo `/api` (`setGlobalPrefix` en `src/config/http-app.config.ts`, compartido por `main.ts` y las pruebas e2e) y Swagger queda en `/api/docs`. El frontend usa `API_URL="/api"` por defecto (`src/config/env.ts`). Un controller nuevo funciona sin tocar Caddy ni Vite, y ninguna pantalla de React puede chocar con la API.
- Única regla para el equipo: ninguna pantalla de React puede llamarse `/api/...` ni `/socket.io/...`.

**Criterio de aceptación:** `docker build` de cada repo funciona en local con `docker build -t test ./luminaul-new-backend`.

---

### Fase 2 — Ajustes del `docker-compose.yml` (repo Docker)

**T2.1** Sincronizar `init-db.sql` con `luminaul-new-backend/init-db.sql` (el backend es la fuente de verdad). Opcional: un script `scripts/sync-init-db.sh` que lo descargue de `raw.githubusercontent.com/.../main/init-db.sql`.

**T2.2** Reescribir el compose:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:?Definir DB_PASSWORD en .env}
      POSTGRES_DB: ${DB_NAME:-luminaul}
    ports: ["127.0.0.1:${DB_PORT:-5432}:5432"]
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/01-init-db.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      retries: 10
    networks: [luminaul-net]

  backend:
    build:
      context: https://github.com/LuminaUL-Grupo3/luminaul-new-backend.git#${BACKEND_REF:-main}
    image: luminaul-backend:${BACKEND_REF:-main}
    restart: unless-stopped
    environment:
      NODE_ENV: production
      PORT: 8000
      DATABASE_URL: postgresql://${DB_USER:-postgres}:${DB_PASSWORD}@postgres:5432/${DB_NAME:-luminaul}?sslmode=disable
      JWT_SECRET: ${JWT_SECRET:?Definir JWT_SECRET en .env}
      JWT_EXPIRES_IN: ${JWT_EXPIRES_IN:-86400}
      WEB_ORIGIN: ${PUBLIC_URL:-http://localhost}
      OPENAI_API_KEY: ${OPENAI_API_KEY:-}
      MAIL_TRANSPORT: ${MAIL_TRANSPORT:-console}
      SMTP_HOST: ${SMTP_HOST:-}
      SMTP_PORT: ${SMTP_PORT:-587}
      SMTP_SECURE: ${SMTP_SECURE:-false}
      SMTP_USER: ${SMTP_USER:-}
      SMTP_PASSWORD: ${SMTP_PASSWORD:-}
      MAIL_FROM: ${MAIL_FROM:-}
    ports: ["127.0.0.1:8000:8000"]       # Swagger: http://localhost:8000/api/docs (o http://localhost/api/docs vía Caddy)
    depends_on:
      postgres: { condition: service_healthy }
    networks: [luminaul-net]

  frontend:
    build:
      context: https://github.com/LuminaUL-Grupo3/luminaul-frontend.git#${FRONTEND_REF:-main}
    image: luminaul-frontend:${FRONTEND_REF:-main}
    restart: unless-stopped
    ports: ["${HTTP_PORT:-80}:80"]
    depends_on:
      backend: { condition: service_healthy }
    networks: [luminaul-net]

volumes:
  postgres_data: { name: luminaul-postgres-data }
networks:
  luminaul-net: { name: luminaul-network }
```

Cambios clave:
- `?sslmode=disable` resuelve el problema #2 sin tocar código. **Hay que verificarlo**: node-pg combina el `sslmode` de la URL con la opción `ssl` de TypeORM. Si no funciona, se pide al backend una variable `DATABASE_SSL=false` en `database.config.ts`.
- Los secretos son obligatorios (`:?`), sin valores por defecto.
- Se quita `container_name` (deja escalar y recrear sin choques) y el puerto duplicado `5173:80`.
- `*_REF` admite rama, tag o commit, lo que permite despliegues reproducibles (`BACKEND_REF=v1.2.0`).
- El healthcheck del backend lo define su Dockerfile (problema #5).

**T2.3** Actualizar `.env.production.example` con todas las variables anteriores más `PUBLIC_URL` y `HTTP_PORT`, cada una con un comentario. Generar los secretos con `openssl rand -base64 48`.

**T2.4** Si los repos son **privados**, el build remoto necesita credenciales: `context: https://<token>@github.com/...` (no se recomienda, el token queda en la configuración) o, mejor, SSH (`git@github.com:...git#main` más `docker compose build --ssh default`). Hay que confirmar la visibilidad de los repos.

---

### Fase 3 — Modo local para el equipo (sin GitHub)

**T3.1** `docker-compose.local.yml` (override) que compila desde los clones hermanos:

```yaml
services:
  backend:  { build: { context: ../luminaul-new-backend } }
  frontend: { build: { context: ../luminaul-frontend } }
```

Uso: `docker compose -f docker-compose.yml -f docker-compose.local.yml up -d --build`.
Este override reemplaza al `docker-compose.yml` suelto de la carpeta raíz `GitHub/`, que se puede retirar.

**T3.2** (Opcional) Un `Makefile` o `scripts/*.sh|ps1` con atajos: `up`, `up-local`, `logs`, `reset-db`, `backup`.

---

### Fase 4 — Verificación (smoke tests)

Checklist que se ejecuta después de cada despliegue (automatizable en `scripts/smoke.sh`):

1. `docker compose ps` → los 3 servicios en estado `healthy`.
2. `curl -f http://localhost/api/health` → `{"status":"ok"}` (pasa por Caddy). `http://localhost/api/docs` abre Swagger.
3. `docker compose exec postgres psql -U postgres -d luminaul -c '\dt'` → todas las tablas de `init-db.sql`.
4. Login en la UI con `20210001@aloe.ulima.edu.pe` → aparece la cookie de sesión y una recarga mantiene la sesión.
5. Recargar con F5 rutas profundas de la SPA (`/admin`, detalle de grupo, perfil) → no devuelven JSON del backend.
6. Chat: abrir dos navegadores en el mismo grupo → `chat:new_message` llega en tiempo real (pestaña Network → WS `101 Switching Protocols`).
7. `MAIL_TRANSPORT=console`: el código OTP aparece en `docker compose logs backend`.
8. `docker compose down && docker compose up -d` → los datos persisten.

---

### Fase 5 — Producción en servidor (HTTPS)

Hace falta porque la cookie es `secure` (problema #7).

**T5.1** El contenedor `frontend` ya es Caddy, así que no se añade ningún servicio. Solo cambian variables y puertos:

```yaml
  frontend:
    environment:
      SITE_ADDRESS: ${SITE_ADDRESS:-:80}     # en servidor: luminaul.midominio.pe
    ports: ["80:80", "443:443"]
    volumes:
      - caddy_data:/data                      # certificados Let's Encrypt (persistir)
```

Con un dominio que apunte al servidor y los puertos 80/443 abiertos, Caddy obtiene y renueva el certificado solo. `PUBLIC_URL=https://<dominio>`.
Sin dominio (IP de laboratorio), las alternativas son un certificado autofirmado (`tls internal`) o, solo en entornos de prueba, desactivar `secure` con una variable (cambio en `auth.controller.ts`, que pertenece a Dev 1).

**T5.2** Endurecimiento: límites de memoria (`deploy.resources.limits`), `logging: { driver: json-file, options: { max-size: 10m, max-file: "3" } }` y `read_only: true` en el frontend.

---

### Fase 6 — Operación: actualizaciones, esquema y respaldos

**T6.1 Actualizar código:** `docker compose build --pull && docker compose up -d`.
Para fijar una versión: `BACKEND_REF=<tag> FRONTEND_REF=<tag>` en `.env`. Rollback: volver al tag anterior y repetir.

**T6.2 Cambios de esquema:** `init-db.sql` **no se vuelve a ejecutar** sobre un volumen existente.
- En entornos de prueba: `docker compose down -v && docker compose up -d` (borra todos los datos).
- En entornos con datos: a corto plazo, scripts incrementales `migrations/NNN_*.sql` aplicados con `docker compose exec -T postgres psql ... < migrations/NNN.sql`. A medio plazo, migraciones TypeORM en el backend (decisión del Tech Lead).

**T6.3 Backups:**
```bash
docker compose exec -T postgres pg_dump -U postgres -Fc luminaul > backups/luminaul_$(date +%F).dump
docker compose exec -T postgres pg_restore -U postgres -d luminaul --clean < backups/<archivo>.dump
```
Añadir `backups/` al `.gitignore`.

---

### Fase 7 — CI (opcional, recomendado)

- GitHub Action en cada repo de aplicación: `docker build` en cada PR, para que `main` siempre compile.
- Publicar imágenes en GHCR (`ghcr.io/luminaul-grupo3/backend:<sha>`). Así el compose de producción pasa de `build:` a `image:` y el servidor solo hace `docker compose pull && up -d`, sin compilar ni depender de GitHub en tiempo de build.

---

### Fase 8 — Documentación

**T8.1** Reescribir el `README.md` del repo Docker: requisitos, `.env`, modo remoto y local, smoke tests, actualización, backups y troubleshooting (SSL de BD, origen de WebSocket, cookie sobre HTTP). Quitar la frase "cero tiempo de inactividad".

---

## 3. Orden de ejecución y dependencias

| Orden | Tarea | Depende de | Repo |
|-------|-------|-----------|------|
| 1 | T1.1, T1.2, T1.3 Dockerfiles + Caddyfile | — | backend / frontend |
| 2 | T2.1 sincronizar init-db.sql | — | docker |
| 3 | T2.2, T2.3 compose + .env example | 1 | docker |
| 4 | T3.1 override local | 3 | docker |
| 5 | Fase 4 smoke tests (modo local) | 1–4 | — |
| 6 | T2.4 acceso a repos + smoke en modo remoto | 5 | — |
| 7 | Fase 5 HTTPS | 6 | docker |
| 8 | Fases 6–8 operación, CI, docs | 6 | todos |

Las fases 1 a 4 forman el **MVP**: con ellas la aplicación se despliega en `http://localhost`.

## 4. Decisiones pendientes (Tech Lead)

1. ~~¿Montar la API bajo `/api`?~~ **Hecho:** API en `/api/*` y Swagger en `/api/docs`. Actualizar las rutas de `SPECS.md` cuando convenga (los contratos se siguen escribiendo sin prefijo, como rutas de controller).
2. Visibilidad de los repos de GitHub (pública o privada), que define cómo autentica el build remoto.
3. ¿Hay dominio para HTTPS o se despliega por IP?
4. Estrategia de migraciones de esquema (SQL incremental o migraciones TypeORM).
5. ¿Compilar en el servidor (`build:` remoto) o publicar imágenes en GHCR?
