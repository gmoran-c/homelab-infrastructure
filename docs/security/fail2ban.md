# Seguridad: fail2ban

> Navegación: [índice de documentación](../README.md) · [README del proyecto](../../README.md)

Protección de SSH y nginx, exportación de logs desde Docker y verificación.

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

> **`backend = polling` es obligatorio aquí, no opcional.** El backend por defecto de fail2ban usa `inotify` para detectar cambios en el archivo — pero cuando el archivo lo escribe un proceso *dentro de un contenedor* hacia un volumen bind-mount, los eventos de `inotify` no siempre se propagan al host. Con `polling`, fail2ban revisa el archivo activamente en vez de esperar una notificación, y funciona de forma fiable en este escenario. Consulta la [bitácora de incidencias](../troubleshooting/incident-log.md), caso 11, para el diagnóstico completo.

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
