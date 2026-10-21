# Taller Docker — EcoAir Backend

Proyecto del taller de Docker para empaquetar y ejecutar un backend Spring Boot de EcoAir.

## Tecnologías

- Java 17
- Spring Boot 3.3.5
- Maven
- Docker
- H2 Database
- Spring Data JPA
- Spring Boot Actuator

## Estructura

```text
.
├── Dockerfile
├── .dockerignore
├── pom.xml
├── src/
├── taller-docker-semana13/
├── FORENSIC_REPORT.md
├── audit_session.log
├── taller_docker_semana13.pdf
└── taller_docker_semana13.zip
```

## Ejecutar localmente con Maven

Requiere Java 17 y Maven.

```bash
mvn spring-boot:run
```

La aplicación queda disponible en `http://localhost:8080`.

## Construir la imagen Docker

El argumento `STUDENT_ID` identifica al estudiante y se inyecta como `OPERATOR_ID` dentro del contenedor.

```bash
docker build --build-arg STUDENT_ID=TU_ID -t eco-air-backend:1.0.0 .
```

## Ejecutar el contenedor

```bash
docker run --rm \
  --name eco-air-backend \
  -p 8080:8080 \
  eco-air-backend:1.0.0
```

También puede inyectarse el identificador durante la ejecución:

```bash
docker run --rm \
  --name eco-air-backend \
  -p 8080:8080 \
  -e OPERATOR_ID=TU_ID \
  eco-air-backend:1.0.0
```

## Verificar el servicio

```bash
curl http://localhost:8080/actuator/health
```

La respuesta esperada indica que el servicio está disponible.

## Características del Dockerfile

- Construcción multi-stage para separar compilación y ejecución.
- Imagen final basada en un JRE liviano.
- Usuario no root (`sensor_user`) para ejecutar la aplicación.
- Soporte para límites de memoria del contenedor mediante `JAVA_TOOL_OPTIONS`.
- Puerto `8080` expuesto.
- Configuración del identificador del operador mediante `STUDENT_ID` y `OPERATOR_ID`.

## Entregables

El repositorio incluye el código fuente, la configuración Maven, los archivos Docker, el informe forense, el registro de auditoría y los archivos de entrega del taller.
