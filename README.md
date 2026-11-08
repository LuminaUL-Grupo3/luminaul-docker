# LuminaUL - Repositorio de Despliegue y Orquestación Cloud (Producción)

Repositorio oficial de infraestructura, contenedores y despliegue continuo de grado de industria para la plataforma **LuminaUL** (Universidad de Lima).

Orquesta la arquitectura de producción mediante **Docker Compose**, **Caddy 2** (Reverse Proxy + TLS automático con Let's Encrypt), **PostgreSQL 16** y **NestJS 12**.

* **Backend:** [LuminaUL-Grupo3/luminaul-new-backend](https://github.com/LuminaUL-Grupo3/luminaul-new-backend)
* **Frontend (PWA):** [LuminaUL-Grupo3/luminaul-frontend](https://github.com/LuminaUL-Grupo3/luminaul-frontend)

---

## 1. Arquitectura de Producción

* **Modo Estricto de Producción:** `NODE_ENV=production` enforzado en todas las capas.
* **Correo Transaccional Real:** `MAIL_TRANSPORT=smtp` obligatorio para el despacho de códigos OTP de autenticación institucional.
* **Seguridad de Red (OWASP / CIS):**
  - **IMDSv2 Obligatorio:** Mitigación de ataques SSRF contra metadatos de AWS.
  - **Rotación de Logs:** Docker daemon configurado con límite de 30 MB (`json-file`) para prevenir la saturación del disco EBS.
  - **Reverse Proxy Caddy:** HSTS (1 año + preload), `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, y cabeceras `X-Forwarded-For` para soporte nativo de `trust proxy` en NestJS.
  - **Base de Datos Aislada:** PostgreSQL (`DATABASE_SSL=false`) opera estrictamente en la red interna privada de Docker (`luminaul-net`), sin exposición a internet.
* **Frontend PWA Móvil:** Progressive Web App instalable con soporte offline y banner interactivo exclusivo para celulares y tablets (Android e iOS).

---

## 2. Despliegue en AWS Academy (Elige tu Opción)

### Opción A: Despliegue Automatizado con CloudFormation (Recomendado - Cero `nano`)

Utiliza la plantilla de grado de industria [`luminaul-cloudformation.yaml`](./luminaul-cloudformation.yaml). CloudFormation creará toda la infraestructura y te solicitará los parámetros en un formulario web visual:

1. En la consola de AWS, dirígete a **CloudFormation** $\rightarrow$ **Create stack** (with new resources) $\rightarrow$ **Upload a template file** y sube `luminaul-cloudformation.yaml`.
2. Completa los campos en el asistente visual de AWS:
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
     * `FrontendBranch`: `main` (o rama específica como `feature/mobile-PWA`).
     * `BackendBranch`: `main`.
3. Haz clic en **Create stack**.
4. **Listo.** En ~4 minutos, CloudFormation:
   * Crea el Security Group y la instancia EC2 con 30 GB gp3 y 4 GB de SWAP.
   * Genera el archivo `.env` automáticamente con permisos seguros (`chmod 600`).
   * Registra la IP en DuckDNS, compila los contenedores e inicializa LuminaUL en HTTPS con certificado TLS oficial.

---

### Opción B: Despliegue Manual por SSH

Si prefieres aprovisionar manualmente dentro de una instancia Ubuntu 24.04 LTS:

#### 2.1 Preparar Memoria SWAP de 4GB
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
> En servidores pequeños (`t3.small` / `t2.micro`), **compila siempre los servicios uno por uno**. Compilar en paralelo satura el procesador y puede congelar la sesión SSH.

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
| **Frontend PWA (HTTPS)** | `https://<subdominio>.duckdns.org` (:443) | Aplicación React 19 servida por Caddy con **TLS oficial Let's Encrypt (candado verde)** |
| **Frontend (HTTP)** | `http://<subdominio>.duckdns.org` (:80) | Redirección 301 automática a HTTPS y validación ACME |
| **API REST Backend** | `/api/*` (enrutado a NestJS :8000) | Endpoints de negocio (Auth, Perfil, Posts, Reseñas, Moderación, Admin) |
| **WebSockets Chat** | `/socket.io/*` (WSS en tiempo real) | Pasarela de chat grupal con control anti-spam (SPEC-04) |
| **Swagger UI** | `https://<subdominio>.duckdns.org/api/docs` | Documentación interactiva de la API |
| **PostgreSQL 16** | Puerto interno `5432` | Base de datos relacional aislada en la red privada de Docker |

---

## 4. Instalación como PWA en Dispositivos Móviles

LuminaUL incluye soporte PWA completo con un banner interactivo que **se activa exclusivamente en smartphones y tablets táctiles**:

* **En Android (Google Chrome):**  
  Aparece el banner inferior *"Instala LuminaUL - Acceso rápido sin abrir navegador"*. Al presionar el botón **"Instalar"**, el navegador abre el diálogo nativo de instalación en el dispositivo.
* **En iPhone / iPad (Safari iOS):**  
  Muestra las instrucciones guiadas: *"Toca Compartir [↑] y luego 'Añadir a inicio' ➕"*.
* **En PC / Laptops con mouse:**  
  El banner se apaga automáticamente para no ser invasivo.

---

## 5. ¿Cómo actualizar cuando hay cambios nuevos en GitHub?

Cuando subas nuevos commits a las ramas de producción:

```bash
cd /home/ubuntu/luminaul-docker

# 1. Reconstruir el backend sin caché
docker compose build --no-cache backend

# 2. Reconstruir el frontend sin caché (especificando rama si usas una distinta a main)
FRONTEND_BRANCH=feature/mobile-PWA docker compose build --no-cache frontend

# 3. Reiniciar contenedores
docker compose up -d backend frontend
```

---

## 6. Procedimiento para Apagar y Reanudar el Laboratorio (AWS Academy)

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

## 7. Comandos de Diagnóstico y Operación

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
