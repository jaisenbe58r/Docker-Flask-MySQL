# Código para crear una aplicación Docker con Flask y tutorial de MySQL

Refer to [blog post about creating a flask-mysql app with docker](https://stavshamir.github.io/python/dockerizing-a-flask-mysql-app-with-docker-compose/)


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

```python

FROM python:3.8

EXPOSE 5000

WORKDIR /app

COPY requirements.txt /app
RUN pip install -r requirements.txt

COPY app.py /app
CMD python app.py
```

Lo que hace esto es:
- Se crea imagen de Python 3.6.
- Se expone el puerto 5000 (para Flask).
- Se crea un directorio de trabajo en el que se copiarán ```requirements.txt``` y ```app.py```.

Necesitamos que nuestras dependencias (```Flask``` y ```mysql-connector```) se instalen y entreguen con la imagen, por lo que debemos crear el archivo ```requirements.txt``` antes mencionado:
```
Flask
mysql-connector
```

En el caso de la imagen de MySQL, no hará falta crear un ```Dockerfile```puesto que utilizaremos la imagen descargada directamente del ```Docker Hub```.

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

A continuación se muestra el ódigo que se conecta a la base de datos (app.py):
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
