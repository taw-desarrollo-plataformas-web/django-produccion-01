# Django en produccion a través de gunicorn y nginx

## Antecedentes
* La presente guía está basada en el tutorial de la plataforma **DigitalOcean** [[enlace - web]](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-20-04) 

## Softaware y tecnologías
* Se necesita un computador con las siguientes características:
	* Sistema operativo GNU/Linux UBUNTU 18.04 o 20.04
	* Lenguaje de programación Python (3.7, 3.8, 3.9)
	* Librerías de pyhton: django, corsheaders, rest_framework, gunicorn; (se pueden instalar a través de pip install). 
	* Servidor Web - nginx
	
## Proceso 

### Parte 1

Se asume que se tiene un proyecto de django funcional a través del servidor propio de la framework (python manage.py runserver)

1. Instalar la librería gunicorn (pip install gunicorn)
2. Agregar la variable **ALLOWED_HOSTS**  con algunas direcciones; en el archivo **settings.py** del proyecto de django; que permitan acceder desde gunicorn y luego desde el servidor web.
```
ALLOWED_HOSTS = ["0.0.0.0", "127.0.0.1"] 	 
```
3. En el archivo **urls.py** del proyecto de django agregar lo siguiente
```
# importar
from django.contrib.staticfiles.urls import staticfiles_urlpatterns

# agregar el siguiente valor a la variable urlpatterns
urlpatterns += staticfiles_urlpatterns()
```
4. Recopilar el contenido estático en al carpeta static (principalmente los archivos de admin)

```
python manage.py collectstatic
```

5. Levantar o iniciar el proyecto con gunicorn, desde la carpeta raíz del mismo, a través del siguiente comando:

```
gunicorn --bind 0.0.0.0:8000 proyectoUno.wsgi
```
Donde, **proyectoUno** es el nombre del proyecto. En el navegador, se debe observar el proyecto funcionando. Consejo, si no es posible visualizar el proyecto, debe revisar los errores; cuando se superen los mismos, pasar a la siguiente parte.

5. Terminar la ejecución a través de **control+c**

### Parte 2

Es momento de iniciar el proceso de enlazar el servidor nginx a través de gunicorn con el proyecto de django.

1) Agregar un servicio en el sistema operativo; mismo que será encargado de levantar el proyecto de django mediante gunicorn. Luego el servicio será usado por nginx.

