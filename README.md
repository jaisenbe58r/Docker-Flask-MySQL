# Código para crear una aplicación Docker con Flask y tutorial de MySQL

 Basado en [blog post about creating a flask-mysql app with docker](https://stavshamir.github.io/python/dockerizing-a-flask-mysql-app-with-docker-compose/)


En este Post vamos a tratar de tomar una aplicación web simple existente basada en ```Flask``` y ```MySQL``` y desplegarla con ```Docker``` y ```docker-compose``` en ```Digital Ocean```.

Se considera una mejor práctica que un contenedor tenga solo una responsabilidad y un proceso, por lo que para nuestra aplicación necesitaremos al menos dos contenedores: uno para ejecutar la aplicación en sí y otro para ejecutar la base de datos. Coordinaremos estos dos contenedores con ```docker-compose```. 

Todo el código utilizado en este Post está disponible aquí:

[jaisenbe58r/Docker-Flask-MySQL](https://github.com/jaisenbe58r/Docker-Flask-MySQL)

 Comenzamos con el siguiente diseño del proyecto:
```
Proyect/
├── app
│   └── app.py
└── db
    └── init.sql
```

- ```app.py```: contiene la aplicación Flask que se conecta a la base de datos y expone un punto final de la API REST
- ```init.sql```: un script SQL para inicializar la base de datos antes de que se ejecute la aplicación por primera vez.

### Creando una imagen de Docker para nuestra aplicación.

Necesitamos crear un ```Dockerfile``` en el directorio de la aplicación, este archivo contiene un conjunto de instrucciones que describen nuestra imagen deseada y permiten su compilación automática.

```Dockerfile
# Dockerfile-flask
#  Python 3 image
FROM python:3
RUN apt -qq -y update \
	&& apt -qq -y upgrade
# Set an environment variable with the directory
# where we'll be running the app
ENV APP /app
# Create the directory and instruct Docker to operate
# from there from now on
RUN mkdir $APP
WORKDIR $APP
# Expose the port uWSGI will listen on
EXPOSE 56734
# Copy the requirements file in order to install
# Python dependencies
COPY requirements.txt .
# Install Python dependencies
RUN pip install -r requirements.txt
# We copy the rest of the codebase into the image
COPY . .
# Finally, we run uWSGI with the ini file we
# created earlier
CMD [ "uwsgi", "--ini", "app.ini" ]
```

Lo que hace esto es:
- la imagen de Docker se creará a partir de una imagen existente, ```python:3```, correspondiente a una imagen ligera de _python3_.
- seleccionar el puerto de trabajo, en este caso el ```56734```, primero asegúrese de disponer de un puerto abierto para usarlo en la configuración. Para verificar si hay un puerto libre, ejecute el siguiente comando:

    ```cmd
    sudo nc localhost 56734 < /dev/null; echo $?
    ```

    Si el resultado del comando anterior es ```1```, el puerto estará libre y podrá utilizarse. De lo contrario, deberá seleccionar un puerto diferente y repetir el procedimiento.

- Se copia el archivo ```requirements.txt``` al contenedor para que pueda ejecutarse y luego se analiza el archivo ```requirements.txt``` para instalar las dependencias especificadas. También copiaremos todo el directorio de trabajo del repositorio dentro de la imagen para posteriormente compartarlo como volumen externo.

- Se crea un directorio de trabajo en el que se copia todo el repositorio.

Necesitamos que nuestras dependencias (```Flask``` y ```mysql-connector```) se instalen y entreguen con la imagen, por lo que debemos crear el archivo ```requirements.txt``` antes mencionado:
```
Flask
mysql-connector
```

En el caso de la imagen de MySQL, no hará falta crear un ```Dockerfile```puesto que utilizaremos la imagen descargada directamente del ```Docker Hub```.


El archivo ```app.ini``` contendrá las configuraciones de uWSGI para nuestra aplicación. uWSGI es una opción de implementación para Nginx que es tanto un protocolo como un servidor de aplicaciones. El servidor de aplicaciones puede proporcionar los protocolos uWSGI, FastCGI y HTTP.

```ini
[uwsgi]
protocol = uwsgi
; This is the name of our Python file
; minus the file extension
module = app
; This is the name of the variable
; in our script that will be called
callable = app
master = true
; Set uWSGI to start up 5 workers
processes = 5
; We use the port 56734 which we will
; then expose on our Dockerfile
socket = 0.0.0.0:56734
vacuum = true
die-on-term = true
```

Este código define el módulo desde el que se proporcionará la aplicación de Flask, en este caso es el archivo ```app.py```. La opción _callable_ indica a uWSGI que use la instancia de app exportada por la aplicación principal. La opción master permite que su aplicación siga ejecutándose, de modo que haya poco tiempo de inactividad incluso cuando se vuelva a cargar toda la aplicación.



### Construcción imagen de Nginx en Docker

Antes de implementar la construcción de la imagen del contenedor Nginx, crearemos nuestro archivo de configuración que le dirá a Nginx cómo enrutar el tráfico a uWSGI en nuestro otro contenedor. El archivo ```app.conf``` reemplazará el ```/etc/nginx/conf.d/default.conf``` que el contenedor Nginx incluye implícitamente. [Lea más sobre los archivos conf de Nginx aquí.](http://nginx.org/en/docs/beginners_guide.html)

```conf
server {
    listen 80;
    root /usr/share/nginx/html;
    location / { try_files $uri @app; }
    location @app {
        include uwsgi_params;
        uwsgi_pass flask:56734;
    }
}
```

La línea se uwsgi_pass ```flask:56734``` está utilizando flask como host para enrutar el tráfico. Esto se debe a que configuraremos ```docker-compose``` para conectar nuestros contenedores Flask y Nginx a través del ```flask``` como nombre de host.


Nuestro Dockerfile para Nginx simplemente heredará la última imagen de [Nginx del registro de Docker](https://hub.docker.com/_/nginx/), eliminará el archivo de configuración predeterminado y agregará el archivo de configuración que acabamos de crear durante la compilación. Nombraremos el archivo ```Dockerfile-nginx```.

```Dockerfile
# Dockerfile-nginx
FROM nginx:latest
# Nginx will listen on this port
EXPOSE 80
# Remove the default config file that
# /etc/nginx/nginx.conf includes
RUN rm /etc/nginx/conf.d/default.conf
# We copy the requirements file in order to install
# Python dependencies
COPY app.conf /etc/nginx/conf.d
```

### Creando el orquestador de contenedores docker-compose.yml

Se crea el archivo ```docker-compose.yml``` en el directorio raíz de nuestro proyecto:
```yml
version: "2"
services:
  app:
    build: ./app
    links:
      - db
    ports:
      - "5000:5000"
```
Estamos utilizando dos servicios, uno es un contenedor que expone la API (Flask) y otro contiene la base de datos MySQL (db).

- ```build```: especifica el directorio que contiene el Dockerfile que contiene las instrucciones para construir este servicio
- ```links```: vincula este servicio a otro contenedor. Esto también nos permitirá usar el nombre del servicio en lugar de tener que buscar la ip del contenedor de la base de datos, y expresar una dependencia que determinará el orden de inicio del contenedor.
- ```ports```: asignación de <Host>: <Container> puertos.
```yml
  db:
    image: mysql:5.7
    ports:
      - "32000:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
    volumes:
      - ./db:/docker-entrypoint-initdb.d/:ro
```
- ```image```: En lugar de escribir un nuevo Dockerfile, usamos una imagen existente de un repositorio. Es importante especificar la versión, si su cliente mysql instalado no es de la misma versión, pueden ocurrir problemas.
- ```environment```: Agregación de variables de entorno. La variable especificada es necesaria para esta imagen. Como sugiere su nombre, configura la contraseña para el usuario raíz de MySQL en este contenedor. Aquí se especifican más variables.
- ```ports```: Como ya tengo una instancia de mysql en ejecución en mi host usando este puerto, lo estoy mapeando a uno diferente. 
- ```volumes```: Dado que queremos que el contenedor se inicialice con nuestro esquema, conectamos el directorio que contiene nuestro script ```init.sql``` al punto de entrada para este contenedor, que según la especificación de la imagen ejecuta todos los scripts ```.sql``` en el directorio dado.

A continuación se muestra el código que se conecta a la base de datos (app.py):
```python
config = {
        'user': 'root',
        'password': 'root',
        'host': 'db',
        'port': '3306',
        'database': 'knights'
    }
connection = mysql.connector.connect(**config)
```

 Se conecta como ```root``` con la contraseña configurada en el archivo ```docker-compose```. Observe que definimos explícitamente el host (que es localhost por defecto) ya que el servicio SQL está en un contenedor diferente al que ejecuta este código. Podemos (y debemos) usar el nombre 'db' ya que este es el nombre del servicio que definimos y vinculamos anteriormente. El puerto es ```3306``` y no ```32000``` ya que este código no se está ejecutando en el host.


```yml
version: '3'
services:
  flask:
    image: webapp-flask
    build:
      context: .
      dockerfile: Dockerfile-flask
    volumes:
      - "./:/app"
  nginx:
    image: webapp-nginx
    build:
      context: .
      dockerfile: Dockerfile-nginx
    ports:
      - 56734:80
    depends_on:
      - flask
```

Las claves debajo ```services:``` definen los nombres de cada uno de nuestros servicios, contenedores Docker. De ahí, flask y nginx sean los nombres de nuestros dos contenedores.

```yml
imagen: webapp-flask
```

Esta línea especifica qué nombre tendrá nuestra imagen después de crearla. ```docker-compose``` construirá la imagen la primera vez que la lancemos y mantendrá un registro del nombre para todos los lanzamientos futuros.

```yml
build:
  context: .
  dockerfile: Dockerfile-flask
```

```context``` le dice al motor de Docker que solo use archivos en el directorio actual para construir la imagen. En segundo lugar ```dockerfile``` le está diciendo al motor que busque el archivo ```Dockerfile-flask``` para construir la imagen de flask.

```yml
  volumes:
  - "./:/app"
```

Aquí simplemente estamos diciendo a ```docker-compose``` que monte nuestra carpeta actual en el directorio ```/app``` en el contenedor cuando se activa. De esta manera, a medida que hacemos cambios en la aplicación, no tendremos que seguir construyendo la imagen a menos que sea un cambio importante, como una dependencia de un módulo de software. En este caso en el ```Dockerfile```de la imagen de flask ya habíamos preparado este directorio de trabajo.

Para el servicio ```nginx```, hay algunas cosas a tener en cuenta:

```yml
ports: 
  - 56734:80
```

Esta pequeña sección le dice a ```docker-compose```que asigne el puerto ```56734``` de su máquina local al puerto ```80``` del contenedor Nginx (puerto que Nginx sirve por defecto).

```yml
depends_on:
  - flask
```

Tal y como se ha implementado en el ```app.conf```, enrutamos el tráfico de Nginx a uWSGI y viceversa enviando datos a través del ```flask``` como nombre de host. Lo que hace esta sección es crear un nombre de host virtual ```flask```en nuestro contenedor ```nginx``` y configurar la red para que podamos enrutar los datos entrantes a nuestra aplicación uWSGI que vive en un contenedor diferente. La ```depends_on``` directiva también espera hasta que el contenedor ```flask```esté en un estado funcional antes de lanzar el contenedor ```nginx```, lo que evita tener un escenario en el que Nginx falla si el host ```flask``` no responde.

### Actualizar la aplicación

A veces, deberá realizar en la aplicación cambios que pueden incluir instalar nuevos requisitos, actualizar el contenedor de Docker o aplicar modificaciones vinculadas al HTML y a la lógica. A lo largo de esta sección, configurará touch-reload para realizar estos cambios sin necesidad de reiniciar el contenedor de Docker.

```Autoreloading``` de Python controla el sistema completo de archivos en busca de cambios y actualiza la aplicación cuando detecta uno. No se aconseja el uso de _autoreloading_ en producción porque puede llegar a utilizar muchos recursos de forma muy rápida. En este paso, utilizará touch-reload para realizar la verificación en busca de cambios en un archivo concreto y volver a cargarlo cuando se actualice o sustituya.

Para implementar esto, abra el archivo uwsgi.ini e incorpore la siguiente linea de código:

```python
touch-reload = /app/uwsgi.ini
```

Esto especifica un archivo que se modificará para activar una recarga completa de la aplicación.

A continuación, si hace una modificación en cualquier _template_ y abre la página de inicio de su aplicación en http://```your-domain:56733``` observará que los cambios no se reflejan. Esto se debe a que la condición para volver a cargar es un cambio en el archivo uwsgi.ini. Para volver a cargar la aplicación, use touch a fin de activar la condición:

```shell
sudo touch uwsgi.ini
```

