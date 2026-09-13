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