2) En el directorio **/etc/systemd/system/** agregar un archivo con la siguiente extensión y estructura. Se debe usar **sudo** para acceder y crear el archivo.

2.1. Nombre del archivo **proyecto01.service** Donde proyecto01 es un nombre cualquiera.
2.2. En el archivo agregar la siguiente información
```

[Unit]
# metadatos necesarios
Description=gunicorn daemon
After=network.target

[Service]
# usuario del sistema operativo que ejecutará el procesos
User=usuario-sistema-operativo
# el grupo del sistema operativo que permite la comunicación a desde el servidor web-nginx con gunicorn. No se debe cambiar el valor
Group=www-data

# a través de la variable WorkingDirectory se indicar la dirección absoluta del proyecto de Django
WorkingDirectory=/home/usuario-sistema/carpeta/proyectos/nombre-proyecto
# se indicar el path de python
# Ejemplo 1 /usr/bin/python3.9
# Ejemplo 2 (con el uso de entornos virtuales) /home/usuario/entornos/entorno01/bin
Environment="PATH=agregar-path-python"

# Detallar el comando para iniciar el servicio
ExecStart=path-python/bin/gunicorn --workers 3 --bind unix:application.sock -m 007 proyectoDjango.wsgi:application

# Donde: aplicacion.sock es el nombre del archivo que se debe crear en el directorio del proyecto; proyectoDjango el nombre del proyecto que se intenta vincular con nginx.


[Install]
# esta sección será usada para indicar que el servicio puede empezar cuando se inicie el sistema operativo. Se sugiere no cambiar el valor dado.
WantedBy=multi-user.target
```
3) Iniciar y habilitar el proceso a través de los siguiente comando:
```
sudo systemctl start proyecto01
sudo systemctl enable proyecto01
```
Donde **proyecto01**, es el nombre que se le asigno al archivo creado con extensión **service**

Verificar que todo esté en orden con el servicio, usar el comando:
```
sudo systemctl status proyecto01
```

4) Este paso es importante, se debe verificar que el archivo .sock esté creado en el directorio del proyecto.
Ejemplo:
![https://github.com/taw-desarrollo-plataformas-web/django-produccion-01/raw/main/imgs/img-app-sock.png](https://github.com/taw-desarrollo-plataformas-web/django-produccion-01/raw/main/imgs/img-app-sock.png  "img-sock")

5) Si todo marcha bien, se puede pasar a la siguiente parte.

### Parte 3
* Configuración del servidor nginx. Se asume que tiene instalado nginx en el sistema operativo.
* Los comandos para iniciar, reiniciar, parar y verificar el servicio son:
	* sudo service nginx start
	* sudo service nginx stop
	* sudo service nginx restart
	* sudo service nginx status
	
#### Opción 1
1) Crear un archivo **sites-available** de nginx; la ruta de acceso es: /etc/nginx/sites-available/. Se debe ingresar con permisos de administrador (sudo).
```
Ejemplo: para crear un archivo llamado proyecto01, se puede usar el siguiente comando desde la terminal.

sudo touch /etc/nginx/sites-available/proyecto01

```
2) En el archivo se puede usar la siguiente estructura
```
server {
    listen 81;
    server_name localhost;
    
    location / {
        include proxy_params;
        proxy_pass http://unix:/ruta/al/archivo/sock/application.sock;
    }

    
    location /static/ {
        root /ruta/a/la/carpeta/staticos/del/proyecto-django/static/;
    }

}

```
3) Crear un enlace simbólico del archivo creado en el directorio sites-available.

```
sudo ln -s /etc/nginx/sites-available/proyecto01 /etc/nginx/sites-enabled
```
4) Iniciar o reiniciar el servicio de nginx.

5) Si todo marcha bien, en un navegador con las siguiente direcciones se debe deplegar el proyecto a través de nginx:

* http://localhost:81
* http://0.0.0.0:81
* http://127.0.0.0:81

![](https://github.com/taw-desarrollo-plataformas-web/django-produccion-01/raw/main/imgs/proyecto-0.png) 

6) Verificar que el proyecto funcione.

#### Opción 2

1) Crear un archivo **sites-available** de nginx; la ruta de acceso es: /etc/nginx/sites-available/. Se debe ingresar con permisos de administrador (sudo).
```
Ejemplo: para crear un archivo llamado proyecto01, se puede usar el siguiente comando desde la terminal.

sudo touch /etc/nginx/sites-available/proyecto01

```
2) En el archivo se puede usar la siguiente estructura
```
server {
    listen 81;
    server_name localhost;
    
    location /proyecto1 {
       rewrite /proyecto1/(.*) /$1 break;
        include proxy_params;
        proxy_pass http://unix:/ruta/al/archivo/sock/application.sock;
    }

    
    location /proyecto1/static/ {
        rewrite /proyecto1/static/(.*) /$1 break;
        root /ruta/a/la/carpeta/staticos/del/proyecto-django/static/;
    }

}
```
3) Crear un enlace simbólico del archivo creado en el directorio sites-available.

```
sudo ln -s /etc/nginx/sites-available/proyecto01 /etc/nginx/sites-enabled
```

4) En el archivo settings.py del proyecto realizar la siguiente modificaciones.
```
# modificar esta variable solo en producción, donde proyecto1, es el subdirectorio usado en el archivo de nginx
STATIC_URL = '/proyecto1/static/'

# agregar y usar esta variable solo en producción, donde proyecto1, es el subdirectorio usado en el archivo de nginx
FORCE_SCRIPT_NAME = '/proyecto1'

```

4) Iniciar o reiniciar el servicio de nginx.

5) Si todo marcha bien, en un navegador con las siguiente direcciones se debe deplegar el proyecto a través de nginx:

* http://localhost:81/proyecto1
* http://0.0.0.0:81/proyecto1
* http://127.0.0.0:81/proyecto1

![](https://github.com/taw-desarrollo-plataformas-web/django-produccion-01/raw/main/imgs/proyecto-01.png) 

6) Verificar que el proyecto funcione.

