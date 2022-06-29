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

Toda la base de datos queda guardada en **/database/data**

Para conectarse a una terminal del contenedor (sólo para debug)

**CONTAINER_NAME**: está en el archivo _.env_

```bash
$ docker container exec -it CONTAINER_NAME bash
```

## Backups

Se creó un volumen para guardar los _backups_ en **/backups**.

### Realizar backup

Para hacer el backup tenemos que entrar a una shell del contenedor y generar el archivo de backup en la carpeta donde esta montado el volumen.
Usar los datos configurados previamente en **.env**.

**CONTAINER_NAME**: está en el archivo _.env_

**MYSQL_DATABASE**: está en el archivo _.env_

**MYSQL_ROOT_PASSWORD**: está en el archivo _.env_

```bash
$ docker container exec -it CONTAINER_NAME bash
root@container:$ mysqldump -p ${MYSQL_DATABASE} > /backups/${MYSQL_DATABASE}$(date "+%Y%m%d-%H_%M")hs.sql
root@container:$ # MYSQL_ROOT_PASSWORD
root@container:$ exit
```

### Restaurar backup

Para restaurar el backup tenemos que entrar a una shell del contenedor y restaurar el backup que se encuentra en el volumen montado.

1. Hay que asegurarse de tener el backup en la carpeta **/backups**.

**CONTAINER_NAME**: está en el archivo _.env_

**MYSQL_ROOT_PASSWORD**: está en el archivo _.env_

```bash
$ cd backups
$ sudo unzip NOMBRE_BACKUP.zip
$ cd ..
$ docker container exec -it CONTAINER_NAME bash
root@container:$ mysql -p ${MYSQL_DATABASE} < /backups/NOMBRE_BACKUP.sql
root@container:$ # MYSQL_ROOT_PASSWORD
root@container:$ exit
```

## Programar backups automáticos (CRON)

### Prerrequisitos

En el servidor de destino tiene que estar configurado ssh para aceptar autenticación con Public Keys.

```bash
$ # En el servidor de destino
$ sudo nano /etc/ssh/sshd_config
$ # Descomentar la linea que tiene: PubkeyAuthentication yes
$ sudo systemctl restart sshd.service
$ #------------------------------------------------------------------------------
$ # En el servidor de origen del backup (USUARIO ROOT)
root$ # Generar las claves del usuario que va a ejecutar el script en este equipo
root$ ssh-keygen
root$ ssh-copy-id USUARIO@SERVIDOR_DESTINO
```

### Programar tarea

```bash
$ # En el servidor de origen del backup
$ cp backup_cron.sh.example backup_cron.sh
$ # Configurar el destino. Editar la linea #DEST=usuaro@host:/path
$ nano backup_cron.sh
$ # Programar la tarea con cron (USUARIO ROOT)
root$ crontab -e
root$ # En cron:
			# Para probar un backup cada 5 minutos
			*/5 * * * * /ruta_al_script/backup_cron.sh

			# Para dejar el backup 1 vez por dia
			0 0 * * * /ruta_al_script/backup_cron.sh
```
