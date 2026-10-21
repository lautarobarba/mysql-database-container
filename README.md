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

Se creó un volumen para guardar los _backups_ en **/backups**. Los backups se hacen **por base de datos** con el usuario `root`.

### Realizar backup

Reemplazar `mi_proyecto_db` por la base de datos a respaldar.

```bash
$ docker compose exec -it mysql_db bash
root@container:$ mysqldump -uroot -p mi_proyecto_db > /backups/mi_proyecto_db$(date "+%Y%m%d-%H_%M")hs.sql
Enter password: # DB_ROOT_PASSWORD
root@container:$ exit
```

### Restaurar backup

Para restaurar el backup tenemos que entrar a una shell del contenedor y restaurar el backup que se encuentra en el volumen montado.

1. Hay que asegurarse de tener el backup en la carpeta **/backups**.
2. La base de datos tiene que existir (ver [Crear la base de datos](#1-crear-la-base-de-datos)).

```bash
$ cd backups
$ sudo unzip NOMBRE_BACKUP.zip
$ cd ..
$ docker compose exec -it mysql_db bash
root@container:$ mysql -uroot -p mi_proyecto_db < /backups/NOMBRE_BACKUP.sql
Enter password: # DB_ROOT_PASSWORD
root@container:$ exit
```

## Programar backups automáticos (CRON)

### Prerrequisitos

En el servidor de destino tiene que estar configurado ssh para aceptar autenticación con Public Keys.

```bash
$ #------------------------------------------------------------------------------
$ # ¡¡¡En el servidor de destino!!!
$ # Preparar un usuario y ruta de backups
$ useradd -m backup_user -s /bin/bash
$ passwd backup_user
$ # Tendremos un usuario backup_user para realizar los backups
$ #   y un dir /home/backup_user para guardarlos
$
$ # Configurar autenticación de sshd
$ sudo nano /etc/ssh/sshd_config
$ # Descomentar la linea que tiene: PubkeyAuthentication yes
$ sudo systemctl restart sshd.service
$ #------------------------------------------------------------------------------
$ # ¡¡¡En el servidor de origen del backup (USUARIO ROOT)!!!
root$ # Generar las claves del usuario que va a ejecutar el script en este equipo
root$ ssh-keygen
root$ ssh-copy-id backup_user@SERVIDOR_DESTINO
```

### Programar tarea

```bash
$ # En el servidor de origen del backup
$ cp backup_cron.sh.example backup_cron.sh
$ chmod +x backup_cron.sh
$ # Configurar DB_NAME, DB_ROOT_PASSWORD y el destino (SSH_DEST_*)
$ #   Para respaldar varias bases de datos, hacer una copia del script por cada una
$ nano backup_cron.sh
$ # Programar la tarea con cron (USUARIO ROOT)
root$ crontab -e
root$ # En cron:
			# Para probar un backup cada 5 minutos
			*/5 * * * * bash /ruta_al_script/backup_cron.sh

			# Para dejar el backup 1 vez por dia
			0 0 * * * bash /ruta_al_script/backup_cron.sh
```
