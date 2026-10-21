# Informe forense — Taller Docker EcoAir

**Estudiante:** 905523

**Fecha de ejecución:** 2026-10-20 (hora local, UTC-05:00)

**Motor:** Docker Desktop 4.91.0, Linux/amd64, cgroup v2
**Daemon ID:** `bae1e2b7-bdf6-48a8-8530-a671cb4b6c54`

La salida completa, en orden cronológico, de los comandos ejecutados y sus timestamps se conserva en [`audit_session.log`](./audit_session.log). Este informe resume los resultados medidos directamente en el daemon local; no se sustituyeron salidas.

## Identidad final

| Campo | Evidencia |
|---|---|
| Imagen | `eco-air-backend:1.0.0` |
| ID de imagen (`docker inspect --format='{{.Id}}'`) | `sha256:9910633fcdb9e34c5f0d3333bdb1f5d1241fa453b461f5d412839f7d3c22aae4` |
| Digest de configuración reportado por BuildKit | `sha256:bf80f39a1bcc8c664bddbcdf592133e4f07368f5db0a6021999e80bb647115cd` |
| `OPERATOR_ID` en el contenedor | `905523` |
| Contenedor final | `eco_air_active`, ID `44ded8cbb97243a3920036952f390b7fdc9b95a56028bdf21c6072c85886351b` |
| Estado final | `running`, límite `134217728` bytes (128 MiB), puerto `8080:8080` |
| Health endpoint | `{"status":"UP"}` |

La imagen se reconstruyó después de normalizar Dockerfile y `.dockerignore` a LF sin BOM. BuildKit conservó el digest de configuración y las capas Maven/JAR en caché, pero emitió un digest de manifiesto nuevo; por ello se recreó el contenedor final desde la etiqueta actual y se verificó de nuevo su hash, arranque y health.

### `docker stats --no-stream` final

Timestamp local del comando: `2026-10-20T21:02:40.0434000-05:00`.

```text
CONTAINER ID   NAME             CPU %     MEM USAGE / LIMIT   MEM %     NET I/O         BLOCK I/O       PIDS
44ded8cbb972   eco_air_active   0.63%     122MiB / 128MiB     95.33%    2.99kB / 1.77kB  144MB / 187MB   31
```

La lectura de memoria es alta pero se mantuvo en el rango observado durante las pruebas a 128 MiB. En la verificación final, después de varios minutos, el contenedor seguía `running`, el endpoint devolvió `UP` y el inicio final apareció en `9.824` segundos.

## Reto 2 — capas y usuario

### Historial de imagen

Timestamp: `2026-10-20T20:47:43.9084671-05:00`

Comando: `docker history --human --format "table {{.Size}}\t{{.CreatedBy}}" eco-air-backend:1.0.0`
Código de salida: `0`.

```text
SIZE      CREATED BY
0B        ENTRYPOINT ["/bin/sh" "-c" "exec java $JAVA_…
0B        USER sensor_user
0B        EXPOSE [8080/tcp]
0B        ENV OPERATOR_ID=905523
0B        ENV JAVA_TOOL_OPTIONS=-XX:+UseContainerSuppo…
50.1MB    RUN |1 STUDENT_ID=905523 /bin/sh -c chown -R…
50.1MB    COPY /build/target/eco-air-backend-1.0.0.jar…
8.19kB    WORKDIR /app
41kB      RUN |1 STUDENT_ID=905523 /bin/sh -c addgroup…
0B        ARG STUDENT_ID=905523
0B        ENTRYPOINT ["/__cacert_entrypoint.sh"]
12.3kB    COPY --chmod=755 entrypoint.sh /__cacert_ent…
12.3kB    RUN /bin/sh -c set -eux;     echo "Verifying…
141MB     RUN /bin/sh -c set -eux;     ARCH="$(apk --…
0B        ENV JAVA_VERSION=jdk-17.0.20.1+1
44MB      RUN /bin/sh -c set -eux;     apk add --no-ca…
0B        ENV LANG=en_US.UTF-8 LANGUAGE=en_US:en LC_AL…
0B        ENV PATH=/opt/java/openjdk/bin:/usr/local/sb…
0B        ENV JAVA_HOME=/opt/java/openjdk
0B        CMD ["/bin/sh"]
9.08MB    ADD alpine-minirootfs-3.24.2-x86_64.tar.gz /…
```

### Usuario, permisos y contenido de runtime

| Timestamp local | Comando | Salida exacta / código | Interpretación |
|---|---|---|---|
| `20:47:44.2769842-05:00` | `docker run -d --name eco_air_active -p 8080:8080 eco-air-backend:1.0.0` | `0e5f6b44a128664230860a99255ec29b8057b4de249d3b8ab0560b71b9595061` (0) | Primera ejecución de comprobación; luego se recreó con el límite calibrado. |
| `20:47:58.0195000-05:00` | `docker top eco_air_active -o pid,user,comm` | `PID USER COMMAND` / `875 10001 java` (0) | Docker muestra UID numérico. |
| `20:47:58.1960803-05:00` | `docker exec eco_air_active id` | `uid=10001(sensor_user) gid=101(sensorgroup) groups=101(sensorgroup)` (0) | Confirma que UID 10001 corresponde a `sensor_user`. |
| `20:47:58.3970159-05:00` | `docker exec eco_air_active touch /etc/crontab_fake` | `touch: cannot touch '/etc/crontab_fake': Permission denied` (1) | El proceso no puede escribir en el área protegida del sistema. |
| `20:47:58.5526451-05:00` | `docker exec eco_air_active which mvn` | Sin salida (1) | Maven no está instalado en la imagen runtime. |
| `20:47:58.6969342-05:00` | `docker exec eco_air_active sh -c 'find / -name "*.java" 2>/dev/null'` | Sin salida (1) | No se encontraron fuentes `.java`; el `find` devolvió 1 al encontrar rutas inaccesibles con stderr suprimido. |
| `20:47:59.5337729-05:00` | `docker logs --timestamps eco_air_active` | Incluye `Started EcoAirApplication in 6.685 seconds (process running for 7.579)` | El backend inició bajo `sensor_user`. |
| `20:47:59.6240716-05:00` | `docker inspect --format='{{.State.Status}} {{.State.ExitCode}} {{.State.OOMKilled}}' eco_air_active` | `running 0 false` (0) | La primera ejecución permaneció activa. |

