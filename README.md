# LuminaUL - Repositorio de Despliegue y Orquestación Docker

Repositorio de infraestructura y despliegue continuo para la plataforma **LuminaUL** (Universidad de Lima).

Este repositorio **no contiene código fuente duplicado**. Utiliza las capacidades nativas de compilación remota de Docker Compose para compilar e iniciar directamente las versiones más recientes desde los repositorios oficiales de la organización:

* **Backend:** [LuminaUL-Grupo3/luminaul-new-backend](https://github.com/LuminaUL-Grupo3/luminaul-new-backend)
* **Frontend:** [LuminaUL-Grupo3/luminaul-frontend](https://github.com/LuminaUL-Grupo3/luminaul-frontend)

---

## 1. Requisitos Previos

* **Docker:** Versión 24 o superior.
* **Docker Compose:** Versión 2.20 o superior (incluido con Docker Desktop o `docker-compose-plugin`).
* Conexión a Internet para descargar las imágenes base y el código de GitHub.

---

## 2. Puesta en Marcha en Producción

### 2.1 Configuración de Variables (Opcional)
Para cambiar credenciales de base de datos o secretos de sesión:
```bash
cp .env.production.example .env
```

Si deseas compilar una rama específica diferente a `main` (por ejemplo una rama de prueba o feature), puedes definirla en `.env`:
```env
BACKEND_BRANCH=main
FRONTEND_BRANCH=main
```

### 2.2 Compilar e Iniciar los Servicios
Ejecuta el siguiente comando para que Docker descargue la última versión de ambos repositorios, construya las imágenes de producción y levante los contenedores:

```bash
docker compose build --no-cache && docker compose up -d
```

---

## 3. Servicios Disponibles

Una vez iniciados los contenedores:

| Servicio | URL Local | Descripción |
| :--- | :--- | :--- |
| **Frontend Web** | `http://localhost` (puertos 80 y 5173) | Aplicación React 19 servida por Nginx con soporte SPA y proxy reverso |
| **API REST Backend** | `http://localhost:8000` | Endpoints de NestJS |
| **Swagger UI** | `http://localhost:8000/api/docs` | Documentación interactiva de la API (toda la API REST vive bajo `/api`) |
| **PostgreSQL 16** | `localhost:5432` | Base de datos relacional inicializada automáticamente |

---

## 4. ¿Cómo actualizar cuando hay una nueva versión del código?

Cuando el equipo suba nuevos commits o cambios a GitHub (en el backend o en el frontend), simplemente ejecuta en este repositorio:

```bash
docker compose build --no-cache && docker compose up -d
```

Docker descargará los últimos cambios de GitHub en memoria, reconstruirá las imágenes y reiniciará los servicios con **cero tiempo de inactividad** sin tener que clonar ni manipular código fuente manualmente.

---

## 5. Comandos Útiles de Operación

```bash
# Ver el estado y healthchecks de los contenedores
docker compose ps

# Ver logs en tiempo real del backend
docker compose logs -f backend

# Ver logs del frontend (Nginx)
docker compose logs -f frontend

# Detener los servicios preservando los datos de PostgreSQL
docker compose down

# Detener y reiniciar la base de datos limpia desde cero
docker compose down -v
```

