# FORENSIC_REPORT

## Estado de ejecución
Se creó el backend y el Dockerfile solicitado. La ejecución de Docker no pudo realizarse en este entorno porque el comando `docker` no está instalado (`docker version` y `docker info` devolvieron `bash: docker: command not found`, código 127). Por integridad académica, no se inventan hashes, UUIDs, estados OOM, logs de Spring ni tablas de `docker stats`.

## Identidad
| Dato | Resultado |
|---|---|
| STUDENT_ID / OPERATOR_ID | 905523 (configurado en Dockerfile; no verificado en contenedor por bloqueo de Docker) |
| Hash imagen | No disponible: la imagen no pudo construirse |
| docker stats final | No disponible: no existe contenedor calibrado |

## Verificación Docker
Comando: `docker version && docker info`  
Timestamp: `2026-10-21T01:02:00Z`  
Salida: `bash: line 1: docker: command not found`  
Explicación: el daemon/CLI Docker no está disponible en el entorno actual.

## Reto 2
No ejecutable sin Docker. Los comandos `docker history`, `docker run`, `docker top`, `docker exec`, `docker id`, `which mvn` y `find` no fueron simulados.

## Reto 3
No ejecutable sin Docker. El resultado OOM de 48m y la calibración de 256m/384m/512m no fueron inventados. El umbral mínimo queda pendiente de ejecutar en Docker Desktop/WSL2.

## Archivos preparados
- `pom.xml`
- `src/main/java/com/ecoair/EcoAirApplication.java`
- `src/main/java/com/ecoair/alerta/AlertaAmbiental.java`
- `src/main/java/com/ecoair/alerta/AlertaAmbientalRepository.java`
- `src/main/java/com/ecoair/alerta/AlertaAmbientalController.java`
- `src/main/resources/application.properties`
- `Dockerfile`
- `.dockerignore`
- `.gitignore`
- `audit_session.log`

## Revisión pendiente
Ejecutar las fases 0, 2 y 3 en una máquina con Docker Desktop + WSL2, actualizar este reporte únicamente con salidas reales y generar el ZIP/PDF final con hash y stats obtenidos del daemon.
