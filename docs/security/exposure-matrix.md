# Matriz de exposición de servicios

> Navegación: [índice de documentación](../README.md) · [README del proyecto](../../README.md)

Esta matriz resume la superficie descrita en la documentación. “Documentado” no sustituye una comprobación del router, firewalld y Docker en el entorno real.

| Servicio / flujo | Transporte y puerto | Desde Internet | Desde VPN (`10.99.0.0/24`) | Alcance esperado |
|---|---|---|---|---|
| nginx / web | TCP 80 | Publicado; redirige a HTTPS | Accesible si la regla lo permite | Único frontend publicado |
| nginx / HTTPS | TCP 443 | Publicado; certificado autofirmado temporal | Accesible por `10.99.0.1` | Entrada web y `/api/` |
| WireGuard | UDP 51820 | Publicado para establecer el túnel | No aplica | Único punto de entrada VPN |
| Aplicación | TCP 8000 | No publicado | No directo; se alcanza mediante nginx | Solo redes Docker (`net_dmz`/`net_backend`) |
| PostgreSQL | TCP 5432 | No publicado | No publicado directamente | Red Docker interna (`net_backend`) |
| SSH | TCP 22 | No; no debe estar en `public` | Sí, mediante `trusted`/`wg0` | Administración por VPN |
| Cockpit | TCP 9090 | No; no debe estar en `public` | Sí, mediante `trusted`/`wg0` | Administración por VPN |

## Comprobaciones mínimas

- Revisar `docker compose ps` y confirmar que solo nginx publica puertos al host.
- Revisar `firewall-cmd --zone=public --list-all` y `--zone=trusted --list-all`.
- Revisar el NAT y las excepciones del router en la misma interfaz WAN (`wan0`).
- Probar desde una red externa real; las pruebas desde la LAN pueden verse afectadas por NAT hairpinning.
