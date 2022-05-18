# MySQL Database Container

## Dependencias

- Docker

## Configuración

```bash
$ cp .env.example .env
$ nano .env
```

## Iniciar

```bash
$ docker compose up -d prod
```

_Quitando la opción *-d* se ven los logs del contenedor._

## Detener

```bash
$ # Si estan corriendo con logs visibles
$ #     detener con Ctr+C
$ docker compose down
```

# Database (MySQL)

Toda la base de datos queda guardada en **/database/data**.

Para conectarse a una terminal del contenedor (sólo para debug)

```bash
$ docker container exec -it prod bash
```

## Backups

Se creó un volumen para guardar los _backups_ en **/backups**.

### Realizar backup

Para hacer el backup tenemos que entrar a una shell del contenedor y generar el archivo de backup en la carpeta donde esta montado el volumen.
Usar los datos configurados previamente en **.env**.

**MYSQL_DATABASE**: está en el archivo _.env_

**MYSQL_ROOT_PASSWORD**: está en el archivo _.env_

```bash
$ docker container exec -it prod bash
root@container:$ mysqldump -p ${MYSQL_DATABASE} > /backups/${MYSQL_DATABASE}$(date "+%Y%m%d-%H_%M")hs.sql
root@container:$ # MYSQL_ROOT_PASSWORD
root@container:$ exit
```

### Restaurar backup

Para restaurar el backup tenemos que entrar a una shell del contenedor y restaurar el backup que se encuentra en el volumen montado.

1. Hay que asegurarse de tener el backup en la carpeta **/backups**.

**MYSQL_DATABASE**: está en el archivo _.env_

**NOMBRE_DEL_BACKUP**: archivo ubicado en _backups_

**MYSQL_ROOT_PASSWORD**: está en el archivo _.env_

```bash
$ docker container exec -it prod bash
root@container:$ mysql -p ${MYSQL_DATABASE} < /backups/NOMBRE_BACKUP.sql
root@container:$ # MYSQL_ROOT_PASSWORD
root@container:$ exit
```
