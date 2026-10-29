# MS Vehículos CI/CD con GitHub Actions y AWS EC2

Proyecto de microservicio REST desarrollado con **Spring Boot 4**, **Java 21** y **Maven**, conectado a una base de datos **MySQL 8**. Se utiliza **Docker** para empaquetar la aplicación, **Docker Compose** para ejecutar los servicios y **GitHub Actions** para automatizar las pruebas, el análisis de calidad, la construcción y publicación de la imagen y el despliegue en **AWS EC2 (Ubuntu)**.


## Arquitectura general

El proyecto contiene un microservicio de vehículos y una base de datos MySQL. Ambos se ejecutan como contenedores independientes y se comunican a través de una red interna de Docker Compose.

```text
Push a main
    |
    v
GitHub Actions
    |-- Compilación y pruebas unitarias (Maven + JUnit + Mockito)
    |-- Cobertura de código (JaCoCo)
    |-- Análisis SonarQube Cloud
    |-- Construcción de imagen Docker
    |-- Publicación en GitHub Container Registry (GHCR)
    `-- Conexión SSH y despliegue en AWS EC2
                            |
                            v
                       Docker Compose
                         |         |
                    MS Vehículos  MySQL 8
                      :9004       red interna
```

### Componentes principales

- `ms-vehiculos/`: código fuente, pruebas, Maven Wrapper, `pom.xml` y `Dockerfile` del microservicio.
- `compose.yaml`: definición de los servicios `ms-vehiculos` y `mysql`, red y volumen persistente.
- `.github/workflows/deploy.yml`: pipeline de CI/CD.
- **SonarQube Cloud**: análisis de calidad y seguridad del código.
- **GHCR**: registro de la imagen Docker versionada con el identificador del commit.
- **AWS EC2**: servidor Ubuntu donde Docker Compose mantiene los contenedores ejecutándose.

## Prerrequisitos

Para ejecutar el proyecto localmente con Docker Compose:

- Git.
- Docker Engine o Docker Desktop con Docker Compose v2.
- Puerto **9004** disponible.

Para ejecutar las pruebas o compilar sin Docker:

- JDK **21**.
- Maven, o el **Maven Wrapper** incluido en `ms-vehiculos/`.

Para desplegar en AWS:

- Instancia **EC2 Ubuntu** con Docker y Docker Compose instalados.
- Acceso SSH a la instancia.
- Reglas de Security Group que permitan el acceso necesario.
- Secretos configurados en GitHub Actions.
- Acceso desde EC2 a la imagen publicada en GHCR.

**Nota:** EC2 no necesita Java ni Maven instalados para ejecutar la imagen Docker del microservicio.

## Secretos

El proyecto utiliza variables de entorno para no almacenar credenciales en el código.

| Variable | Uso |
|---|---|
| `MYSQL_ROOT_PASSWORD` | Contraseña del usuario administrador de MySQL. |
| `DB_PASSWORD` | Contraseña utilizada para la conexión del microservicio con MySQL. |
| `EC2_HOST` | Dirección IP pública de la instancia AWS EC2 donde se realiza el despliegue. |
| `EC2_USER` | Usuario SSH de la instancia EC2 (`ubuntu` en este caso). |
| `EC2_SSH_KEY` | Clave privada SSH utilizada por GitHub Actions para conectarse a EC2 |
| `SONAR_ORGANIZATION` | Organization de SonarQube. |
| `SONAR_PROJECT_KEY` | Project key de SonarQube. |
| `SONAR_TOKEN` | Token de autenticación utilizado para ejecutar el análisis de SonarQube Cloud. |



## Instalación y ejecución local

Clonar el repositorio:

```bash
git clone https://github.com/belruz/devops_2.git
```

Crear en la raíz del proyecto un archivo `.env` con contraseñas de desarrollo (no reales):

```dotenv
MYSQL_ROOT_PASSWORD=CAMBIAR_CLAVE_ROOT
DB_PASSWORD=CAMBIAR_CLAVE_APLICACION
```

Construir e iniciar los servicios:

```bash
docker compose up -d --build
```

Comprobar el estado:

```bash
docker compose ps
```

Consultar los registros:

```bash
docker compose logs -f ms-vehiculos
```

Detener los servicios sin eliminar los datos persistentes:

```bash
docker compose down
```

### Compilación y pruebas unitarias

Desde el directorio `ms-vehiculos/`:

**Windows (PowerShell):**

```powershell
.\mvnw.cmd clean verify
```

**Linux/macOS:**

```bash
chmod +x mvnw
./mvnw clean verify
```

El proyecto incluye **10 pruebas unitarias** de `VehiculoServiceTest`, ejecutadas con **JUnit y Mockito**. **JaCoCo** genera un reporte de cobertura en `ms-vehiculos/target/site/jacoco/`.

## Base de datos

- Motor: **MySQL 8.0**.
- Base de datos: `vehiculos`.
- Usuario de aplicación: `vehiculos_user`.
- Nombre de servicio Docker Compose: `mysql`.
- Conexión interna del microservicio: `jdbc:mysql://mysql:3306/vehiculos`.
- Persistencia: volumen Docker `mysql_data`.

MySQL **no publica el puerto 3306 hacia internet**. El acceso a la base de datos se realiza desde la red interna de Docker Compose.

