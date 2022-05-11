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

**CONTAINER_NAME**: está en el archivo _.env_

```bash
$ docker container exec -it CONTAINER_NAME bash
```

## Backups

Se creó un volumen para guardar los **backups**.

### Realizar backup

Para hacer el backup tenemos que entrar a una shell del contenedor y generar el archivo de backup en la carpeta donde esta montado el volumen.
Usar los datos configurados previamente en .env

**CONTAINER_NAME**: está en el archivo _.env_

**POSTGRES_USER**: está en el archivo _.env_

**POSTGRES_DB**: está en el archivo _.env_

```bash
$ docker container exec -it CONTAINER_NAME bash
root# pg_dump -U POSTGRES_USER POSTGRES_DB > backups/${POSTGRES_DB}$(date "+%Y%m%d-%H:%M").sql
root# exit
```

### Restaurar backup

Para restaurar el backup tenemos que entrar a una shell del contenedor y restaurar el backup que se encuentra en el volumen montado.

1. Hay que asegurarse de tener el backup en la carpeta local backups.
2. Detener el contenedor de docker si esta corriendo.
3. Eliminar la carpeta database que tiene la base de datos actual.
4. Volver a iniciar el contenedor.

**CONTAINER_NAME**: está en el archivo _.env_

**POSTGRES_USER**: está en el archivo _.env_

**NOMBRE_DEL_BACKUP**: archivo ubicado en _backups_

```bash
$ docker compose down
$ sudo rm -r database
$ docker compose up -d
$ docker container exec -it CONTAINER_NAME bash
root# cd backups
root# psql -U POSTGRES_USER
usuario_postgres=# \conninfo
usuario_postgres=# \i NOMBRE_DEL_BACKUP.sql
usuario_postgres=# \q
root# exit
```
