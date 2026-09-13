# Backup y restore (procedimiento recomendado)

> Navegación: [índice de documentación](../README.md) · [README del proyecto](../../README.md)

Este documento propone un procedimiento de backup y restore para PostgreSQL y las configuraciones. **Es una recomendación operativa; no afirma que exista una automatización ni que estos backups estén implementados actualmente.** Adapta rutas, usuario y política de retención al entorno real.

## Principios

- Mantener las copias fuera del volumen Docker y, para recuperación ante desastre, fuera del propio servidor.
- No incluir `.env`, claves privadas, certificados privados ni claves WireGuard en el repositorio o en copias sin cifrar.
- Probar periódicamente un restore en una base de datos o servidor aislado; un archivo creado no demuestra que sea restaurable.
- Registrar fecha, origen, versión de PostgreSQL y resultado de la verificación.

## Backup lógico recomendado de PostgreSQL

Ejemplo con formato custom, que permite restaurar selectivamente. Sustituye `<USUARIO_POSTGRES>` y la ruta de destino según tu instalación:

```bash
cd /opt/server
mkdir -p backups/postgres

# Ejecutar con permisos adecuados; el archivo se crea fuera del contenedor.
docker compose exec -T db pg_dump \
  -U <USUARIO_POSTGRES> \
  -d mydb \
  --format=custom \
  > backups/postgres/mydb-$(date +%Y%m%d-%H%M%S).dump

sha256sum backups/postgres/*.dump | tail -n 1
```

El backup debe copiarse después a un destino protegido y con retención definida. No pegues aquí contraseñas ni el contenido de `.env`; usa el mecanismo local de autenticación configurado para PostgreSQL.

## Backup recomendado de configuración

Conservar el `docker-compose.yml`, configuraciones de nginx, código de la aplicación, reglas documentadas y archivos de systemd/firewalld necesarios para reconstruir el servicio. Excluir secretos y material privado:

```bash
cd /opt/server
tar --create --gzip \
  --file backups/config-$(date +%Y%m%d-%H%M%S).tar.gz \
  --exclude='./.env' \
  --exclude='./.env.*' \
  --exclude='./nginx/certs/*.key' \
  --exclude='./logs' \
  docker-compose.yml nginx web app
```

Para WireGuard, conservar por un canal seguro la configuración necesaria y las claves privadas en un gestor de secretos o almacenamiento cifrado; no las añadas a este repositorio. La reconstrucción debe poder hacerse con placeholders hasta inyectar los secretos localmente.

## Restore de PostgreSQL (procedimiento recomendado)

1. Preparar el host, Docker y la configuración local (`.env`) sin publicar sus valores.
2. Levantar únicamente la base de datos y comprobar que está lista:

```bash
cd /opt/server
docker compose up -d db
docker compose ps db
```

3. Para una restauración destructiva, confirmar dos veces el destino y disponer de un backup reciente. Ejemplo sobre una base de datos ya creada:

```bash
cat backups/postgres/<BACKUP>.dump | \
  docker compose exec -T db pg_restore \
    -U <USUARIO_POSTGRES> \
    -d mydb \
    --clean --if-exists --exit-on-error
```

4. Comprobar tablas, permisos y una consulta funcional antes de levantar toda la aplicación. Si se restaura en una base de datos de prueba, usa otro nombre y evita `--clean` sobre producción.
5. Levantar la aplicación y ejecutar las pruebas de humo de [referencia rápida](quick-reference.md).

## Restauración de configuración

Extrae el archivo en una ruta de revisión, compara con la configuración existente y aplica los cambios de forma controlada. Regenera o inyecta secretos desde el almacén local, valida nginx (`nginx -t`), revisa permisos y recrea solo los contenedores afectados. No sobrescribas a ciegas `/etc/wireguard/`, `.env` ni claves privadas.

## Retención y verificación

La frecuencia, retención, cifrado y destino deben decidirse según el valor de los datos y el tiempo de recuperación esperado. Como mínimo, valida que los tamaños no sean cero, conserva hashes y realiza restores de prueba antes de declarar recuperable el servicio.
