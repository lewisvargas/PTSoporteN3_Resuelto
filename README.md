# Prueba Técnica: Ingeniero de Soporte N3 - Diagnóstico de Microservicios

## Escenario
Se ha reportado una caída crítica en el portal de clientes. El equipo de Nivel 2 escaló el caso indicando que, tras el último despliegue, el servicio no responde correctamente. Tu misión es estabilizar el entorno, identificar las fallas y asegurar la comunicación entre los componentes.

## Arquitectura del Entorno
El sistema consta de tres capas corriendo en contenedores Docker:
1. **nginx-proxy**: Balanceador de carga y punto de entrada (Puerto 8080).
2. **api-service**: Lógica de negocio (Node.js).
3. **database**: Motor de base de datos (PostgreSQL).

---

## Notas de resolución del incidente (post-mortem)

Tras revisar el `docker-compose.yml` y el `nginx.conf` se identificaron **5 fallas** que en conjunto impedían que el portal respondiera correctamente. A continuación el detalle de cada una y la corrección aplicada.

### 1. `version: '1'` inválida en `docker-compose.yml`
**Síntoma:** el archivo ni siquiera puede ser interpretado por Docker Compose actual; `docker-compose up -d` falla antes de crear cualquier contenedor.
**Causa raíz:** `'1'` es un formato de Compose obsoleto/no soportado.
**Corrección:** se actualizó a `version: '3.9'`, compatible con Docker Compose v2.

### 2. `DB_HOST: 'localhost'` en `api-service`
**Síntoma:** la API responde siempre `503 - Bd No contectada` porque nunca logra conectarse a Postgres.
**Causa raíz:** en Docker, cada servicio corre en su propio contenedor/red. `localhost` dentro del contenedor `api-service` apunta al propio contenedor de la API, no al de la base de datos.
**Corrección:** se cambió `DB_HOST` a `database`, que es el nombre del servicio definido en el `docker-compose.yml` y que Docker resuelve automáticamente por DNS interno dentro de la red compartida.

### 3. Límite de memoria de 5 MB en `api-service`
**Síntoma:** el contenedor de la API se reinicia en bucle o ni siquiera arranca (OOMKilled).
**Causa raíz:** `deploy.resources.limits.memory: 5M` asigna solo 5 MB, insuficiente para inicializar el runtime de Node.js (V8 requiere bastante más solo para arrancar). Adicionalmente, `deploy.resources` únicamente se aplica en modo Swarm, no con `docker compose up` estándar, por lo que el límite ni siquiera se estaba aplicando de la forma esperada.
**Corrección:** se reemplazó por `mem_limit: 128m`, que sí se respeta en Compose estándar y es suficiente para el proceso Node.

### 4. Puerto incorrecto en el `upstream` de Nginx
**Síntoma:** Nginx devuelve `502 Bad Gateway` al intentar acceder al portal en el puerto 8080.
**Causa raíz:** en `nginx.conf` el `upstream api_servers` apuntaba a `api-service:8080`, pero el servidor Node en realidad escucha en el puerto definido por la variable `target_output` = `4500`. Nginx intentaba conectarse a un puerto donde no había nada escuchando.
**Corrección:** se actualizó el `upstream` a `api-service:4500`.

### 5. Falta de dependencia entre `api-service` y `database` (Plus)
**Síntoma:** en el primer arranque, la API puede intentar conectarse a Postgres antes de que el contenedor de la base de datos exista o esté listo para aceptar conexiones, devolviendo `503` de forma intermitente.
**Causa raíz:** no existía ningún `depends_on` entre `api-service` y `database`, y aunque lo hubiera, un `depends_on` simple solo garantiza orden de arranque del contenedor, no que el servicio interno (Postgres) ya esté listo para aceptar conexiones.
**Corrección (Plus solicitado):**
- Se agregó un `healthcheck` al servicio `database` usando `pg_isready`.
- Se agregó `depends_on: database: condition: service_healthy` en `api-service`, de modo que la API solo arranca cuando Postgres ya está realmente disponible.
- Adicionalmente se agregó un `healthcheck` al propio `api-service` y se condicionó el arranque de `nginx-proxy` a que la API esté `healthy`, evitando también que Nginx reciba tráfico antes de tiempo.

---

## Resultado esperado
Con estas correcciones, al ejecutar:
```bash
docker-compose up -d
```
Los tres servicios levantan en el orden correcto (`database` → `api-service` → `nginx-proxy`), y al consultar:
```bash
curl http://localhost:8080
```
se debe obtener:
```html
<h1>Ingeniero de Soporte</h1><label>Has resuelto el incidente</label>
```
con código de respuesta `200 OK`.

---

## Instrucciones para el Candidato

### 1. Preparación
Asegúrate de tener instalado **Docker** y **Docker Compose** y bajar los archivos necesarios de este repositorio.

### 2. Ejecución del Incidente
Inicia el entorno ejecutando el siguiente comando en tu terminal:
```bash
docker-compose up -d
```

### 3. Entrega
1. Debe hacer un Pull request al repositorio para que el propietario del repo pueda ver los cambios.
2. Si desea puede hacer un fork del proyecto.
3. También es válido crear un repo completamente nuevo y subir allí los archivos corregidos. Debe enviar el Link de su repo a la persona que lo ha estado acompañando en el proceso de selección
4. En el archivo readme.md debe subir las notas con la explicación o justificación de los cambios efectuados para que el proyecto pueda correr. Es decir documente cómo ha resuelto el incidente y resalte las partes donde tuvo que hacer las modificaciones.

#### Plus
Agrega un healthcheck para que la API no se conecte a la base de datos sin que este servicio de Postgres esté listo.
