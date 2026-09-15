# HomeLab

Documentación profesionalizada de un servidor doméstico autoalojado con Arch Linux, Docker, nginx, PostgreSQL y WireGuard.

> **Scope:** This repository documents a personal homelab deployment and its
> operational lessons. It is not a turnkey production stack. Review every
> command, replace placeholders locally, and validate the security impact before
> exposing any service to the Internet.

## Estado

La configuración documentada incluye Docker segmentado, HTTPS con certificado autofirmado temporal, rate limiting, firewalld, WireGuard, fail2ban y administración de Cockpit/SSH limitada a la VPN. El dominio, Let's Encrypt, DuckDNS y algunos mecanismos operativos siguen figurando como pendientes.

Los procedimientos de backup/restore de PostgreSQL y configuración están documentados como **recomendados**, no como automatizaciones ya implementadas. Verifica el estado real del host antes de aplicar cualquier instrucción.

## Qué demuestra el proyecto

- Segmentación de servicios con Docker y redes internas.
- Publicación controlada mediante nginx, HTTPS y rate limiting.
- Acceso administrativo restringido mediante WireGuard.
- Firewall del servidor y del router coordinados.
- Detección y diagnóstico de fallos con fail2ban, logs y pruebas desde una red externa.
- Documentación operativa de incidencias, recuperación y decisiones de diseño.

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

Las subredes privadas y nombres de interfaz que aparecen son valores de ejemplo
o de laboratorio; deben sustituirse por los de cada instalación. No se incluyen
claves privadas, certificados, contraseñas, tokens, dominios reales ni copias de
logs del host.

## Limitaciones actuales

- El repositorio contiene documentación y ejemplos, no todos los archivos
  necesarios para reconstruir el servicio automáticamente.
- El certificado descrito es autofirmado; Let's Encrypt y DuckDNS siguen
  pendientes.
- El backup/restore está documentado como procedimiento recomendado, no como
  automatización implementada.
- No se debe interpretar la documentación como una auditoría de seguridad ni
  como garantía de configuración segura en otro entorno.
