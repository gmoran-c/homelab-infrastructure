# Arquitectura y estado

> Navegación: [índice de documentación](../README.md) · [README del proyecto](../../README.md)

Documentación del diseño lógico del servidor doméstico: entrada pública, segmentación Docker, VPN y administración. Las rutas y valores sensibles siguen siendo placeholders.

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
                    10.99.0.0/24)    puertos publicados)
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

## Matriz de exposición

Consulta la [matriz de exposición de servicios](../security/exposure-matrix.md) para revisar qué llega desde Internet, qué requiere la VPN y qué queda solo dentro de Docker.

## 12. Decisiones de diseño

- **nginx en Docker, no nativo**: aislamiento del proceso, filesystem desechable, se integra con la segmentación de redes docker.
- **`resolver` + variable en `proxy_pass`**: nginx resuelve el DNS interno en cada petición, no solo al arrancar — evita que un orden de arranque distinto tumbe nginx.
- **`net_backend` con `internal: true`**: PostgreSQL sin salida a internet ni ruta directa desde `net_dmz`, aunque un contenedor fuera comprometido.
- **Solo nginx publica puertos al host**: reduce la superficie expuesta a un único punto de entrada controlado.
- **WireGuard sobre alternativas más pesadas (OpenVPN)**: más simple, integrado en el kernel, mejor rendimiento.
- **Cliente WireGuard a demanda, no como servicio permanente**: un portátil no necesita el túnel siempre activo.
- **NAT + firewall del router en la misma interfaz WAN**: requisito no evidente en routers con múltiples interfaces virtuales (`wan0` vs `wan1`), documentado explícitamente para no repetir el error.
- **`ignoreip` incluye la propia LAN y subred VPN**: evita que fail2ban banee accidentalmente al propio administrador por probar demasiadas veces seguidas — a costa de que las pruebas locales nunca muestren un baneo real (comportamiento esperado, no un bug).
- **Cockpit y SSH solo por VPN, nunca en `public`**: reduce la superficie de ataque de los servicios de administración a un único punto controlado (el túnel WireGuard), en vez de exponerlos directamente a todo internet.

---
