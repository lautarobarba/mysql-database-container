# MySQL Database Container

## Configuración

El proyecto **no crea ninguna base de datos ni usuario por defecto**. La única configuración es el puerto y la contraseña del usuario `root`.

```bash
$ cp .env.example .env
$ nano .env
```

| Variable           | Descripción                            |
| ------------------ | -------------------------------------- |
| `DB_PORT`          | Puerto del host donde se expone MySQL. |
| `DB_ROOT_PASSWORD` | Contraseña del usuario `root`.         |

> `DB_ROOT_PASSWORD` sólo se aplica la **primera vez** que se levanta el contenedor (cuando **/database/data** está vacío). Cambiarla después en el _.env_ no modifica la contraseña real de `root`; para eso ver [Resetear la contraseña de root](#resetear-la-contraseña-de-root).

## Iniciar

```bash
$ docker compose up -d
```

_Quitando la opción *-d* se ven los logs del contenedor._

## Detener

```bash
$ # Si estan corriendo con logs visibles
$ #     detener con Ctr+C
$ docker compose down
```

# Database (MySQL 9.7 LTS)

Toda la base de datos queda guardada en **/database/data**

## Conectarse como root

El usuario `root` es **sólo para administrar** este servicio (crear bases de datos, usuarios, backups). Ninguna aplicación debería conectarse con `root`.

```bash
$ docker compose exec -it mysql_db mysql -uroot -p
Enter password: # DB_ROOT_PASSWORD
```

Todos los comandos SQL de las secciones siguientes se ejecutan desde esta consola (`mysql>`).

## Agregar una base de datos para un nuevo proyecto

Cada proyecto/aplicación que use este MySQL tiene que tener **su propia base de datos y su propio usuario**, con permisos únicamente sobre esa base de datos. Repetir estos pasos por cada proyecto nuevo.

Reemplazar `mi_proyecto_db`, `mi_proyecto_user` y `una_password_segura` por los valores del proyecto.

### 1. Crear la base de datos

```sql
mysql> CREATE DATABASE mi_proyecto_db CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

### 2. Crear el usuario

`'%'` permite que el usuario se conecte desde cualquier host (necesario para conectarse desde fuera del contenedor).

```sql
mysql> CREATE USER 'mi_proyecto_user'@'%' IDENTIFIED BY 'una_password_segura';
```

### 3. Darle todos los permisos sobre esa base de datos

```sql
mysql> GRANT ALL PRIVILEGES ON mi_proyecto_db.* TO 'mi_proyecto_user'@'%';
```

_No hace falta `FLUSH PRIVILEGES`: `GRANT` aplica los cambios inmediatamente._

### 4. Verificar

```sql
mysql> SHOW GRANTS FOR 'mi_proyecto_user'@'%';
```

Probar la conexión con el nuevo usuario:

```bash
$ docker compose exec -it mysql_db mysql -umi_proyecto_user -p mi_proyecto_db
Enter password: # una_password_segura
```

Desde la aplicación (o desde el host) los datos de conexión quedan:

| Dato          | Valor                                     |
| ------------- | ----------------------------------------- |
| Host          | IP del servidor (`127.0.0.1` si es local) |
| Puerto        | `DB_PORT` del _.env_                      |
| Base de datos | `mi_proyecto_db`                          |
| Usuario       | `mi_proyecto_user`                        |
| Contraseña    | `una_password_segura`                     |

> En MySQL 9.x se eliminó el plugin `mysql_native_password`; los usuarios usan `caching_sha2_password`. Si una aplicación no conecta, verificar que su driver/cliente MySQL esté actualizado.

## Listar bases de datos

```sql
mysql> SHOW DATABASES;
```

_`information_schema`, `mysql`, `performance_schema` y `sys` son bases de datos internas de MySQL._

## Listar usuarios

```sql
mysql> SELECT user, host, plugin, account_locked FROM mysql.user ORDER BY user;
```

_`mysql.infoschema`, `mysql.session` y `mysql.sys` son usuarios internos de MySQL._

Para ver a qué bases de datos tiene acceso un usuario:

```sql
mysql> SHOW GRANTS FOR 'mi_proyecto_user'@'%';
```

## Resetear o actualizar la contraseña de un usuario

Conectado como `root`:

```sql
mysql> ALTER USER 'mi_proyecto_user'@'%' IDENTIFIED BY 'nueva_password';
```

Recordar actualizar la contraseña en la configuración de la aplicación que use ese usuario.

## Eliminar una base de datos y su usuario

**¡Cuidado! Esto borra los datos definitivamente.** Hacer un backup antes.

```sql
mysql> DROP DATABASE mi_proyecto_db;
mysql> DROP USER 'mi_proyecto_user'@'%';
```

## Resetear la contraseña de root

Si se perdió la contraseña de `root` no es posible conectarse para hacer `ALTER USER`. En ese caso se levanta MySQL una única vez con un archivo que cambia la contraseña al iniciar.

```bash
$ # 1. Detener el servicio
$ docker compose down

$ # 2. Crear el archivo con la nueva contraseña (en la carpeta del proyecto)
$ cat > reset-root.sql <<'SQL'
ALTER USER 'root'@'localhost' IDENTIFIED BY 'nueva_password_root';
ALTER USER 'root'@'%' IDENTIFIED BY 'nueva_password_root';
SQL

$ # 3. Iniciar MySQL ejecutando ese archivo.
$ #    Esperar a ver "ready for connections" y detener con Ctrl+C
$ docker compose run --rm -v ./reset-root.sql:/reset-root.sql:ro mysql_db mysqld --init-file=/reset-root.sql

$ # 4. Borrar el archivo (contiene la contraseña en texto plano)
$ rm reset-root.sql

$ # 5. Actualizar DB_ROOT_PASSWORD en el .env y levantar el servicio
$ nano .env
$ docker compose up -d
```

## Backups

Los backups se hacen **por base de datos** con el usuario `root`. Los archivos se guardan en **/backups**.

Para backups automáticos (a otro servidor con rsync o a AWS S3 Glacier Deep Archive), cómo descargarlos y cómo restaurarlos en una instancia nueva, ver **[BACKUPS.md](BACKUPS.md)**.

### Backup manual

Reemplazar `mi_proyecto_db` por la base de datos a respaldar.

```bash
$ docker compose exec -T mysql_db sh -c 'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" --single-transaction --routines --triggers --events --set-gtid-purged=OFF mi_proyecto_db' | gzip > mi_proyecto_db-$(date +%Y%m%d-%H%M).sql.gz
```

### Restaurar backup

Ver [Restaurar en una instancia nueva](BACKUPS.md#restaurar-en-una-instancia-nueva).