Para ingresar a MySQL dentro del contenedor (desde el directorio que contiene el Compose y su `.env`):

```bash
docker compose exec mysql mysql -u vehiculos_user -p vehiculos
```

MySQL solicitará la contraseña de aplicación. Luego se pueden ejecutar consultas como:

```sql
SHOW TABLES;
SELECT * FROM vehiculos;
```

## Docker y Docker Compose

El `Dockerfile` de `ms-vehiculos` construye la aplicación con Java 21 y genera una imagen para ejecutar el JAR de Spring Boot. `compose.yaml` define dos servicios:

1. `mysql`, con volumen persistente y comprobación de salud (*healthcheck*).
2. `ms-vehiculos`, que espera a que MySQL esté saludable antes de iniciarse.

Para uso local, `compose.yaml` utiliza la imagen `ms-vehiculos:1.0` si no se define `VEHICULOS_IMAGE`. En AWS, el pipeline indica la imagen versionada de GHCR mediante esa variable.

## CI/CD con GitHub Actions

El workflow está ubicado en `.github/workflows/deploy.yml`. Se ejecuta al hacer **push a `main`**, ante un Pull Request hacia `main` y también de forma manual (`workflow_dispatch`). Los Pull Requests no publican imágenes ni despliegan en EC2.

### Job 1: Compilación y pruebas unitarias

- Configura JDK 21.
- Ejecuta `mvn -B clean verify`.
- Ejecuta las pruebas existentes y genera cobertura JaCoCo.
- Publica los reportes como artefactos de GitHub Actions.

### Job 2: SonarQube Cloud

- Analiza el código Java y los resultados de JaCoCo.
- Comprueba el **Quality Gate** con `sonar.qualitygate.wait=true`.
- Si el análisis incumple el Quality Gate, el pipeline no continúa con la construcción y el despliegue.

**Proyecto SonarQube:** https://sonarcloud.io/dashboard?id=belruz_devops_2&branch=main

### Job 3: Construcción y publicación de Docker

- Construye la imagen desde `ms-vehiculos/Dockerfile`.
- La etiqueta con el **SHA del commit** permite identificar la versión construida.
- Publica la imagen en **GitHub Container Registry (GHCR)** mediante el `GITHUB_TOKEN` del workflow.


### Job 4: Despliegue automático en AWS EC2

- Establece una conexión **SSH** con la instancia utilizando secretos de GitHub.
- Copia el archivo `compose.yaml` a `/home/ubuntu/devops_2`.
- Escribe las variables necesarias en un archivo `.env` protegido.
- Ejecuta `docker compose pull` y `docker compose up -d --no-build`.
- Comprueba que la API responde correctamente en `http://localhost:9004/v3/api-docs`.
- El job falla si la API no responde dentro del tiempo previsto.

El despliegue está condicionado al éxito de las etapas anteriores y se realiza sobre la rama `main`.

## Puesta en marcha y administración en AWS

En la instancia EC2 Ubuntu, los archivos de despliegue se mantienen en:

```text
/home/ubuntu/devops_2/
├── compose.yaml
└── .env
```

Para comprobar los contenedores:

```bash
cd /home/ubuntu/devops_2
docker compose ps
```

Para revisar los logs del microservicio:

```bash
docker compose logs --tail=100 ms-vehiculos
```

## Proyecto en funcionamiento

El microservicio se publica en el **puerto 9004** de EC2.

**Swagger UI:**

```text
http://IP_PUBLICA_EC2:9004/swagger-ui/index.html
```

**Documentación OpenAPI:**

```text
http://IP_PUBLICA_EC2:9004/v3/api-docs
```

**Endpoint de vehículos:**

```text
GET http://IP_PUBLICA_EC2:9004/api/v1/vehiculos
```

Reemplazar `IP_PUBLICA_EC2` por la IP pública vigente de la instancia. El acceso externo depende de las reglas del Security Group. Para esta evaluación se habilitó TCP 9004 restringido a la IP autorizada.


## Pruebas y validaciones realizadas

- Compilación Maven exitosa con Java 21.
- **10 pruebas unitarias aprobadas**, sin fallos.
- Generación e importación de cobertura con JaCoCo.
- **SonarQube Quality Gate: Passed**.
- Cobertura global reportada por SonarQube: **82,3 %**.
- **0 problemas** de seguridad, fiabilidad y mantenibilidad reportados por SonarQube; **0,0 %** de duplicación.
- Construcción y publicación de la imagen Docker en GHCR.
- Ejecución exitosa de los cuatro jobs del pipeline.
- Contenedores `ms-vehiculos` y `mysql` iniciados en EC2.
- Acceso externo a Swagger y respuesta **HTTP 200** del endpoint `GET /api/v1/vehiculos` desde navegador.


## Seguridad y consideraciones

- Credenciales almacenadas como **GitHub Secrets** y no en el repositorio.
- Archivo `.env` en EC2 con permisos restringidos (`chmod 600`).
- MySQL sin publicación del puerto 3306 al exterior.
- Acceso SSH a EC2 mediante clave privada, no contraseña.
- Acceso a la API controlado mediante Security Group.
- **SonarQube Quality Gate** obligatorio antes del despliegue.
- Imágenes identificadas por commit para favorecer la trazabilidad.
