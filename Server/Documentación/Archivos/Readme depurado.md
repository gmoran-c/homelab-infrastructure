# Servidor personal — Arch Linux, Docker, nginx, PostgreSQL, WireGuard

> **Nota:** las IPs, la marca del router y otros identificadores de esta documentación se han generalizado (`<TU_IP_PUBLICA>`, `192.168.1.X`, etc.) respecto al entorno real donde se implementó, por motivos de privacidad. La estructura, los comandos y las soluciones son los aplicados en el entorno original.

Servidor doméstico auto-alojado: sitio web + API detrás de nginx, base de datos aislada, acceso remoto de administración vía VPN propia (WireGuard). Documentación completa del proceso, decisiones de diseño, y bitácora de todos los errores reales encontrados al montarlo.

**Estado actual:**
- ✅ Docker (nginx + app + db) segmentado en redes internas
- ✅ HTTPS funcionando (certificado autofirmado — pendiente real de Let's Encrypt)
- ✅ Rate limiting activo y verificado
- ✅ firewalld configurado en el servidor
- ✅ WireGuard funcionando de extremo a extremo, verificado desde fuera de la red local
- ✅ fail2ban activo (sshd + nginx), verificado con logs reales
- ✅ Cockpit y SSH restringidos exclusivamente a la VPN (zona `trusted`), fuera de `public`
- ⏳ Pendiente: dominio + certificado real, DuckDNS (ver [Pendientes](#13-pendientes-generales))

---

## Índice

1. [Arquitectura general](#1-arquitectura-general)
2. [Ubicación y estructura del proyecto](#2-ubicación-y-estructura-del-proyecto)
3. [Docker — servicios y archivos](#3-docker--servicios-y-archivos)
4. [firewalld — servidor](#4-firewalld--servidor)
5. [WireGuard — VPN self-hosteada](#5-wireguard--vpn-self-hosteada)
6. [Router doméstico — NAT y firewall](#6-router-doméstico--nat-y-firewall)
7. [fail2ban](#7-fail2ban)
8. [Cockpit y SSH exclusivos por VPN](#8-cockpit-y-ssh-exclusivos-por-vpn)
9. [Chuleta — activar la VPN en el cliente](#9-chuleta--activar-la-vpn-en-el-cliente)
10. [Bitácora completa de errores resueltos](#10-bitácora-completa-de-errores-resueltos)
11. [Comandos de referencia rápida](#11-comandos-de-referencia-rápida)
12. [Decisiones de diseño](#12-decisiones-de-diseño)
13. [Pendientes generales](#13-pendientes-generales)

---

## 1. Arquitectura general

```
Internet
   │
   ▼
Router doméstico (NAT + firewall propio)
   │
   ├── 80/tcp, 443/tcp  ──────────────────┐
   └── 51820/udp (WireGuard) ─────┐       │
                                   ▼       ▼
                          firewalld (servidor, zona public)
                                   │       │
                         ┌─────────┘       │
                         ▼                 ▼
                    interfaz wg0      nginx (contenedor,
                    (túnel VPN,       único servicio con
                    10.10.10.0/24)    puertos publicados)
                         │                 │
                         │        ┌────────┴────────┐
                         │        ▼                 │
                         │   sirve /web/ (estático)  │
                         │        │                 │
                         └───►  proxy /api/ ──► app (contenedor)
                                                     │
                                                     ▼
                                          db / PostgreSQL
                                    (red interna, sin puerto
                                     publicado, sin salida a internet)
```

**Principios de diseño aplicados:**
- Solo nginx tiene puertos publicados al host (`80`, `443`). Ningún otro contenedor es alcanzable directo desde fuera.
- `app` usa `expose`, no `ports` → solo visible entre contenedores de la misma red docker.
- `db` está en una red `internal: true` → sin salida a internet, inalcanzable desde fuera de esa red.
- firewalld controla qué llega al host; las redes docker controlan qué se hablan los contenedores entre sí.
- Administración (Cockpit, SSH) pensada para ir solo por la VPN (zona `trusted`), no expuesta directo a internet.

---

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

WireGuard vive fuera de esta carpeta, en la ruta estándar del sistema (`/etc/wireguard/`, ver [sección 5](#5-wireguard--vpn-self-hosteada)).

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

> `restart: unless-stopped` en los tres servicios: se recuperan solos tras un fallo o un reinicio del servidor (requiere `sudo systemctl enable docker`). `POSTGRES_PASSWORD` se carga desde un `.env` local no versionado en git; nunca se debe sustituir por una contraseña escrita en este archivo. El servicio `certbot` aún no está incluido; se añade cuando haya dominio real (ver [Pendientes](#13-pendientes-generales)).

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

## 4. firewalld — servidor

```bash
sudo systemctl enable --now firewalld

firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --zone=public --add-port=443/tcp --permanent
firewall-cmd --reload

firewall-cmd --list-all   # verificación
```

Configuración final, incluyendo lo añadido para WireGuard (ver sección 5):

```bash
sudo firewall-cmd --zone=public --add-port=51820/udp --permanent
sudo firewall-cmd --zone=public --add-masquerade --permanent
sudo firewall-cmd --permanent --zone=trusted --add-interface=wg0
sudo firewall-cmd --zone=trusted --add-port=443/tcp --permanent
sudo firewall-cmd --reload
```

---

## 5. WireGuard — VPN self-hosteada

**Estado: funcional.** Túnel establecido y verificado desde fuera de la red local (datos móviles), con acceso confirmado a HTTPS del servidor a través de la VPN.

### 5.1 Arquitectura del túnel

```
Portátil (cliente)
  interfaz: client
  IP túnel: 10.10.10.2
        │
        │  Endpoint = IP_PUBLICA:51820
        ▼
Router doméstico ([modelo de router del ISP])
  entrada: ppp0.1 (WAN)
        │
        ▼
  Firewall → regla "Allow_WireGuard"
  (UDP 51820, Permit)
        │
        ▼
  NAT → 51820 UDP → 192.168.1.X
  (¡debe ser la MISMA interfaz WAN
   que la regla de firewall: ppp0.1!)
        │
        ▼
Servidor Arch
  interfaz: wg0
  IP túnel: 10.10.10.1
        │
        ▼
  firewalld, zona "public"
  51820/udp + masquerade

──────────────────────────────
Túnel cifrado activo:
10.10.10.2  ⇄  10.10.10.1
(subred 10.10.10.0/24)
```

### 5.2 Archivos — dónde vive cada cosa

**Servidor:**

| Archivo | Ruta | Rol |
|---|---|---|
| Llave privada del servidor | `/etc/wireguard/server_private.key` | Nunca se comparte |
| Llave pública del servidor | `/etc/wireguard/server_public.key` | Se entrega a cada cliente |
| Config de la interfaz | `/etc/wireguard/wg0.conf` | Define `wg0` + lista de peers |
| IP forwarding | `/etc/sysctl.d/99-wireguard.conf` | Habilita enrutar tráfico entre túnel y red |

**Cliente (portátil):**

| Archivo | Ruta (Linux) | Rol |
|---|---|---|
| Llave privada del cliente | `/etc/wireguard/client_private.key` | Nunca se comparte |
| Llave pública del cliente | `/etc/wireguard/client_public.key` | Se entrega al servidor |
| Config de la interfaz | `/etc/wireguard/client.conf` | Define `client` + a qué servidor conectar |

### 5.3 Configuración — servidor

```bash
sudo pacman -S wireguard-tools
cd /etc/wireguard
umask 077
wg genkey | sudo tee server_private.key | wg pubkey | sudo tee server_public.key
```

`/etc/wireguard/wg0.conf` (sin `iptables` manuales — el NAT lo gestiona firewalld):
```ini
[Interface]
Address = 10.10.10.1/24
ListenPort = 51820
PrivateKey = <contenido de server_private.key>
SaveConfig = false

[Peer]
PublicKey = <client_public.key del portátil>
AllowedIPs = 10.10.10.2/32
```

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl --system

sudo systemctl enable --now wg-quick@wg0
sudo wg show
```

### 5.4 Configuración — cliente (portátil)

```bash
wg genkey | tee client_private.key | wg pubkey > client_public.key
sudo mkdir -p /etc/wireguard
sudo mv client_private.key client_public.key /etc/wireguard/
sudo chmod 600 /etc/wireguard/client_private.key
sudo chmod 644 /etc/wireguard/client_public.key
```

`/etc/wireguard/client.conf`:
```ini
[Interface]
PrivateKey = <client_private.key>
Address = 10.10.10.2/24

[Peer]
PublicKey = <server_public.key del servidor>
Endpoint = <TU_IP_PUBLICA>:51820
AllowedIPs = 10.10.10.0/24
PersistentKeepalive = 25
```

> **Sin línea `DNS =`** — se quitó deliberadamente (ver bitácora #2). Como `AllowedIPs` solo cubre la subred del túnel (no `0.0.0.0/0`), no hace falta DNS especial para "solo llegar al servidor".

Uso a demanda (no como servicio permanente — no tiene sentido mantener el túnel siempre activo desde un portátil):
```bash
sudo systemctl disable wg-quick@client   # asegurar que no arranca solo

sudo wg-quick up client      # conectar cuando se necesite
sudo wg-quick down client    # desconectar
```

---

## 6. Router doméstico — NAT y firewall

Router [modelo de router del ISP] (típico de operadores tipo MásOrange/Jazztel en España). Dos configuraciones **independientes**, que deben coincidir en la **misma interfaz WAN** (`ppp0.1` en este caso — no `veip0.2`, ver bitácora #5):

### 6.1 NAT / Port Forwarding

`Advanced Setup → NAT → Virtual Servers`

| Server Name | Ext. Port | Protocol | Int. Port | Server IP | WAN Interface |
|---|---|---|---|---|---|
| WireGuard | 51820 | UDP | 51820 | 192.168.1.X | **ppp0.1** |
| HTTPS | 443 | TCP | 443 | 192.168.1.X | **ppp0.1** |

### 6.2 Firewall — reglas de excepción

`Advanced Setup → Firewall → Rules`

Por defecto, `WAN_DEFAULT` tiene `Default Action: Drop` para todo lo entrante por `ppp0.1` — hacen falta reglas específicas de excepción:

| Campo | Valor (WireGuard) |
|---|---|
| Active | ✓ |
| Rule Name | Allow_WireGuard |
| Interface | ppp0.1 |
| Direction | Incoming |
| Protocol | UDP |
| Source IP/Subnet/Port | *(vacío — cualquier origen)* |
| Destination IP Address | 192.168.1.X |
| Destination Subnet Mask | 255.255.255.255 |
| Destination Port | 51820 : 51820 |
| Action | Permit |

Mismo patrón replicado para 443/tcp.

### 6.3 Reserva DHCP

Asignar IP fija al servidor en el router, para que `192.168.1.X` no cambie y desalinee las reglas de NAT/firewall.

### 6.4 Comprobación de CGNAT (antes de configurar nada de esto)

```bash
curl -4 ifconfig.me
```
Comparar con la IP WAN que muestra el propio panel del router (`Device Info` → estado WAN). Si coinciden, no hay CGNAT y el port forwarding funcionará. Si son distintas, el ISP aplica NAT de operador y ningún port forwarding funcionará — habría que usar Tailscale o Cloudflare Tunnel en su lugar.

---

## 7. fail2ban

Protege SSH y nginx (rate-limit + auth) baneando IPs que insisten demasiado.

### 7.1 Instalación

```bash
sudo pacman -S fail2ban
```

### 7.2 Exportar los logs de nginx del contenedor al host

Los logs de nginx van por defecto a stdout/stderr dentro del contenedor (estándar en Docker), pero fail2ban corre en el host y necesita un archivo real que leer. Se añadió un segundo destino de log en `nginx/nginx.conf`:

```nginx
access_log  /dev/stdout  main;
access_log  /var/log/nginx-host/access.log  main;
error_log   /var/log/nginx-host/error.log warn;
```

Y el volumen correspondiente en `docker-compose.yml` (servicio `nginx`):
```yaml
volumes:
  - ./logs/nginx:/var/log/nginx-host
```

```bash
mkdir -p /opt/server/logs/nginx
docker compose up -d --force-recreate nginx
```

### 7.3 `/etc/fail2ban/jail.local`

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5

# Nunca banear la propia LAN ni la subred VPN, para no bloquearse a uno mismo
ignoreip = 127.0.0.1/8 192.168.1.0/24 10.10.10.0/24

[sshd]
enabled = true
port    = ssh
filter  = sshd
logpath = %(sshd_log)s
backend = %(sshd_backend)s
maxretry = 4

[nginx-http-auth]
enabled = true
filter  = nginx-http-auth
port    = http,https
logpath = /opt/server/logs/nginx/error.log
backend = polling

[nginx-limit-req]
enabled = true
filter  = nginx-limit-req
port    = http,https
logpath = /opt/server/logs/nginx/error.log
backend = polling
maxretry = 10
```

> **`backend = polling` es obligatorio aquí, no opcional.** El backend por defecto de fail2ban usa `inotify` para detectar cambios en el archivo — pero cuando el archivo lo escribe un proceso *dentro de un contenedor* hacia un volumen bind-mount, los eventos de `inotify` no siempre se propagan al host. Con `polling`, fail2ban revisa el archivo activamente en vez de esperar una notificación, y funciona de forma fiable en este escenario. Ver bitácora #11 para el diagnóstico completo.

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
```

### 7.4 Verificación

```bash
sudo fail2ban-client status sshd
sudo fail2ban-client status nginx-http-auth
sudo fail2ban-client status nginx-limit-req
```

### 7.5 Comportamiento importante: `ignoreself` / `ignoreip`

Al probar el rate-limit **desde el propio servidor o desde la LAN/VPN**, fail2ban **nunca** va a banear esa IP — es el comportamiento correcto por diseño:
- Peticiones hechas a `localhost` llegan a nginx como la IP del gateway de Docker (`172.x.x.x`), y fail2ban la ignora automáticamente por la regla `ignoreself` (protege de que el propio host se banee a sí mismo y rompa su red interna).
- Peticiones hechas desde la LAN (`192.168.1.0/24`) o desde la VPN (`10.10.10.0/24`) se ignoran por estar explícitamente en `ignoreip`.

Para ver un baneo real en acción hay que generar tráfico desde una IP externa de verdad (por ejemplo, datos móviles **sin VPN activa**, apuntando directo a la IP pública/dominio):

```bash
for i in $(seq 1 60); do curl -sk -o /dev/null -w "%{http_code}\n" https://<IP_PUBLICA_O_DOMINIO>; done
```

Si el propio IP público llega a banearse haciendo esta prueba (queda 1h sin acceso HTTPS directo, aunque el acceso por VPN/LAN sigue intacto por estar en whitelist), se puede revertir manualmente:
```bash
sudo fail2ban-client set nginx-limit-req unbanip <IP>
```

---

## 8. Cockpit y SSH exclusivos por VPN

Antes: `ssh` y `cockpit` estaban permitidos en la zona `public` de firewalld (alcanzables desde toda internet). Se movieron a la zona `trusted` (donde vive la interfaz `wg0`), quedando accesibles **solo a través del túnel VPN**.

```bash
# Quitar de la zona pública
sudo firewall-cmd --zone=public --remove-service=ssh --permanent
sudo firewall-cmd --zone=public --remove-service=cockpit --permanent

# Añadir a la zona confiable (donde está wg0)
sudo firewall-cmd --zone=trusted --add-service=ssh --permanent
sudo firewall-cmd --zone=trusted --add-service=cockpit --permanent

sudo firewall-cmd --reload
```

Verificación:
```bash
sudo firewall-cmd --zone=public --list-all     # NO debe listar ssh ni cockpit
sudo firewall-cmd --zone=trusted --list-all    # SÍ debe listarlos, junto a la interfaz wg0
```

**Prueba antes de dar por bueno el cambio** (sin cerrar la sesión SSH activa, por seguridad):
```bash
# Sin VPN, desde fuera de la LAN — debe fallar/timeout
ssh usuario@<IP_PUBLICA>

# Con VPN activa — debe funcionar
ssh usuario@10.10.10.1
https://10.10.10.1:9090   # Cockpit, con VPN activa
```

Confirmado y verificado: ambos servicios responden únicamente por `10.10.10.1`, no por la IP pública.

---

## 9. Chuleta — activar la VPN en el cliente

Para no repetir el comando completo cada vez, alias en `~/.bashrc` / `~/.zshrc` del cliente (portátil):

```bash
alias vpn-on='sudo wg-quick up client'
alias vpn-off='sudo wg-quick down client'
alias vpn-status='sudo wg show'
```

```bash
source ~/.zshrc   # o ~/.bashrc, según la shell
```

Uso diario:
```bash
vpn-on        # conectar antes de necesitar acceso al servidor (SSH, Cockpit, admin)
vpn-status    # confirmar handshake reciente y tráfico
vpn-off       # desconectar al terminar
```

**Mantenimiento necesario si la IP pública de casa cambia** (es dinámica — resuelto de forma permanente cuando se configure DuckDNS, ver Pendientes):
```bash
# en el servidor
curl -4 ifconfig.me

# comparar con lo que tiene el cliente
grep Endpoint /etc/wireguard/client.conf
```
Si no coinciden, actualizar `Endpoint =` en el cliente con la IP nueva.

---

## 10. Bitácora completa de errores resueltos

### Docker / nginx

| # | Síntoma | Causa | Solución |
|---|---|---|---|
| 1 | `no configuration file provided: not found` (`docker compose exec`) | Faltaban `nginx.conf`/`default.conf` reales, o volumen apuntando a carpeta vacía | Crear los archivos con contenido antes de levantar |
| 2 | `no configuration file provided: not found` (`docker compose up`) | Archivo se llamaba `docker_compose.yml` (guion bajo) en vez de `docker-compose.yml` | `mv docker_compose.yml docker-compose.yml` |
| 3 | `error empty compose file` | `docker-compose.yml` existía pero vacío | Pegar el YAML completo |
| 4 | `unable to prepare context: ... Dockerfile ... no such file` | Servicio `app` sin `Dockerfile` en `./app/` | Crear `Dockerfile` + código mínimo |
| 5 | `failed to bind host port 0.0.0.0:80/tcp: address already in use` | nginx nativo de Arch seguía corriendo, ocupando el puerto | `systemctl stop nginx && systemctl disable nginx` |
| 6 | Contenedor `nginx` en `Exited (1)`, `host not found in upstream "app"` | `proxy_pass` estático resuelve DNS una sola vez al arrancar; si `app` no está listo, nginx muere | `resolver 127.0.0.11 valid=10s;` + `set $upstream_app app:8000;` + `proxy_pass http://$upstream_app/;` (resolución dinámica) |
| 7 | `curl https://localhost` → `Connection refused` | nginx corría sin el mapeo de puertos tras editar el compose con el contenedor ya vivo | `docker compose down && docker compose up -d` |
| 8 | Rate limiting daba `404` en vez de `503` | `error_page 503 → /50x.html` pero el archivo no existía | Crear `web/50x.html` |
| 9 | `unknown directive "httpd2"` | Typo: `httpd2 on;` en vez de `http2 on;` | Corregir el typo |
| 10 | `cannot load certificate ... No such file or directory` | Certificado no generado, o volumen `./nginx/certs` no montado | Generar con `openssl` + agregar volumen + `--force-recreate` |

### WireGuard / router

| # | Síntoma | Causa real | Solución |
|---|---|---|---|
| 1 | `wg-quick: 'client' is not a WireGuard interface` / `Name or service not known` | `Endpoint =` con el placeholder sin reemplazar | Sustituir por la IP real (`curl -4 ifconfig.me`) |
| 2 | `resolvconf: command not found`, la interfaz se autodestruye | `DNS = 1.1.1.1` requiere `resolvconf`/`openresolv`, no instalado | Eliminar la línea `DNS =` (no era necesaria para este caso de uso) |
| 3 | `ping`/`curl` fallan con interfaz arriba, `0 B received` | Pruebas dentro de la misma LAN apuntando a la IP pública → **NAT hairpinning** | Repetir pruebas desde otra red (datos móviles) |
| 4 | Sigue sin llegar tráfico desde datos móviles; `nmap -sU` da `open\|filtered` | Firewall del router con `WAN_DEFAULT: Drop` por defecto | Crear regla de excepción explícita `Allow_WireGuard` (Permit) |
| 5 | Tras la excepción, sigue sin llegar tráfico | Regla de NAT en interfaz `veip0.2`, firewall y tráfico real en `ppp0.1` — desalineadas | Editar el NAT para usar `ppp0.1`, igual que el firewall |
| 6 | `nc -ul 51820 > archivo.log`: `permission denied` | Archivo de log con permisos de una ejecución anterior | `sudo rm -f archivo.log` o usar nombre nuevo cada vez |
| 7 | `wg-quick: 'wg0' already exists` | Interfaz levantada manualmente antes, systemd no puede recrearla | `wg-quick down wg0` + `systemctl restart wg-quick@wg0` |
| 8 | Ping funciona por túnel, `curl https://10.10.10.1` falla | Puerto 443/tcp sin su propia regla de NAT + excepción de firewall | Replicar el mismo patrón NAT + firewall, esta vez para 443/tcp |

**Lección general:** un router doméstico con firewall propio tiene dos capas independientes (NAT y firewall) que deben coincidir en la misma interfaz WAN. Fallar en cualquiera de las dos —o tenerlas en interfaces distintas— produce el mismo síntoma confuso ("no llega nada"). La forma más rápida de aislarlo es netcat manual (`nc -ul` en el servidor + `echo | nc -u` desde fuera), que separa "¿llega el paquete?" de "¿responde el servicio?" sin la complejidad del protocolo de por medio.

**Lección general (Docker):** cualquier cambio en `volumes:`, `ports:` o el `Dockerfile` de un servicio requiere recrear el contenedor (`--force-recreate` o `down && up`) — no basta con editar el archivo. Cambios solo en `nginx/conf.d/*.conf` sí se aplican en caliente con `nginx -s reload`.

### fail2ban

| # | Síntoma | Causa real | Solución |
|---|---|---|---|
| 11 | `Failed during configuration: Have not found any log file for 'nginx-http-auth' jail` — servicio entero no arranca | El jail `nginx-http-auth` apuntaba a un log que aún no existía (`docker-compose.yml` sin recrear tras añadir el volumen de logs) | Recrear nginx primero (`--force-recreate`) para que el archivo exista, y solo entonces arrancar fail2ban |
| 12 | Jail sigue sin detectar nada tras corregir el log path | `nginx-http-auth` apuntaba a `/var/log/nginx/error.log` (ruta nativa, inexistente en este setup) en vez de `/opt/server/logs/nginx/error.log` | Corregir el `logpath` a la ruta real montada desde Docker |
| 13 | `fail2ban-client status`: `Failed to access socket path` | Se ejecutó el comando demasiado rápido tras el `restart`, el socket aún no se había creado | Esperar 1-2 segundos tras el restart, o reintentar |
| 14 | `fail2ban-regex` confirma 12/12 líneas matcheadas offline, pero `status` en vivo sigue en `Currently failed: 0` | El backend por defecto de fail2ban (basado en `inotify`) no detecta cambios en un archivo escrito por un proceso *dentro de un contenedor Docker* hacia un volumen bind-mount — el evento de modificación no siempre se propaga al host | Forzar `backend = polling` en los jails de nginx (revisa el archivo activamente en vez de esperar una notificación) |
| 15 | Con `polling` activo, prueba desde `localhost` sigue sin banear nada; el log de journal (`journalctl -u fail2ban`) no muestra nada de actividad de filtros | fail2ban no escribe su actividad interna al journal de systemd por defecto, solo eventos de arranque/parada — el log real está en `/var/log/fail2ban.log` | Consultar `/var/log/fail2ban.log` (o `journalctl -t fail2ban-server`) en vez de `journalctl -u fail2ban` para ver actividad de filtros con `loglevel DEBUG` |
| 16 | `docker compose ps` de repente solo muestra `app`, ni `nginx` ni `db` — de ahí que las pruebas dieran `000`/`Connection refused` durante un tramo del diagnóstico | Los contenedores se habían detenido/no recreado correctamente en algún punto de los múltiples `--force-recreate` de la sesión | `docker compose up -d` desde `/opt/server` los recupera; usar `docker compose ps -a` para detectar esto temprano en vez de asumir que siguen corriendo |
| 17 | Con todo correcto (backend polling, log real, tráfico real generado), `fail2ban-client status` seguía en `0` | El log de fail2ban en DEBUG revela `[nginx-limit-req] Ignore <IP> by ignoreself rule` / `by ip` — la IP de prueba (`172.18.0.1` del gateway Docker, o `10.10.10.2` de la VPN) estaba cubierta por la regla `ignoreself` o por `ignoreip` (`10.10.10.0/24` incluye la subred VPN a propósito) | No era un fallo: es el comportamiento correcto y buscado (nunca banear la propia infraestructura de administración). Para verificar un baneo real hace falta probar desde una IP externa fuera de `ignoreip` (datos móviles sin VPN, apuntando a la IP pública) |

**Lección general (fail2ban + Docker):** cuando el servicio protegido corre en contenedores, hay tres capas nuevas de posible fallo silencioso que no existen en un despliegue nativo: (1) los logs deben exportarse explícitamente del contenedor al host, (2) el backend por defecto de detección de cambios (`inotify`) puede no funcionar sobre logs escritos desde dentro de un contenedor hacia un bind-mount — usar `polling`, y (3) fail2ban tiene sus propias reglas de "no te bloquees a ti mismo" (`ignoreself`, `ignoreip`) que hacen que probar desde la propia LAN/VPN/host nunca muestre un baneo, aunque todo esté funcionando correctamente.

---

## 11. Comandos de referencia rápida

### Docker

```bash
cd /opt/server
docker compose up -d
docker compose ps -a
docker compose logs -f nginx
docker compose exec nginx nginx -t
docker compose exec nginx nginx -s reload
docker compose build app && docker compose up -d app
docker compose down            # mantiene volúmenes
docker compose down -v         # borra también volúmenes (¡incluye datos de postgres!)
```

### WireGuard

```bash
# Servidor
sudo wg show
sudo systemctl status wg-quick@wg0
sudo systemctl restart wg-quick@wg0

# Cliente
sudo wg-quick up client
sudo wg-quick down client
sudo wg show

# Diagnóstico (aislar router vs. WireGuard)
sudo systemctl stop wg-quick@wg0
sudo timeout 60 nc -ul 51820 > /tmp/nc_test.log 2>&1 &
cat /tmp/nc_test.log
sudo systemctl start wg-quick@wg0
# desde el cliente, en la ventana de 60s:
echo "prueba" | nc -u -w1 <IP_PUBLICA> 51820
```

### fail2ban

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo fail2ban-client status nginx-limit-req
sudo fail2ban-client set nginx-limit-req unbanip <IP>   # desbanear manualmente
sudo tail -f /var/log/fail2ban.log                       # log real (no journalctl -u)
```

### Pruebas de humo

```bash
docker compose ps
curl -I http://localhost              # debe redirigir (301) a https
curl -k -I https://localhost          # -k: certificado autofirmado
curl -k https://localhost/api/

ping 10.10.10.1                        # desde fuera de la red, con VPN activa
curl -k https://10.10.10.1
```

---

## 12. Decisiones de diseño

- **nginx en Docker, no nativo**: aislamiento del proceso, filesystem desechable, se integra con la segmentación de redes docker.
- **`resolver` + variable en `proxy_pass`**: nginx resuelve el DNS interno en cada petición, no solo al arrancar — evita que un orden de arranque distinto tumbe nginx.
- **`net_backend` con `internal: true`**: PostgreSQL sin salida a internet ni ruta directa desde `net_dmz`, aunque un contenedor fuera comprometido.
- **Solo nginx publica puertos al host**: reduce la superficie expuesta a un único punto de entrada controlado.
- **WireGuard sobre alternativas más pesadas (OpenVPN)**: más simple, integrado en el kernel, mejor rendimiento.
- **Cliente WireGuard a demanda, no como servicio permanente**: un portátil no necesita el túnel siempre activo.
- **NAT + firewall del router en la misma interfaz WAN**: requisito no evidente en routers con múltiples interfaces virtuales (`ppp0.1` vs `veip0.2`), documentado explícitamente para no repetir el error.
- **`ignoreip` incluye la propia LAN y subred VPN**: evita que fail2ban banee accidentalmente al propio administrador por probar demasiadas veces seguidas — a costa de que las pruebas locales nunca muestren un baneo real (comportamiento esperado, no un bug).
- **Cockpit y SSH solo por VPN, nunca en `public`**: reduce la superficie de ataque de los servicios de administración a un único punto controlado (el túnel WireGuard), en vez de exponerlos directamente a todo internet.

---

## 13. Pendientes generales

1. **Comprobar CGNAT** antes de dar por sentado que el port forwarding sirve (ya confirmado que NO hay, en este caso).
2. **Dominio real** — apuntar registro `A` a la IP pública, o usar DuckDNS si la IP es dinámica.
3. **DuckDNS** — sustituir la IP fija en `Endpoint =` de WireGuard por un hostname, para no editar `client.conf` cada vez que cambie la IP pública dinámica.
4. **Certificado real de Let's Encrypt** — vía certbot, una vez el dominio apunte y el puerto 80 sea alcanzable. Solo cambian 3 líneas en `default.conf`.
5. **Renovación automática del certificado** — systemd timer + reload de nginx.
6. **Mecanismo de visibilidad pública on/off** (si se necesita) — toggle de regla firewalld con timeout, o lógica en nginx.
7. **Prueba de baneo real con fail2ban** — desde una IP externa fuera de `ignoreip` (datos móviles sin VPN), para confirmar el baneo efectivo, no solo la detección.

~~fail2ban~~ y ~~Cockpit/SSH exclusivos por VPN~~ — resueltos, ver secciones [7](#7-fail2ban) y [8](#8-cockpit-y-ssh-exclusivos-por-vpn).