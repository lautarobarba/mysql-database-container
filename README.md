# My Database Container
MySQL with Docker

# Dependencias 

- Docker

# Ejecución

Copiar el archivo env.example o renombrarlo a env
```bash
$ cp env.example env
```

Editar el contenido de env
```bash
$ nano env
```

Iniciar los contenedores para producción (logs ocultos)
```bash
$ docker-compose up -d
```

Iniciar los contenedores para desarrollo (logs visibles)
```bash
$ docker-compose up
```

## Detener ejecución
```bash
$ # Si estan corriendo para desarrollo 
$ #     detener con Ctr+C
$ docker-compose down
```


### Notas
Antes de ejecutar asegurarse de tener docker corriendo
```bash
$ sudo systemctl status docker.service 
# O el administrador de servicios que corresponda para ese sistema
```

Si no esta *active (running)* iniciarlo
```bash
$ sudo systemctl start docker.service 
$ sudo systemctl enable docker.service 
# O el administrador de servicios que corresponda para el sistema
```

## Database (MySQL)
Toda la base de datos queda guardada en **/database/data**

(solo para debug) Para conectarse a una terminal del contenedor
```bash
$ docker container exec -it db_mysql bash
```