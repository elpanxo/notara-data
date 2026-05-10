# notara-data

Repositorio de la capa de datos de **Notara**. Contiene la configuración Docker Compose para levantar todas las bases de datos y servicios de persistencia que utilizan los microservicios del backend.

---

## Servicios incluidos

| Servicio | Imagen | Puerto | Uso |
|---|---|---|---|
| `postgres-usuarios` | `postgres:16-alpine` | `5432` | Datos de usuarios y autenticación |
| `postgres-notas-metas` | `postgres:16-alpine` | `5433` | Notas y metas de los usuarios |
| `mongodb` | `mongo:7-jammy` | `27017` | Caché de canciones y letras de Spotify |
| `redis` | `redis:7-alpine` | `6379` | Caché de tokens JWT y sesiones |

---

## Persistencia de datos

Todos los servicios utilizan **named volumes** de Docker para garantizar que los datos sobrevivan a reinicios y recreaciones de contenedores.

| Volumen | Servicio | Datos que protege |
|---|---|---|
| `postgres_usuarios_data` | PostgreSQL usuarios | Cuentas, emails, contraseñas hasheadas |
| `postgres_notas_metas_data` | PostgreSQL notas | Notas y metas de aprendizaje |
| `mongo_data` | MongoDB | Metadatos de canciones, letras cacheadas |
| `redis_data` | Redis | Tokens de sesión, caché temporal |

### ¿Por qué named volumes y no bind mounts?

Los **named volumes** son gestionados completamente por Docker, lo que significa que funcionan igual en cualquier sistema operativo (Linux, Windows, macOS) y en cualquier instancia EC2, sin depender de rutas absolutas del host. Esto hace el despliegue reproducible y portátil.

Los **bind mounts** acoplan el contenedor a una ruta específica del sistema de archivos del host, lo que puede causar problemas de permisos en EC2 y dificulta la migración entre instancias.

---

## Levantar localmente

### Requisitos previos

- Docker Desktop instalado y corriendo

### Pasos

```bash
# 1. Clonar el repositorio
git clone https://github.com/elpanxo/notara-data.git
cd notara-data

# 2. Levantar todos los servicios de datos
docker compose up -d

# 3. Verificar que todos los servicios tienen healthcheck OK
docker compose ps
```

Todos los servicios incluyen healthcheck. El estado debe ser `healthy` antes de levantar el backend.

---

## Healthchecks

Cada servicio verifica su disponibilidad antes de que los microservicios dependientes inicien:

```yaml
# PostgreSQL
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres"]
  interval: 10s
  timeout: 5s
  retries: 5

# MongoDB
healthcheck:
  test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
  interval: 10s
  timeout: 5s
  retries: 5

# Redis
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 10s
  timeout: 3s
  retries: 5
```

---

## Configuración de bases de datos

### PostgreSQL usuarios

```
Host:     localhost (o postgres-usuarios dentro de Docker)
Puerto:   5432
Base de datos: notara-usuarios-db
Usuario:  postgres
Password: password
```

### PostgreSQL notas y metas

```
Host:     localhost (o postgres-notas-metas dentro de Docker)
Puerto:   5433
Base de datos: notara-notas-metas-db
Usuario:  postgres
Password: password
```

### MongoDB

```
Host:     localhost (o mongodb dentro de Docker)
Puerto:   27017
Base de datos: linguaflow (creada automáticamente)
```

### Redis

```
Host:     localhost (o redis dentro de Docker)
Puerto:   6379
Sin autenticación en entorno de desarrollo
```

---

## Pipeline CI/CD

El repositorio cuenta con un pipeline en **GitHub Actions** (`.github/workflows/deploy-data.yml`) que se activa al hacer push sobre la rama `deploy`.

### Flujo del pipeline

```
Push a rama deploy
       │
       ▼
  Checkout código
       │
       ▼
  Deploy en EC2 Data via AWS SSM
       │
       ├── git clone / git pull del repositorio
       ├── docker compose down
       └── docker compose up -d
       │
       ▼
  Verificar estado del comando SSM
```

El deploy se realiza mediante **AWS SSM (Systems Manager)**, lo que permite ejecutar comandos en la instancia EC2 sin necesidad de abrir el puerto SSH (22), mejorando la seguridad del entorno.

### Secrets requeridos en GitHub

Configurar en `Settings > Secrets and variables > Actions`:

```
EC2_DATA_INSTANCE_ID   # ID de la instancia EC2 dedicada a bases de datos
```

Las credenciales AWS se configuran en el runner self-hosted mediante IAM Role.

---

## Despliegue en AWS EC2

Este servicio corre en una instancia EC2 **en subred privada**, sin acceso directo desde Internet. Solo acepta conexiones desde la instancia EC2 del backend a través de Security Groups de AWS.

```
Internet
   │  (sin acceso directo)
   │
[EC2 Frontend - IP pública]
   │
   ▼
[EC2 Backend - IP privada]
   │  (Security Group: acceso a puertos 5432, 5433, 27017, 6379
   │   solo desde EC2 Backend)
   ▼
[EC2 Data - IP privada]
   ├── PostgreSQL :5432
   ├── PostgreSQL :5433
   ├── MongoDB    :27017
   └── Redis      :6379
```

---

## Comandos útiles

```bash
# Ver logs de un servicio específico
docker compose logs -f postgres-usuarios

# Conectarse a PostgreSQL usuarios
docker exec -it notara-postgres-usuarios psql -U postgres -d notara-usuarios-db

# Conectarse a MongoDB
docker exec -it notara-mongo mongosh

# Conectarse a Redis
docker exec -it notara-redis redis-cli

# Detener todos los servicios (datos se preservan en volumes)
docker compose down

# Detener y eliminar volumes (CUIDADO: borra todos los datos)
docker compose down -v
```
