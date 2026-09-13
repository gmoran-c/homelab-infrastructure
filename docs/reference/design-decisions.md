# Decisiones de diseño

> Navegación: [índice de documentación](../README.md) · [README del proyecto](../../README.md)

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
