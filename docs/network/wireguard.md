# Red: WireGuard

> Navegación: [índice de documentación](../README.md) · [README del proyecto](../../README.md)

Configuración de la VPN y guía de operación del cliente. No se incluyen claves reales: los valores entre `<...>` deben proceder de archivos locales protegidos.

## 5. WireGuard — VPN self-hosteada

**Estado: funcional.** Túnel establecido y verificado desde fuera de la red local (datos móviles), con acceso confirmado a HTTPS del servidor a través de la VPN.

### 5.1 Arquitectura del túnel

```
Portátil (cliente)
  interfaz: client
  IP túnel: 10.10.10.2
        │
        │  Endpoint = IP_PUBLICA:51820
        ▼
Router doméstico ([modelo de router del ISP])
  entrada: ppp0.1 (WAN)
        │
        ▼
  Firewall → regla "Allow_WireGuard"
  (UDP 51820, Permit)
        │
        ▼
  NAT → 51820 UDP → 192.168.1.X
  (¡debe ser la MISMA interfaz WAN
   que la regla de firewall: ppp0.1!)
        │
        ▼
Servidor Arch
  interfaz: wg0
  IP túnel: 10.10.10.1
        │
        ▼
  firewalld, zona "public"
  51820/udp + masquerade

──────────────────────────────
Túnel cifrado activo:
10.10.10.2  ⇄  10.10.10.1
(subred 10.10.10.0/24)
```

### 5.2 Archivos — dónde vive cada cosa

**Servidor:**

| Archivo | Ruta | Rol |
|---|---|---|
| Llave privada del servidor | `/etc/wireguard/server_private.key` | Nunca se comparte |
| Llave pública del servidor | `/etc/wireguard/server_public.key` | Se entrega a cada cliente |
| Config de la interfaz | `/etc/wireguard/wg0.conf` | Define `wg0` + lista de peers |
| IP forwarding | `/etc/sysctl.d/99-wireguard.conf` | Habilita enrutar tráfico entre túnel y red |

**Cliente (portátil):**

| Archivo | Ruta (Linux) | Rol |
|---|---|---|
| Llave privada del cliente | `/etc/wireguard/client_private.key` | Nunca se comparte |
| Llave pública del cliente | `/etc/wireguard/client_public.key` | Se entrega al servidor |
| Config de la interfaz | `/etc/wireguard/client.conf` | Define `client` + a qué servidor conectar |

### 5.3 Configuración — servidor

```bash
sudo pacman -S wireguard-tools
cd /etc/wireguard
umask 077
wg genkey | sudo tee server_private.key | wg pubkey | sudo tee server_public.key
```

`/etc/wireguard/wg0.conf` (sin `iptables` manuales — el NAT lo gestiona firewalld):
```ini
[Interface]
Address = 10.10.10.1/24
ListenPort = 51820
PrivateKey = <contenido de server_private.key>
SaveConfig = false

[Peer]
PublicKey = <client_public.key del portátil>
AllowedIPs = 10.10.10.2/32
```

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl --system

sudo systemctl enable --now wg-quick@wg0
sudo wg show
```

### 5.4 Configuración — cliente (portátil)

```bash
wg genkey | tee client_private.key | wg pubkey > client_public.key
sudo mkdir -p /etc/wireguard
sudo mv client_private.key client_public.key /etc/wireguard/
sudo chmod 600 /etc/wireguard/client_private.key
sudo chmod 644 /etc/wireguard/client_public.key
```

`/etc/wireguard/client.conf`:
```ini
[Interface]
PrivateKey = <client_private.key>
Address = 10.10.10.2/24

[Peer]
PublicKey = <server_public.key del servidor>
Endpoint = <TU_IP_PUBLICA>:51820
AllowedIPs = 10.10.10.0/24
PersistentKeepalive = 25
```

> **Sin línea `DNS =`** — se quitó deliberadamente (ver bitácora #2). Como `AllowedIPs` solo cubre la subred del túnel (no `0.0.0.0/0`), no hace falta DNS especial para "solo llegar al servidor".

Uso a demanda (no como servicio permanente — no tiene sentido mantener el túnel siempre activo desde un portátil):
```bash
sudo systemctl disable wg-quick@client   # asegurar que no arranca solo

sudo wg-quick up client      # conectar cuando se necesite
sudo wg-quick down client    # desconectar
```

---

## 9. Chuleta — activar la VPN en el cliente

Para no repetir el comando completo cada vez, alias en `~/.bashrc` / `~/.zshrc` del cliente (portátil):

```bash
alias vpn-on='sudo wg-quick up client'
alias vpn-off='sudo wg-quick down client'
alias vpn-status='sudo wg show'
```

```bash
source ~/.zshrc   # o ~/.bashrc, según la shell
```

Uso diario:
```bash
vpn-on        # conectar antes de necesitar acceso al servidor (SSH, Cockpit, admin)
vpn-status    # confirmar handshake reciente y tráfico
vpn-off       # desconectar al terminar
```

**Mantenimiento necesario si la IP pública de casa cambia** (es dinámica — resuelto de forma permanente cuando se configure DuckDNS; consulta los [pendientes](../operations/roadmap.md)):
```bash
# en el servidor
curl -4 ifconfig.me

# comparar con lo que tiene el cliente
grep Endpoint /etc/wireguard/client.conf
```
Si no coinciden, actualizar `Endpoint =` en el cliente con la IP nueva.

---
