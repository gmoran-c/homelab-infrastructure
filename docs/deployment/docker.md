# Despliegue con Docker

> Navegación: [índice de documentación](../README.md) · [README del proyecto](../../README.md)

Procedimiento y archivos del despliegue de nginx, la aplicación y PostgreSQL. El contenido describe la configuración documentada; comprueba los archivos reales antes de aplicarlo.

## 2. Ubicación y estructura del proyecto

Todo vive en `/opt/server/` (convención estándar en Linux para servicios que no vienen del gestor de paquetes):

```bash
sudo mkdir -p /opt/server
sudo chown $USER:$USER /opt/server
cd /opt/server
```

> **Importante**: `docker compose` siempre se ejecuta desde esta carpeta — las rutas del `docker-compose.yml` son relativas a donde se invoca el comando.

```
/opt/server/
├── docker-compose.yml
├── nginx/
│   ├── nginx.conf
│   ├── conf.d/
│   │   └── default.conf
│   └── certs/
│       ├── selfsigned.crt
│       └── selfsigned.key
├── web/
│   ├── index.html
│   ├── css/
│   ├── js/
│   └── 50x.html
└── app/
    ├── Dockerfile
    ├── requirements.txt
    └── main.py
```

WireGuard vive fuera de esta carpeta, en la ruta estándar del sistema (`/etc/wireguard/`, ver [WireGuard](../network/wireguard.md)).

---

## 3. Docker — servicios y archivos

### 3.1 `docker-compose.yml`

```yaml
services:
  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./web:/usr/share/nginx/html:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    networks:
      - net_dmz
    depends_on:
      - app

  app:
    build: ./app
    restart: unless-stopped
    expose:
      - "8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
    networks:
      - net_dmz
      - net_backend
    depends_on:
      - db

  db:
    image: postgres:16
    restart: unless-stopped
    environment:
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:?Define POSTGRES_PASSWORD en .env}
      - POSTGRES_DB=mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - net_backend

networks:
  net_dmz:
    driver: bridge
  net_backend:
    driver: bridge
    internal: true

volumes:
  pgdata:
```

> `restart: unless-stopped` en los tres servicios: se recuperan solos tras un fallo o un reinicio del servidor (requiere `sudo systemctl enable docker`). `POSTGRES_PASSWORD` se carga desde un `.env` local no versionado en git; nunca se debe sustituir por una contraseña escrita en este archivo. El servicio `certbot` aún no está incluido; se añade cuando haya dominio real (ver [pendientes](../operations/roadmap.md)).

### 3.2 `nginx/nginx.conf`

Archivo estructural (rate limiting, keepalive, logs a stdout/stderr, gzip) — no cambia salvo ajustes de fondo:

```nginx
worker_processes  auto;
error_log  /dev/stderr warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                       '$status $body_bytes_sent "$http_referer" '
                       '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /dev/stdout  main;

    sendfile        on;
    tcp_nopush      on;
    keepalive_timeout  65;
    keepalive_requests 100;

    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=api:10m rate=5r/s;
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

### 3.3 `nginx/conf.d/default.conf`

HTTPS activo (certificado autofirmado), redirect HTTP→HTTPS, rate limiting, proxy dinámico hacia `app`:

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name localhost;   # cuando haya dominio real: cambiar por tudominio.com

    ssl_certificate     /etc/nginx/certs/selfsigned.crt;
    ssl_certificate_key /etc/nginx/certs/selfsigned.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    limit_conn addr 20;

    location / {
        root /usr/share/nginx/html;
        index index.html;
        limit_req zone=general burst=20 nodelay;
        try_files $uri $uri/ =404;
    }

    location /api/ {
        limit_req zone=api burst=10 nodelay;

        resolver 127.0.0.11 valid=10s;
        set $upstream_app app:8000;
        proxy_pass http://$upstream_app/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}

server {
    listen 80;
    server_name localhost;
    return 301 https://$host$request_uri;
}
```

> Al migrar a Let's Encrypt, solo cambian 3 líneas: `server_name`, `ssl_certificate`, `ssl_certificate_key`.

### 3.4 Certificado autofirmado (temporal)

```bash
mkdir -p /opt/server/nginx/certs
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout /opt/server/nginx/certs/selfsigned.key \
  -out /opt/server/nginx/certs/selfsigned.crt \
  -subj "/CN=localhost"
```

Los navegadores lo marcan como no confiable — es esperado. Se sustituye por Let's Encrypt cuando haya dominio + puerto 80 alcanzable desde internet.

### 3.5 `web/` — frontend

`index.html`, `css/`, `js/`, y `50x.html` (página de error para cuando el rate limiting o el backend fallan). Servido directamente por nginx desde el volumen montado — cualquier cambio se refleja sin reiniciar contenedores, solo recargando el navegador.

### 3.6 `app/` — backend (ejemplo)

Ejemplo mínimo Python/FastAPI usado para probar el flujo completo; sustituir por el stack real si es distinto.

`app/Dockerfile`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`app/requirements.txt`:
```
fastapi
uvicorn[standard]
psycopg2-binary
```

`app/main.py`:
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"status": "ok"}
```

---
