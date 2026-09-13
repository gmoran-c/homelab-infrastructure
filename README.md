# HomeLab

Documentación profesionalizada de un servidor doméstico autoalojado con Arch Linux, Docker, nginx, PostgreSQL y WireGuard.

## Estado

La configuración documentada incluye Docker segmentado, HTTPS con certificado autofirmado temporal, rate limiting, firewalld, WireGuard, fail2ban y administración de Cockpit/SSH limitada a la VPN. El dominio, Let's Encrypt, DuckDNS y algunos mecanismos operativos siguen figurando como pendientes.

Los procedimientos de backup/restore de PostgreSQL y configuración están documentados como **recomendados**, no como automatizaciones ya implementadas. Verifica el estado real del host antes de aplicar cualquier instrucción.

## Índice de documentación

- [Documentación completa](docs/README.md)
- [Arquitectura](docs/overview/architecture.md)
- [Despliegue Docker](docs/deployment/docker.md)
- [WireGuard](docs/network/wireguard.md)
- [firewalld y router](docs/network/firewalld-router.md)
- [Seguridad y exposición](docs/security/exposure-matrix.md)
- [fail2ban](docs/security/fail2ban.md)
- [Operaciones y backup/restore](docs/operations/quick-reference.md)
- [Backup/restore recomendado](docs/operations/backup-restore.md)
- [Troubleshooting](docs/troubleshooting/incident-log.md)
- [Pendientes](docs/operations/roadmap.md)

## Seguridad documental

La documentación usa placeholders como `<TU_IP_PUBLICA>`, `192.168.1.X`, `ppp0.1` y `<...>` para IPs, dominios, identificadores y secretos. Las credenciales reales deben mantenerse fuera del repositorio, por ejemplo en un `.env` local no versionado. `.gitignore` se conserva sin cambios.
