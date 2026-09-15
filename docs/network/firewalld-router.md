# Red: firewalld y router

> Navegación: [índice de documentación](../README.md) · [README del proyecto](../../README.md)

Reglas del host, NAT y firewall del router doméstico. Mantén los placeholders (`<TU_IP_PUBLICA>`, `192.0.2.X`, `wan0`) hasta sustituirlos localmente.

## 4. firewalld — servidor

```bash
sudo systemctl enable --now firewalld

firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --zone=public --add-port=443/tcp --permanent
firewall-cmd --reload

firewall-cmd --list-all   # verificación
```

Configuración final, incluyendo lo añadido para [WireGuard](wireguard.md):

```bash
sudo firewall-cmd --zone=public --add-port=51820/udp --permanent
sudo firewall-cmd --zone=public --add-masquerade --permanent
sudo firewall-cmd --permanent --zone=trusted --add-interface=wg0
sudo firewall-cmd --zone=trusted --add-port=443/tcp --permanent
sudo firewall-cmd --reload
```

---

## 6. Router doméstico — NAT y firewall

Router [modelo de router del ISP] (típico de operadores tipo proveedor de Internet). Dos configuraciones **independientes**, que deben coincidir en la **misma interfaz WAN** (`wan0` en este caso — no `wan1`, ver bitácora #5):

### 6.1 NAT / Port Forwarding

`Advanced Setup → NAT → Virtual Servers`

| Server Name | Ext. Port | Protocol | Int. Port | Server IP | WAN Interface |
|---|---|---|---|---|---|
| WireGuard | 51820 | UDP | 51820 | 192.0.2.X | **wan0** |
| HTTPS | 443 | TCP | 443 | 192.0.2.X | **wan0** |

### 6.2 Firewall — reglas de excepción

`Advanced Setup → Firewall → Rules`

Por defecto, `WAN_DEFAULT` tiene `Default Action: Drop` para todo lo entrante por `wan0` — hacen falta reglas específicas de excepción:

| Campo | Valor (WireGuard) |
|---|---|
| Active | ✓ |
| Rule Name | Allow_WireGuard |
| Interface | wan0 |
| Direction | Incoming |
| Protocol | UDP |
| Source IP/Subnet/Port | *(vacío — cualquier origen)* |
| Destination IP Address | 192.0.2.X |
| Destination Subnet Mask | 255.255.255.255 |
| Destination Port | 51820 : 51820 |
| Action | Permit |

Mismo patrón replicado para 443/tcp.

### 6.3 Reserva DHCP

Asignar IP fija al servidor en el router, para que `192.0.2.X` no cambie y desalinee las reglas de NAT/firewall.

### 6.4 Comprobación de CGNAT (antes de configurar nada de esto)

```bash
curl -4 ifconfig.me
```
Comparar con la IP WAN que muestra el propio panel del router (`Device Info` → estado WAN). Si coinciden, no hay CGNAT y el port forwarding funcionará. Si son distintas, el ISP aplica NAT de operador y ningún port forwarding funcionará — habría que usar Tailscale o Cloudflare Tunnel en su lugar.

---
