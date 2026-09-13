# Operaciones: pendientes

> Navegación: [índice de documentación](../README.md) · [README del proyecto](../../README.md)

Elementos aún pendientes o que requieren una comprobación explícita. No se deben interpretar como implementados por aparecer en esta lista.

## 13. Pendientes generales

1. **Comprobar CGNAT** antes de dar por sentado que el port forwarding sirve (ya confirmado que NO hay, en este caso).
2. **Dominio real** — apuntar registro `A` a la IP pública, o usar DuckDNS si la IP es dinámica.
3. **DuckDNS** — sustituir la IP fija en `Endpoint =` de WireGuard por un hostname, para no editar `client.conf` cada vez que cambie la IP pública dinámica.
4. **Certificado real de Let's Encrypt** — vía certbot, una vez el dominio apunte y el puerto 80 sea alcanzable. Solo cambian 3 líneas en `default.conf`.
5. **Renovación automática del certificado** — systemd timer + reload de nginx.
6. **Mecanismo de visibilidad pública on/off** (si se necesita) — toggle de regla firewalld con timeout, o lógica en nginx.
7. **Prueba de baneo real con fail2ban** — desde una IP externa fuera de `ignoreip` (datos móviles sin VPN), para confirmar el baneo efectivo, no solo la detección.

~~fail2ban~~ y ~~Cockpit/SSH exclusivos por VPN~~ — resueltos, ver [fail2ban](../security/fail2ban.md) y [acceso administrativo por VPN](../security/access-control.md).

## Objetivos de profesionalización

Estos objetivos describen mejoras futuras. No deben interpretarse como
implementadas hasta que exista una configuración, una prueba y una
verificación documentadas.

### 1. Backup automatizado real

- Crear un script o un timer de systemd para ejecutar backups.
- Definir un destino externo al servidor.
- Establecer una política de retención.
- Cifrar las copias antes de almacenarlas fuera del host.
- Programar y documentar pruebas periódicas de restauración.

El procedimiento manual recomendado está en
[backup y restore](backup-restore.md).

### 2. Configuración reproducible

- Añadir un `.env.example` sin secretos.
- Separar ejemplos de `docker-compose.yml`, nginx y WireGuard.
- Documentar una instalación desde cero.
- Probar esa instalación en una máquina limpia o entorno aislado.

### 3. Validaciones automáticas

- Validar la configuración con `docker compose config`.
- Comprobar automáticamente los enlaces Markdown.
- Incorporar detección de secretos.
- Validar la configuración de nginx con `nginx -t`.

### 4. Inventario formal

- Registrar las versiones exactas de Arch Linux, Docker, nginx,
  PostgreSQL y WireGuard.
- Documentar requisitos mínimos de hardware y software.
- Mantener una tabla de puertos y dependencias.
- Añadir una fecha de última verificación del inventario.

La [matriz de exposición](../security/exposure-matrix.md) cubre actualmente
los puertos y el alcance esperado, pero no sustituye un inventario versionado.

### 5. Licencia

- Elegir y añadir una licencia adecuada para un repositorio público, por
  ejemplo MIT o Apache-2.0.
- Confirmar que la licencia elegida refleja el uso previsto de la
  documentación.

### 6. Separación formal del estado

- Diferenciar qué está activo actualmente.
- Indicar qué fue probado una vez.
- Marcar qué es una recomendación.
- Mantener una lista de pendientes verificable.

El índice de documentación ya advierte de esta diferencia; este objetivo
consiste en aplicarla de forma uniforme a todos los documentos.
