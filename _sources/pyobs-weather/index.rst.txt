*pyobs-weather*
===============

*pyobs-weather* is an aggregator for data from several weather stations. Rules can be defined for when weather is
defined to be "good". It provides both a web frontend and an API for access.

.. _weather Installation:
Installation
------------

While it is definitely possible to run *pyobs-weather* without Docker, we highly recommend it for its simplicity.

First, build the image::

    cd https://github.com/pyobs/pyobs-weather.git
    cd pyobs-weather
    docker build . -t pyobs-weather

*pyobs-weather* requires a database for storing its data, redis for task brokering, celery for providing those tasks,
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

And *pyobs-weather* itself needs a configuration file called local_settings.py. Here is the file for MONET/S as an
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


.. _weather Configuration:
Configuration
-------------

Setting *pyobs-weather* up mainly consists of adding weather stations and evaluators for their sensors in the admin
panel. Three stations have automatically been added by the initweather weather script during the installation:

- Current: Always returns the averaged latest values from all weather stations.
- Average: Returns 5-minutes averages for all values
- Observer: An "astronomical observer" that mainly calculates the position of the sun.


Adding a station
^^^^^^^^^^^^^^^^

When adding a new station, one has to provide a unique code for it and a name. Then a Python class must be given
and optionally JSON encoded arguments that are passed to its constructor. Finally, for regular polling of the station
a crontab or an interval must be specified.

See a list of available weather station classes in the :ref:`Stations` below. New station types can easily be added
by creating new Python classes.


Adding evaluators
^^^^^^^^^^^^^^^^^

Adding a new station automatically adds its sensors to the system. In order to device, whether the current value
of a sensors is good or bad, evaluators need to be attached to them. For a list of available evaluators see
:ref:`Evaluators`. New evaluators can simply be added by creating new Python classes.

For adding an evaluator, you just click on a sensor and select the evaluator from the list of evaluators.

.. _weather API Reference:
API Reference
-------------

Sensor Types
^^^^^^^^^^^^

*pyobs-weather* assumes that every weather station returns only one value for a given *type* of sensor, which makes
it easier to combine values from different stations. For instance, if multiple stations give a value for "humid", the
system can easily combine them to create an average humidity.

Sensor types are defined within the Python classes for the stations, and can be extended by new stations easily. The
current list of sensor types is:

- temp: Temperature [°C]
- humid: Relative humidity [%]
- press: Pressure [hPa]
- winddir: Wind direction [°E of N']
- windspeed: Wind speed [km/h']
- particles: Particle count [ppcm]
- rain: Raining [0/1]
- skytemp: Relative sky temperature [°C]
- sunalt: Solar altitude [°]


Stations
^^^^^^^^

A station in *pyobs-weather* is a single weather station, from which it polls data.

.. toctree::
   :maxdepth: 2

   stations/index


Evaluators
^^^^^^^^^^

Evaluators take sensors and evaluate their values.

.. toctree::
   :maxdepth: 2

   evaluators/index