## Reto 3 — OOM y calibración cgroups

### Límite de 48 MiB

| Timestamp local | Comando | Salida exacta / resultado |
|---|---|---|
| `20:48:15.8734397-05:00` | `docker run -d --name eco_air_crash -m 48m eco-air-backend:1.0.0` | ID `24aeb41f581e3a5f72bdbee483835d134598bb702b2011a50c4ed12bbd8b2eff`, código 0. |
| `20:48:26.4326212-05:00` | `docker inspect eco_air_crash --format='Status: {{.State.Status}} \| ExitCode: {{.State.ExitCode}} \| OOMKilled: {{.State.OOMKilled}}'` | `Status: running \| ExitCode: 0 \| OOMKilled: false` a los 10 s. |
| `20:48:26.6629164-05:00` | `docker stats --no-stream eco_air_crash` | `24aeb41f581e eco_air_crash 35.57% 47.91MiB / 48MiB 99.81% 872B / 126B 300MB / 344MB 15`. |
| `20:48:46.2226466-05:00` | Mismo `docker inspect` | `Status: exited \| ExitCode: 137 \| OOMKilled: true` después de esperar 20 s. |
| `20:48:46.4206386-05:00` | Mismo `docker stats --no-stream` | `24aeb41f581e eco_air_crash 0.00% 0B / 0B 0.00% 0B / 0B 0B / 0B 0`. |

Resultado observado: el límite de memoria de 48 MiB causó terminación por OOM (código 137). No se modificó el estado para producir ese resultado.

### Tabla de calibración

Cada intento se recreó bajo el nombre `eco_air_crash`; se esperó al menos 30 s en los límites bajos para detectar una terminación tardía. El tiempo corresponde al valor `Started EcoAirApplication in ... seconds` de los logs de Spring Boot. Memoria es la salida real de `docker stats --no-stream`.

| Límite | Resultado observado | Arranque | Stats observadas |
|---:|---|---:|---|
| 48 MiB | `exited`, código 137, `OOMKilled=true`; a los 10 s seguía iniciando | No llegó a `Started` | `47.91MiB / 48MiB`, 99.81%; tras OOM, `0B / 0B`. |
| 112 MiB | `running`, no OOM en la observación; incumple el criterio de menos de 15 s | 16.508 s | `106.6MiB / 112MiB`, 95.20%. |
| 128 MiB | `running`, cumplió el umbral de arranque; lectura posterior estable y health `UP` | 9.294 s; validación final con la imagen recompuesta: 9.824 s | `121.8MiB / 128MiB`, 95.18%; luego `122.3MiB / 128MiB`, 95.57%. Contenedor final: `122MiB / 128MiB`, 95.33%. |
| 144 MiB | `running` | 7.559 s | `126.6MiB / 144MiB`, 87.92%; luego `126.9MiB / 144MiB`, 88.09%. |
| 160 MiB | `running` | 7.064 s | `143.3MiB / 160MiB`, 89.58%; luego `143.5MiB / 160MiB`, 89.69%. |
| 192 MiB | `running` | 6.132 s | `175.3MiB / 192MiB`, 91.33% en dos lecturas. |
| 256 MiB | `running` | 6.838 s | `178.2MiB / 256MiB`, 69.61%. |
| 384 MiB | `running` | 7.948 s | `178.7MiB / 384MiB`, 46.54%; luego `177.6MiB / 384MiB`, 46.26%. |
| 512 MiB | `running` | 5.941 s | `179.6MiB / 512MiB`, 35.08% en dos lecturas. |

**Umbral mínimo elegido: 128 MiB.** Fue el límite menor probado que produjo `Started EcoAirApplication` en menos de 15 s y mantuvo el proceso activo en las lecturas siguientes. A 112 MiB el proceso seguía `running`, pero el arranque tardó 16.508 s y por eso no cumple el requisito. El contenedor final se dejó corriendo con `-m 128m -p 8080:8080`; su última lectura registró 123.1 MiB de 128 MiB y el endpoint de salud respondió `{"status":"UP"}`.

## Entorno y preservación de otros contenedores

Al inicio `docker version` y `docker info` confirmaron Docker Desktop y el daemon Linux. El primer build falló mientras Docker Desktop se detenía, con HTTP 500 al obtener BuildKit. Se registraron el error y la recuperación del Desktop en el log; el build posterior completó correctamente. Los contenedores existentes `acme-mq-prod-01` y `kernel-demo` no se eliminaron ni modificaron. La limpieza y recreación se limitaron a `eco_air_active` y `eco_air_crash`.
