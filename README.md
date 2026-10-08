# karting-gateway

Gateway nginx que ocupa los puertos públicos **8080** y **8090** y reenvía el tráfico a los frontales de los dos proyectos. Si el frontal está apagado, en lugar de "conexión rechazada" muestra una página de aviso con los datos de contacto del administrador.

Ruta: `~/PROYECTOS/karting-gateway`

## Arquitectura

```
                    Internet / LAN
                          │
            ┌─────────────┴─────────────┐
            │      gateway (nginx)      │
            │   :8080            :8090  │
            └──────┬──────────────┬─────┘
                   │  red proxy-net│
        ┌──────────▼───┐    ┌──────▼───────────┐
        │ karts-       │    │ karting-live-    │
        │ frontend-1   │    │ web-1            │
        │ (interno :80)│    │ (interno :8090)  │
        └──────┬───────┘    └──────┬───────────┘
               │ red default       │ red default
        ┌──────▼───────┐    ┌──────▼───────────┐
        │ karts-api-1  │    │ karting-live-    │
        │ (:8000)      │    │ collector-1      │
        └──────────────┘    └──────────────────┘
```

| Puerto público | Destino                     | Puerto interno |
|----------------|-----------------------------|----------------|
| 8080           | `karts-frontend-1`          | 80             |
| 8090           | `karting-live-web-1`        | 8090           |

Los backends (`karts-api-1` y `karting-live-collector-1`) no pasan por el gateway. Solo los frontales se conectan a `proxy-net`.

