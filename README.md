# LuminaUL - Repositorio de Despliegue y Orquestación Cloud (Producción)

Repositorio oficial de infraestructura, contenedores y despliegue continuo de grado de industria para la plataforma **LuminaUL** (Universidad de Lima).

Orquesta la arquitectura de producción mediante **Docker Compose**, **Caddy 2** (Reverse Proxy + TLS automático con Let's Encrypt), **PostgreSQL 16** y **NestJS 12**.

* **Backend:** [LuminaUL-Grupo3/luminaul-new-backend](https://github.com/LuminaUL-Grupo3/luminaul-new-backend)
* **Frontend:** [LuminaUL-Grupo3/luminaul-frontend](https://github.com/LuminaUL-Grupo3/luminaul-frontend)

---

## 1. Requisitos Previos y Arquitectura

* **Docker Engine:** Versión 24 o superior.
* **Docker Compose Plugin:** Versión 2.20 o superior.
* **Memoria SWAP:** En instancias con 1 GB o 2 GB de RAM (`t3.small` / `t2.micro`), es **obligatorio configurar 4 GB de SWAP** antes de compilar para evitar congelamientos por Out-of-Memory.
* **Seguridad de Red (OWASP / CIS):**
  - **IMDSv2 Obligatorio:** Mitigación de ataques SSRF contra metadatos de AWS.
  - **Rotación de Logs:** Docker daemon configurado con límite de 30 MB (`json-file`) para prevenir saturación del disco EBS.
  - **Reverse Proxy Caddy:** HSTS, `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, y cabeceras `X-Forwarded-For` para soporte de `trust proxy` en NestJS.
  - **Base de Datos Aislada:** PostgreSQL (`DATABASE_SSL=false`) opera estrictamente en la red interna privada de Docker (`luminaul-net`), sin exposición a internet.

---

## 2. Instrucciones de Despliegue

### Opción A: Despliegue Automatizado con CloudFormation (Recomendado - Cero `nano`)

Utiliza la plantilla [`luminaul-cloudformation.yaml`](./luminaul-cloudformation.yaml). CloudFormation aprovisiona toda la infraestructura y solicita las credenciales mediante un formulario web en la consola de AWS:

1. En la consola de AWS, dirígete a **CloudFormation** $\rightarrow$ **Create stack** (with new resources) $\rightarrow$ **Upload a template file** y sube `luminaul-cloudformation.yaml`.
2. Completa los parámetros en el formulario:
   * **1. Red y Dominio Público:**
     * `DuckDnsSubdomain`: Tu subdominio en DuckDNS (ej: `luminaul`).
     * `DuckDnsToken`: Tu token secreto de DuckDNS *(campo oculto)*.
     * `KeyName`: Tu par de claves EC2 (`.pem`).
   * **2. Correo Transaccional Real (SMTP):**
     * `SmtpUser`: Tu correo emisor (ej: `tu-cuenta@gmail.com`).
     * `SmtpPassword`: Contraseña de aplicación de 16 caracteres de Google *(campo oculto)*.
     * `SmtpHost`: `smtp.gmail.com` (por defecto).
     * `SmtpPort`: `587` (por defecto).
     * `SmtpSecure`: `false` (por defecto para puerto 587 con STARTTLS).
     * `MailFrom`: `LuminaUL Soporte <soporte@luminaul.pe>`.
   * **3. Ramas de GitHub:**
     * `FrontendBranch`: Rama de producción del frontend (ej: `main`).
     * `BackendBranch`: Rama de producción del backend (ej: `main`).
3. Haz clic en **Create stack**.
4. En ~4 minutos la pila finalizará y LuminaUL estará activo con certificado TLS oficial en `https://<tu-subdominio>.duckdns.org`.

---

### Opción B: Despliegue Manual por SSH

Si aprovisionas directamente dentro de una instancia Ubuntu 24.04 LTS:

#### 2.1 Configurar Memoria SWAP de 4GB
```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

#### 2.2 Clonar y Configurar `.env`
```bash
git clone https://github.com/LuminaUL-Grupo3/luminaul-docker.git
cd luminaul-docker
cp .env.production.example .env
nano .env
```

Configura tus credenciales reales en `.env`:
* `DOMAIN=luminaul.duckdns.org`
* `DUCKDNS_SUBDOMAIN=luminaul` y `DUCKDNS_TOKEN=tu-token`
* `MAIL_TRANSPORT=smtp` con tus credenciales `SMTP_USER` y `SMTP_PASSWORD`
* `JWT_SECRET` (mínimo 32 caracteres)

#### 2.3 Compilación Secuencial Segura
> [!IMPORTANT]
> En servidores pequeños (`t3.small` / `t2.micro`), **compila siempre los servicios uno por uno** de forma secuencial para evitar saturar el procesador.

```bash
# 1. Compilar backend NestJS
docker compose build backend

# 2. Compilar frontend React 19 + Caddy
docker compose build frontend

# 3. Levantar los 3 contenedores en segundo plano
docker compose up -d
```

---

## 3. Servicios y Puertos Expuestos

| Servicio | URL / Puerto | Descripción |
| :--- | :--- | :--- |
| **Frontend Web (HTTPS)** | `https://<subdominio>.duckdns.org` (:443) | Aplicación React 19 servida por Caddy con **TLS oficial Let's Encrypt (candado verde)** |
| **Frontend Web (HTTP)** | `http://<subdominio>.duckdns.org` (:80) | Redirección 301 automática a HTTPS y validación ACME |
| **API REST Backend** | `/api/*` (enrutado a NestJS :8000) | Endpoints de negocio (Auth, Perfiles, Publicaciones, Reseñas, Moderación, Admin) |
| **WebSockets Chat** | `/socket.io/*` (WSS en tiempo real) | Pasarela de chat grupal con control anti-spam (SPEC-04) |
| **Swagger UI** | `https://<subdominio>.duckdns.org/api/docs` | Documentación interactiva de la API |
| **PostgreSQL 16** | Puerto interno `5432` | Base de datos relacional aislada en la red privada de Docker |

---

## 4. ¿Cómo actualizar cuando hay cambios nuevos en GitHub?

Cuando se suban nuevos commits a las ramas de producción:

```bash
cd /home/ubuntu/luminaul-docker

# 1. Reconstruir el backend sin caché
docker compose build --no-cache backend

# 2. Reconstruir el frontend sin caché
docker compose build --no-cache frontend

# 3. Reiniciar contenedores
docker compose up -d backend frontend
```

---

## 5. Procedimiento para Apagar y Reanudar el Laboratorio (AWS Academy)

### Para Apagar (Evitar consumo de créditos):
> [!CAUTION]
> **NO ejecutes `docker compose down`** al salir. Si por descuido escribieras `docker compose down -v`, eliminarías los volúmenes de PostgreSQL y los certificados SSL de Caddy.
* Simplemente haz clic en **"End Lab"** en el portal de AWS Academy (o ve a la consola de AWS $\rightarrow$ EC2 $\rightarrow$ **Stop instance**).
* Linux enviará una señal limpia de apagado a Docker cerrando transacciones sin corrupción de datos.

### Para Reanudar el Laboratorio:
1. Haz clic en **"Start Lab"** en el portal de AWS Academy.
2. Espera ~2 minutos a que el círculo se ponga verde.
3. El servicio automático `luminaul.service` detectará la nueva IP asignada por AWS, actualizará DuckDNS y levantará los contenedores automáticamente.
4. Ingresas directo a `https://<tu-subdominio>.duckdns.org`.

---

## 6. Comandos de Diagnóstico y Operación

```bash
# Ver estado y healthchecks de los contenedores
docker compose ps

# Ver logs de emisión de certificados TLS en Caddy
docker compose logs -f frontend

# Ver logs del backend NestJS
docker compose logs -f backend

# Reiniciar únicamente Caddy tras editar Caddyfile
docker compose restart frontend

# Detener los servicios preservando todos los datos
docker compose stop

# Reconstruir base de datos limpia desde cero (¡BORRA DATOS!)
docker compose down -v && docker compose up -d
```
