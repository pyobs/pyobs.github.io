*pyobs-weather*
===============

*pyobs-weather* is an aggregator for data from several weather stations. Rules can be defined for when weather is
defined to be "good". It provides both a web frontend and an API for access.

Installation
------------

While it is definitely possible to run *pyobs-weather* without Docker, we highly recommend it for its simplicity.

First, build the image::

    cd https://github.com/pyobs/pyobs-weather.git
    cd pyobs-weather
    docker build . -t pyobs-weather

pyobs-weather requires a database for storing its data, redis for task brokering, celery for providing those tasks,
and nginx for serving static files. Easiest way to deploy everything is using docker-compose.

A typical docker-compose.yml looks like this::

    version: '3'

    services:
      db:
        image: postgres:11
        volumes:
          - pgdata:/var/lib/postgresql/data
        restart: always

      weather:
        image: pyobs-weather
        volumes:
          - ./local_settings.py:/archive/pyobs_archive/local_settings.py
          - static:/weather/static
        depends_on:
          - db
        restart: always
        command: bash -c "python manage.py collectstatic --no-input && python manage.py makemigrations && python manage.py migrate && gunicorn --workers=3 pyobs_weather.wsgi -b 0.0.0.0:8000"

      redis:
        image: redis
        restart: always

      celery:
        image: pyobs-weather
        command: celery -A pyobs_weather worker --beat --scheduler django --loglevel=info
        depends_on:
          - db
          - redis
        restart: always

      nginx:
        image: nginx
        volumes:
          - ./nginx.conf:/etc/nginx/conf.d/default.conf
          - static:/static/static
        ports:
          - 8002:80
        restart: always

    volumes:
      pgdata:
      static:

In this example, nginx needs a configuration file nginx.conf in the same directory, which might look like this::

    server {
        listen 80;
        server_name  127.0.0.1;

        location / {
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $host;
            proxy_redirect off;
            if (!-f $request_filename) {
                proxy_pass http://weather:8000;
                break;
            }
        }

        location /static/ {
            root /static;
        }
    }

And pyobs-weather itself needs a configuration file called local_settings.py. Here is the file for MONET/S as an
example::

    # disable debug
    DEBUG = False

    # we're reverse proxying, so only localhost is allowed to access
    ALLOWED_HOSTS = ['localhost']

    # weather settings
    OBSERVER_NAME = 'MONET/S @ SAAO'
    OBSERVER_LOCATION = {'longitude': 20.810808, 'latitude': -32.375823, 'elevation': 1798.}
    WINDOW_TITLE = 'Weather at ' + OBSERVER_NAME

With all three files in one directory, you can easily do::

    docker-compose up -d

and the whole system should be up and running within a minute.

Finally, you need to get into the container and init the database (name of container may vary)::

    docker exec -it weather_weather_1 bash
    ./manage.py initweather

While at it, you can also create a super user::

    ./manage.py createsuperuser

The web frontend should now be accessible via web browser at http://localhost:8002/ and the admin panel
at http://localhost:8002/admin.