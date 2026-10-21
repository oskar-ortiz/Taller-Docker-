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
docker build --build-arg STUDENT_ID=905523 -t eco-air-backend:1.0.0 .
```

## Ejecutar el contenedor

```bash
docker run --rm \
  --name eco-air-backend \
  -m 128m \
  -p 8080:8080 \
  eco-air-backend:1.0.0
```

También puede inyectarse el identificador durante la ejecución:

```bash
docker run --rm \
  --name eco-air-backend \
  -p 8080:8080 \
  -e OPERATOR_ID=905523 \
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
- Usuario runtime no root con UID `10001`.

## Auditoría y calibración

La imagen fue probada con límites cgroup de 48, 112, 128, 144, 160, 192, 256, 384 y 512 MiB. En esta máquina, `128m` fue el límite mínimo probado que inició en menos de 15 segundos y permaneció saludable; con `48m` el contenedor terminó por OOM (código 137), y con `112m` el inicio tomó más de 15 segundos. Los resultados exactos, timestamps e ID de imagen están en `FORENSIC_REPORT.md` y `audit_session.log`.

## Entregables

El repositorio incluye el código fuente, la configuración Maven, los archivos Docker, el informe forense, el registro de auditoría y los archivos de entrega del taller.

### Informe completo

[Abrir o descargar el informe del Taller Docker EcoAir (PDF)](./taller_docker_semana13.pdf)
