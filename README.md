CREACIÓN DE CODEBASE PARA CONTROL DE ENTORNOS EN DJANGO
=======================================================

## Crear el proyecto ##

> django-admin startproject my_codebase

## Acceder al proyecto ##

> cd my_codebase

## Crear archivos de arranque de proyectos de docker-compose local y producción ##

> touch local.py

> touch production.py

En cada uno, se carga los contenedores a usar

´´´´´´
# local.yml
version: '3'

services:
  django: &django
    restart: always
    build:
      context: .
      dockerfile: ./Dockerfiles/local/django
    image: mydev
    env_file:
      - ./.envs/.local/.sys
    volumes:
      - .:/usr/src/app
    ports:
      - "8000:8000"
    command: /start
´´´´´´


´´´´´´
#production.yml
version: '3'

services:
  django: &django
    restart: always
    build:
      context: .
      dockerfile: ./Dockerfiles/production/django
    image: system
    env_file:
      - ./.envs/.production/.sys
    ports:
      - "80:80"
    command: /start

´´´´´´

## Crear carpetas de variables de entorno ##

> mkdir .envs

> mkdir .envs/.local

> mkdir .envs/.production

Se cargan variables de entorno

> vi .envs/.local/.sys

´´´´´´
#.envs/.local/.sys
DEBUG=on
DJANGO_SETTINGS_MODULE=my_codebase.settings.local
DB_SQL=local.sqlite3
´´´´´´

> vi .envs/.production/.sys

´´´´´´
#.envs/.production/.sys
DEBUG=off
DJANGO_SETTINGS_MODULE=my_codebase.settings.production
DB_SQL=production.sqlite3
´´´´´´

## Crear archivos de requerimientos por entorno ##

Se crea una carpeta de requerimientos

> mkdir requirements

Se crean un archivo base de requerimientos

> vi requirements/base.txt

´´´´´´
# requirements/base.txt
Django>=2.0,<3.0
django-environ==0.4.5
´´´´´´

Se crean otros dos archivos local.txt y produccion.txt donde iran otros complementos propios de cada entorno, procurando copiar el contenido de la base.

> vi requirements/local.txt

´´´´´´
#r equirement/local.txt
-r ./base.txt
# Debugging
ipdb==0.11
# Testing
mypy==0.650
pytest==4.0.2
pytest-sugar==0.9.2
pytest-django==3.4.4
factory-boy==2.11.1
# Code quality
flake8==3.6.0
´´´´´´

> vi requirements/production.txt

´´´´´´
# requirements/production.txt
-r ./base.txt
´´´´´´

## Dividir el archivo settings ##

Se crea una carpeta "settings" dentro de la carpeta de configuración. Se mueve el settings.py a la nueva carpeta renombrado como "base.py"

> mkdir my_codebase/settings

> mv my_codebase/settings.py my_codebase/settings/base.py

Se deben hacer algunos ajustes al archivo de configuración base para que comience a apuntar a variables de entorno.

> vi my_codebase/settings/base.py

´´´´´´
import os
import environ # Librería de versiones

...

# Carga variables de entorno
env = environ.Env(
    DEBUG=(bool, False)
)

...

# Comentar la siguiente línea

# ALLOWED_HOSTS = []

...

# Modo debug segun lo que marquen los archivos de entorno
DEBUG = env('DEBUG')

...

# Ejemplo para dividir las DB local de producción
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, env('DB_SQL')),
    }
}

´´´´´´

Se deben crear los archivos local.py y production.py con los cambios que sean diferentes uno del otro. Para este ejemplo, solo importarán la información de la base, pero en cada proyecto particular, se deben borrar variables de la base y poner los valores distintos de cada entorno.

> vi my_codebase/settings/local.py

´´´´´´
from .base import *  # noqa
from .base import env # noqa

DEBUG = True
ALLOWED_HOSTS = ['*']
´´´´´´

> vi my_codebase/settings/production.py

´´´´´´
from .base import *  # noqa
from .base import env # noqa

# Está IP variará según donde sea implementado
ALLOWED_HOSTS = ['192.168.0.10:']
´´´´´´

# Arranque con los nuevos settings #

> vi my_codebase/wsgi.py

´´´´´´
# Modificar la siguiente línea
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'my_codebase.settings.local')
´´´´´´

## Dockerfiles ##

Crear la carpeta de los dockerfiles declarados en los archivos yml.

> mkdir Dockerfiles

> mkdir Dockerfiles/local

> mkdir Dockerfiles/production

Cargar la información del funcionamiento de los dockerfiles

> vi Dockerfiles/local/django

´´´´´´
FROM python:3.6

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        postgresql-client \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src/app
COPY ./requirements /requirements
RUN pip install -r /requirements/local.txt

COPY ./Dockerfiles/local/start /start
RUN sed -i 's/\r//' /start
RUN chmod +x /start
´´´´´´

> vi Dockerfiles/production/django

´´´´´´
FROM python:3.6

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        postgresql-client \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src/app
COPY ./requirements /requirements
RUN pip install -r /requirements/production.txt

COPY ./Dockerfiles/production/start /start
RUN sed -i 's/\r//' /start
RUN chmod +x /start
´´´´´´

Los archivos start sirven para correr los los servicios de Django. Hay que procurar que los puertos coincidan con los que mencionan los archivos yml.

> vi Dockerfiles/local/start

´´´´´´
#!/bin/sh
python manage.py migrate
python manage.py runserver 0.0.0.0:8000
´´´´´´

> vi Dockerfiles/production/start

´´´´´´
#!/bin/sh
python manage.py migrate
python manage.py runserver 0.0.0.0:80
´´´´´´

## Test de entornos ##

Cargaremos una página dummy en el urls.

> vi my_codebase/urls.py

´´´´´´
from django.urls import path

from django.http import HttpResponse

def hello_world(request):
    return HttpResponse('Listo!!!')

urlpatterns = [
    path('', hello_world)
]
´´´´´´

Probemos el entorno local

> docker-compose -f local.yml build

> docker-compose -f local.yml up


Al entrar vía browser a http://192.168.0.10:8000/ , abrirá el mensaje de "Listo". Igualmente, si mandamos http://192.168.0.10:8000/abc , nos lanzará el debug del error.

Con el siguiente comando, se observa el nombre de la imagen se cargó del local.yml.

>docker ps

´´´´´´
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                    NAMES
2edbec522ef1        mydev               "/start"            7 minutes ago       Up 3 minutes        0.0.0.0:8000->8000/tcp   my_codebase_django_1
´´´´´´

Ahora, probemos con el entorno productivo

> docker-compose -f production.yml build

> docker-compose -f production.yml up

Para este entorno, debemos ingresar con http://192.168.0.10. Si le ingresamos otro parámetro a la URL, mandará el mensaje "Not Found".

En el listado de contenedores activos, la imegn será la que menciona production.yml.

> docker ps

´´´´´´
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                NAMES
3a53d907080c        real_system         "/start"            21 seconds ago      Up 19 seconds       0.0.0.0:80->80/tcp   my_codebase_django_1
´´´´´´
