# LuminaUL - Repositorio de Despliegue y Orquestación Cloud (Producción)

Repositorio oficial de infraestructura, contenedores y despliegue continuo de grado de industria para la plataforma **LuminaUL** (Universidad de Lima).

Orquesta la arquitectura de producción mediante **Docker Compose**, **Caddy 2** (Reverse Proxy + TLS automático con Let's Encrypt), **PostgreSQL 16** y **NestJS 12**.

* **Backend:** [LuminaUL-Grupo3/luminaul-new-backend](https://github.com/LuminaUL-Grupo3/luminaul-new-backend)
* **Frontend:** [LuminaUL-Grupo3/luminaul-frontend](https://github.com/LuminaUL-Grupo3/luminaul-frontend)

---

## 1. Requisitos Previos y Arquitectura

* **Docker Engine:** Versión 24 o superior.
* **Docker Compose Plugin:** Versión 2.20 o superior.
* **Instancia EC2 y Memoria SWAP:** CloudFormation aprovisiona por defecto una instancia **`t3.medium`** (4 GB RAM; soporta también `t3.small`) y configura **2 GB de SWAP** + rotación automática de logs y parámetros `sysctl`. Si despliegas manualmente en una instancia pequeña (`t3.small`), configura al menos 2–4 GB de SWAP antes de compilar.

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
   * **2. Correo Transaccional (OTP):**
     * `MailTransport`: `console` (imprime OTPs en logs, ideal para pruebas) o `smtp` (envío real).
     * `SmtpUser`: Tu correo emisor (ej: `tu-cuenta@gmail.com`).
     * `SmtpPassword`: Contraseña de aplicación de 16 caracteres de Google *(campo oculto)*.
     * `SmtpHost`: `smtp.gmail.com` (por defecto).
     * `SmtpPort`: `587` (por defecto).
     * `SmtpSecure`: `false` (por defecto para puerto 587 con STARTTLS).
     * `MailFrom`: `LuminaUL Soporte <soporte@luminaul.pe>`.
   * **3. Ramas de GitHub:**
     * `FrontendBranch`: Rama de producción del frontend (ej: `main`).
     * `BackendBranch`: Rama de producción del backend (ej: `main`).
   * **4. Seguridad y Cómputo:**
     * `OpenAiApiKey`: Opcional para moderación semántica en segundo plano *(campo oculto)*.
     * `InstanceType`: `t3.medium` por defecto (permite `t3.small`).
     * *Nota:* El `JWT_SECRET` de producción se genera aleatoriamente por instancia en `UserData` con `openssl rand -base64 48`, y la pila usa `WaitCondition` (`420s`) para confirmar que los contenedores y `/health` arrancaron correctamente antes de marcar `CREATE_COMPLETE`.
     * *CORS:* El backend construye automáticamente la lista blanca usando `DOMAIN`, `WEB_ORIGIN` y `FRONTEND_URL`.
3. Haz clic en **Create stack**.
4. Una vez completada la espera de validación, LuminaUL estará activo con certificado TLS oficial en `https://<tu-subdominio>.duckdns.org`.

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

#### 2.3 Descarga de Imágenes y Arranque (`ghcr.io`)
> [!IMPORTANT]
> Las imágenes de producción se compilan automáticamente en **GitHub Actions** y se publican en **GitHub Container Registry (`ghcr.io`)** cada vez que hay un `push` a `main` en `luminaul-new-backend` o `luminaul-frontend`. La instancia EC2 **no compila código**: solo descarga las imágenes listas.

```bash
# 1. Descargar las imágenes precompiladas desde ghcr.io
docker compose pull

# 2. Levantar los 3 contenedores en segundo plano
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

El flujo usa **build automático en GitHub Actions + deploy manual en EC2** (ideal para AWS Academy, donde la IP de EC2 cambia en cada reinicio del laboratorio y no conviene guardarla en GitHub Secrets):

1. Al hacer `push` a `main` en `luminaul-new-backend` o `luminaul-frontend`, el workflow `.github/workflows/docker-publish.yml` compila y publica automáticamente la nueva imagen `:latest` en `ghcr.io`.
2. Entra por SSH a tu instancia EC2 y ejecuta:

```bash
cd /home/ubuntu/luminaul-docker

# 1. Descargar las imágenes actualizadas desde ghcr.io
docker compose pull

# 2. Recrear únicamente los contenedores que cambiaron
docker compose up -d
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
