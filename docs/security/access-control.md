# Seguridad: acceso administrativo

> Navegación: [índice de documentación](../README.md) · [README del proyecto](../../README.md)

SSH y Cockpit deben quedar accesibles únicamente a través de la interfaz WireGuard (`wg0`).

## 8. Cockpit y SSH exclusivos por VPN

Antes: `ssh` y `cockpit` estaban permitidos en la zona `public` de firewalld (alcanzables desde toda internet). Se movieron a la zona `trusted` (donde vive la interfaz `wg0`), quedando accesibles **solo a través del túnel VPN**.

```bash
# Quitar de la zona pública
sudo firewall-cmd --zone=public --remove-service=ssh --permanent
sudo firewall-cmd --zone=public --remove-service=cockpit --permanent

# Añadir a la zona confiable (donde está wg0)
sudo firewall-cmd --zone=trusted --add-service=ssh --permanent
sudo firewall-cmd --zone=trusted --add-service=cockpit --permanent

sudo firewall-cmd --reload
```

Verificación:
```bash
sudo firewall-cmd --zone=public --list-all     # NO debe listar ssh ni cockpit
sudo firewall-cmd --zone=trusted --list-all    # SÍ debe listarlos, junto a la interfaz wg0
```

**Prueba antes de dar por bueno el cambio** (sin cerrar la sesión SSH activa, por seguridad):
```bash
# Sin VPN, desde fuera de la LAN — debe fallar/timeout
ssh usuario@<IP_PUBLICA>

# Con VPN activa — debe funcionar
ssh usuario@10.99.0.1
https://10.99.0.1:9090   # Cockpit, con VPN activa
```

Confirmado y verificado: ambos servicios responden únicamente por `10.99.0.1`, no por la IP pública.

---
