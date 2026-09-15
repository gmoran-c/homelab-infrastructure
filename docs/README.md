# Documentación del HomeLab

> Navegación: [README del proyecto](../README.md)


Documentación mantenible del servidor doméstico autoalojado con Arch Linux, Docker, nginx, PostgreSQL y WireGuard. Los valores sensibles permanecen como placeholders; no se incluyen credenciales reales.

## Estado documentado

- ✅ Docker (nginx + app + db) segmentado en redes internas
- ✅ HTTPS funcionando (certificado autofirmado — pendiente real de Let's Encrypt)
- ✅ Rate limiting activo y verificado
- ✅ firewalld configurado en el servidor
- ✅ WireGuard funcionando de extremo a extremo, verificado desde fuera de la red local
- ✅ fail2ban activo (sshd + nginx), verificado con logs reales
- ✅ Cockpit y SSH restringidos exclusivamente a la VPN (zona `trusted`), fuera de `public`
- ⏳ Pendiente: dominio + certificado real, DuckDNS (ver [Pendientes](operations/roadmap.md))

> Este estado procede de la documentación de origen. Verifica cada punto en el host antes de usarlo como inventario operativo. Los backups y restores descritos en [operaciones de backup](operations/backup-restore.md) son un procedimiento recomendado, no una funcionalidad declarada como implementada.

## Índice

### Visión general

- [Arquitectura y estado](overview/architecture.md)
- [Decisiones de diseño](reference/design-decisions.md)
- [Matriz de exposición](security/exposure-matrix.md)

### Despliegue

- [Docker, nginx, aplicación y PostgreSQL](deployment/docker.md)

### Red

- [WireGuard](network/wireguard.md)
- [firewalld y router](network/firewalld-router.md)

### Seguridad

- [fail2ban](security/fail2ban.md)
- [Acceso administrativo por VPN](security/access-control.md)

### Operaciones

- [Referencia rápida](operations/quick-reference.md)
- [Backup y restore recomendado](operations/backup-restore.md)
- [Pendientes](operations/roadmap.md)

### Diagnóstico

- [Bitácora de incidencias](troubleshooting/incident-log.md)

## Convenciones de seguridad

- Conserva `<TU_IP_PUBLICA>`, `192.0.2.X`, `wan0`, `<...>` y las claves placeholder hasta completar los valores localmente.
- Mantén `.env`, claves WireGuard, claves TLS y otros secretos fuera del control de versiones; `.gitignore` se conserva en la raíz.
- Antes de ejecutar comandos destructivos (por ejemplo `docker compose down -v` o un restore con `--clean`), confirma el objetivo y el backup.
