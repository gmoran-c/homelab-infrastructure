# Operaciones: referencia rápida

> Navegación: [índice de documentación](../README.md) · [README del proyecto](../../README.md)

Chuleta de comandos habituales para Docker, WireGuard, fail2ban y pruebas de humo.

## 11. Comandos de referencia rápida

### Docker

```bash
cd /opt/server
docker compose up -d
docker compose ps -a
docker compose logs -f nginx
docker compose exec nginx nginx -t
docker compose exec nginx nginx -s reload
docker compose build app && docker compose up -d app
docker compose down            # mantiene volúmenes
docker compose down -v         # borra también volúmenes (¡incluye datos de postgres!)
```

### WireGuard

```bash
# Servidor
sudo wg show
sudo systemctl status wg-quick@wg0
sudo systemctl restart wg-quick@wg0

# Cliente
sudo wg-quick up client
sudo wg-quick down client
sudo wg show

# Diagnóstico (aislar router vs. WireGuard)
sudo systemctl stop wg-quick@wg0
mkdir -p logs
sudo timeout 60 nc -ul 51820 > logs/nc_test.log 2>&1 &
cat logs/nc_test.log
sudo systemctl start wg-quick@wg0
# desde el cliente, en la ventana de 60s:
echo "prueba" | nc -u -w1 <IP_PUBLICA> 51820
```

### fail2ban

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo fail2ban-client status nginx-limit-req
sudo fail2ban-client set nginx-limit-req unbanip <IP>   # desbanear manualmente
sudo tail -f /var/log/fail2ban.log                       # log real (no journalctl -u)
```

### Pruebas de humo

```bash
docker compose ps
curl -I http://localhost              # debe redirigir (301) a https
curl -k -I https://localhost          # -k: certificado autofirmado
curl -k https://localhost/api/

ping 10.10.10.1                        # desde fuera de la red, con VPN activa
curl -k https://10.10.10.1
```

---
