# Resolución de incidencias

> Navegación: [índice de documentación](../README.md) · [README del proyecto](../../README.md)

Bitácora de síntomas, causas y soluciones documentadas durante el montaje. Las soluciones que contienen comandos deben ejecutarse desde el contexto y con los permisos adecuados.

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
| 5 | Tras la excepción, sigue sin llegar tráfico | Regla de NAT en interfaz `wan1`, firewall y tráfico real en `wan0` — desalineadas | Editar el NAT para usar `wan0`, igual que el firewall |
| 6 | `nc -ul 51820 > archivo.log`: `permission denied` | Archivo de log con permisos de una ejecución anterior | `sudo rm -f archivo.log` o usar nombre nuevo cada vez |
| 7 | `wg-quick: 'wg0' already exists` | Interfaz levantada manualmente antes, systemd no puede recrearla | `wg-quick down wg0` + `systemctl restart wg-quick@wg0` |
| 8 | Ping funciona por túnel, `curl https://10.99.0.1` falla | Puerto 443/tcp sin su propia regla de NAT + excepción de firewall | Replicar el mismo patrón NAT + firewall, esta vez para 443/tcp |

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
| 17 | Con todo correcto (backend polling, log real, tráfico real generado), `fail2ban-client status` seguía en `0` | El log de fail2ban en DEBUG revela `[nginx-limit-req] Ignore <IP> by ignoreself rule` / `by ip` — la IP de prueba (`172.18.0.1` del gateway Docker, o `10.99.0.2` de la VPN) estaba cubierta por la regla `ignoreself` o por `ignoreip` (`10.99.0.0/24` incluye la subred VPN a propósito) | No era un fallo: es el comportamiento correcto y buscado (nunca banear la propia infraestructura de administración). Para verificar un baneo real hace falta probar desde una IP externa fuera de `ignoreip` (datos móviles sin VPN, apuntando a la IP pública) |

**Lección general (fail2ban + Docker):** cuando el servicio protegido corre en contenedores, hay tres capas nuevas de posible fallo silencioso que no existen en un despliegue nativo: (1) los logs deben exportarse explícitamente del contenedor al host, (2) el backend por defecto de detección de cambios (`inotify`) puede no funcionar sobre logs escritos desde dentro de un contenedor hacia un bind-mount — usar `polling`, y (3) fail2ban tiene sus propias reglas de "no te bloquees a ti mismo" (`ignoreself`, `ignoreip`) que hacen que probar desde la propia LAN/VPN/host nunca muestre un baneo, aunque todo esté funcionando correctamente.

---
