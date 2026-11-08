# LuminaUL - Repositorio de Despliegue y Orquestación Docker

Repositorio oficial de infraestructura, contenedores y despliegue continuo para la plataforma **LuminaUL** (Universidad de Lima).

Este repositorio orquesta la arquitectura de producción mediante **Docker Compose**, **Caddy 2** (Reverse Proxy + TLS automático con Let's Encrypt), **PostgreSQL 16** y **NestJS 12**.

* **Backend:** [LuminaUL-Grupo3/luminaul-new-backend](https://github.com/LuminaUL-Grupo3/luminaul-new-backend)
* **Frontend:** [LuminaUL-Grupo3/luminaul-frontend](https://github.com/LuminaUL-Grupo3/luminaul-frontend)

---

## 1. Requisitos Previos

* **Docker Engine:** Versión 24 o superior.
* **Docker Compose Plugin:** Versión 2.20 o superior.
* **Memoria SWAP:** En servidores con 1 GB o 2 GB de RAM (como AWS EC2 `t3.small` o `t2.micro`), es **obligatorio configurar 4 GB de SWAP** antes de compilar para evitar bloqueos por falta de memoria.

---

## 2. Puesta en Marcha en Producción

### 2.1 Configuración de Variables de Entorno (.env)
Copia la plantilla y configura tus credenciales reales:
```bash
cp .env.production.example .env
nano .env
```

Parámetros clave a configurar en `.env`:
* `DOMAIN=luminaul.duckdns.org` (o tu dominio público).
* `DUCKDNS_SUBDOMAIN=luminaul` y `DUCKDNS_TOKEN=tu-token` (para resolución dinámica de IP en AWS Academy).
* `WEB_ORIGIN=https://luminaul.duckdns.org,http://luminaul.duckdns.org,http://localhost,https://localhost`.
* `JWT_SECRET` (mínimo 32 caracteres aleatorios en producción).
* `MAIL_TRANSPORT=console` (para ver códigos OTP en los logs) o `smtp` (para envíos reales).

### 2.2 Compilar e Iniciar los Servicios (Compilación Secuencial Segura)
> [!IMPORTANT]
> En servidores de nube pequeños, **compila siempre los servicios uno por uno** de forma secuencial. Compilarlos en paralelo satura la CPU y puede congelar la conexión SSH.

```bash
# 1. Compilar el backend NestJS (~20 segundos)
docker compose build backend

# 2. Compilar el frontend React 19 + Caddy (~40 segundos)
docker compose build frontend

# 3. Levantar los 3 contenedores en segundo plano
docker compose up -d
```

---

## 3. Servicios y Puertos Expuestos

| Servicio | URL / Puerto | Descripción |
| :--- | :--- | :--- |
| **Frontend Web (HTTPS)** | `https://<tu-subdominio>.duckdns.org` (:443) | Aplicación React 19 servida por Caddy con **certificado TLS oficial de Let's Encrypt (candado verde)** |
| **Frontend Web (HTTP)** | `http://<tu-subdominio>.duckdns.org` (:80) | Redirección automática a HTTPS y validación de desafíos ACME |
| **API REST Backend** | `/api/*` (enrutado por Caddy al backend :8000) | Endpoints de NestJS (Auth, Perfiles, Posts, Reviews, Admin) |
| **WebSockets Chat** | `/socket.io/*` (enrutado por Caddy con WSS) | Pasarela de mensajería grupal en tiempo real (SPEC-04) |
| **Swagger UI Docs** | `https://<tu-subdominio>.duckdns.org/api/docs` | Documentación interactiva de OpenAPI |
| **PostgreSQL 16** | Puerto interno `5432` (oculto en red interna) | Base de datos relacional inicializada automáticamente con 22 tablas y 52 cursos |

---

## 4. ¿Cómo actualizar cuando hay cambios nuevos en `main`?

Cuando tú o tu equipo suban nuevos commits o features a la rama `main` en GitHub, actualiza el servidor ejecutando estos 3 comandos:

```bash
cd /home/ubuntu/luminaul-docker

# 1. Descarga el commit más reciente y compila el backend
docker compose build --pull backend

# 2. Descarga el commit más reciente y compila el frontend
docker compose build --pull frontend

# 3. Aplica los cambios y reinicia los contenedores (tarda 2 segundos)
docker compose up -d
```

### Ventajas de este flujo:
* **`--pull`:** Garantiza que Docker descargue el commit más reciente de GitHub sin tener que clonar repositorios manualmente.
* **Secuencial y estable:** Al compilar primero uno y luego el otro, la memoria RAM nunca supera los límites del servidor.
* **Cero pérdida de datos:** La base de datos PostgreSQL, los volúmenes persistentes y los certificados TLS se conservan intactos.

---

## 5. Arranque Automático en AWS Academy (Systemd + DuckDNS)

AWS Academy reasigna la dirección IP pública cada vez que inicias el laboratorio. Para que el servidor reporte su nueva IP a DuckDNS y levante los contenedores automáticamente al encender:

```bash
sudo bash -c 'cat << "EOF" > /etc/systemd/system/luminaul.service
[Unit]
Description=LuminaUL Docker Compose Application
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/ubuntu/luminaul-docker
User=ubuntu
Group=docker
ExecStartPre=/bin/bash -c "set -a; [ -f /home/ubuntu/luminaul-docker/.env ] && . /home/ubuntu/luminaul-docker/.env; set +a; IP=\$(curl -s http://checkip.amazonaws.com || curl -s http://169.254.169.254/latest/meta-data/public-ipv4); if [ -n \"\$IP\" ] && [ -n \"\$DUCKDNS_TOKEN\" ]; then curl -s \"https://www.duckdns.org/update?domains=\${DUCKDNS_SUBDOMAIN:-luminaul}&token=\$DUCKDNS_TOKEN&ip=\$IP\"; sleep 5; fi"
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF'

sudo systemctl daemon-reload
sudo systemctl enable luminaul.service
sudo systemctl enable docker
```

---

## 6. Comandos Útiles de Operación y Diagnóstico

```bash
# Ver el estado y healthchecks de los contenedores
docker compose ps

# Ver logs de emisión de certificados TLS en Caddy
docker compose logs -f frontend

# Ver logs del backend (incluye códigos OTP cuando MAIL_TRANSPORT=console)
docker compose logs -f backend

# Reiniciar únicamente el servidor web Caddy (si cambias el Caddyfile)
docker compose restart frontend

# Detener los servicios preservando todos los datos
docker compose down

# Reiniciar completamente la base de datos limpia desde cero (borra datos)
docker compose down -v && docker compose up -d
```