Las webs son [https://karting.mianca.com.es/](https://karting.mianca.com.es/) y [https://karting-live.mianca.com.es/](https://karting-live.mianca.com.es/)

## Estructura

```
karting-gateway/
├── docker-compose.yml   # contenedor nginx, puertos 8080/8090, volúmenes y red
├── nginx.conf           # un bloque server por puerto, con proxy y fallback
├── apagado.html         # página que se muestra cuando el destino no responde
└── README.md
```

## Funcionamiento

### Flujo normal (frontal encendido)

1. El navegador llega al 8080 o al 8090 del servidor.
2. nginx resuelve el nombre del contenedor destino con el DNS interno de Docker (`127.0.0.11`).
3. Reenvía la petición con las cabeceras `Host`, `X-Real-IP` y `X-Forwarded-For`.
4. Devuelve la respuesta del frontal sin modificarla.

### Flujo con el frontal apagado

1. El contenedor destino está parado, por lo que su nombre ya no se resuelve en `proxy-net`, o la conexión falla en 3 s (`proxy_connect_timeout 3s`).
2. nginx recibe un error 502, 503 o 504.
3. `proxy_intercept_errors on` intercepta ese error y `error_page 502 503 504 =503 /apagado.html` lo sustituye por la página de aviso.
4. El navegador recibe **HTTP 503** con `apagado.html`, `Cache-Control: no-store` y `Retry-After: 300`.

### Detalles de la configuración

- **`resolver 127.0.0.11 valid=5s`** junto con `set $upstream ...; proxy_pass $upstream;`: obliga a nginx a resolver el nombre en cada petición (cache de 5 s). Sin esto, nginx falla al arrancar si algún contenedor destino está parado.
- **`location = /apagado.html` con `internal`**: la página solo se sirve como respuesta a un error, no se puede pedir directamente desde fuera.
- **`no-store`**: evita que el navegador guarde el aviso y lo siga mostrando cuando el servicio vuelva.
- **Error del propio frontal**: si el frontal responde pero su API está caída y devuelve 502/503/504, también se muestra el aviso. Un 500 o un 404 del frontal pasan tal cual.

## Requisitos previos

1. Red externa compartida (se crea una sola vez):

   ```bash
   docker network create proxy-net
   ```

2. En el compose de **karts**: el servicio frontend sin `ports:` y con `networks: [default, proxy-net]`.
3. En el compose de **karting-live**: sin `network_mode: host`, `web` sin `ports:` y con `networks: [default, proxy-net]`. `web.port` en `race.cfg` debe ser 8090.
4. Los puertos 8080 y 8090 del host deben estar libres antes de arrancar el gateway.

## Uso

```bash
cd ~/PROYECTOS/karting-gateway

docker compose up -d          # arrancar
docker compose restart        # tras editar nginx.conf o apagado.html
docker compose down           # parar
docker logs -f gateway        # ver peticiones y errores
```

Para validar la configuración antes de aplicarla:

```bash
docker exec gateway nginx -t
docker exec gateway nginx -s reload     # recarga sin cortar conexiones
```

## Cómo analizar que funciona

### 1. Estado general

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Debe verse `gateway` con `0.0.0.0:8080->8080` y `0.0.0.0:8090->8090`, y los frontales sin puertos publicados.

### 2. Red compartida

```bash
docker network inspect proxy-net --format '{{range .Containers}}{{.Name}} {{end}}'
```

Deben aparecer `gateway`, `karts-frontend-1` y `karting-live-web-1`. Si falta alguno, el gateway no podrá alcanzarlo.

### 3. Prueba con todo encendido

```bash
curl -I http://localhost:8080     # esperado: 200 (o la respuesta del frontal)
curl -I http://localhost:8090     # esperado: 200
```

### 4. Prueba con un servicio apagado

```bash
docker stop karts-frontend-1
curl -i http://localhost:8080     # esperado: HTTP/1.1 503 + página de aviso
docker start karts-frontend-1
curl -I http://localhost:8080     # vuelve a responder normal
```

Lo mismo para el 8090 con `karting-live-web-1`.

### 5. Resolución de nombres desde el gateway

```bash
docker exec gateway nslookup karts-frontend-1 127.0.0.11
docker exec gateway nslookup karting-live-web-1 127.0.0.11
```

Si el contenedor está encendido, devuelve su IP; si está parado, no resuelve (y ese es el caso que dispara el aviso).

## Problemas frecuentes

| Síntoma | Causa probable | Solución |
|---------|----------------|----------|
| El gateway no arranca: `address already in use` | El 8080 u 8090 sigue ocupado por el frontal antiguo | `ss -tlnp \| grep -E '8080\|8090'`, y recrear los frontales sin `ports:` / sin `network_mode: host` |
| Siempre sale el aviso aunque el servicio esté encendido | El contenedor no está en `proxy-net`, o el nombre o puerto interno del upstream no coinciden | Revisar `docker network inspect proxy-net` y los `set $upstream` de `nginx.conf` |
| Error `network proxy-net declared as external, but could not be found` | Falta crear la red | `docker network create proxy-net` |
| El aviso sale con retraso | `proxy_connect_timeout` de 3 s si el contenedor existe pero no responde | Reducir el valor en `nginx.conf` |
| `web.py` responde en el contenedor pero no desde fuera | Escucha en `127.0.0.1` en vez de `0.0.0.0` | Cambiar el bind en `web.py` o `race.cfg` |
| La web carga pero no actualiza datos en tiempo real | Falta soporte de WebSocket o SSE en el proxy | Añadir al bloque `location /` las líneas de abajo |

Si la web usa WebSocket, añade a cada `location /`:

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_read_timeout 3600s;
```

## Añadir otro servicio

1. Conecta el contenedor a `proxy-net` y quítale el `ports:` público.
2. Añade un bloque `server` en `nginx.conf` con un nuevo `listen` y su `set $upstream`.
3. Publica el nuevo puerto en `docker-compose.yml` del gateway.
4. `docker compose up -d` para recrear el gateway con el puerto nuevo.

## Contacto mostrado en el aviso

El texto de `apagado.html` apunta a:

- Web: https://miguelandrescaballero.es
- Telegram: https://t.me/TwiBlog_bot
