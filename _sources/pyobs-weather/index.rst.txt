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


.. _weather REST API Reference:
REST API Reference
------------------

All API endpoints return JSON data.

Current weather
^^^^^^^^^^^^^^^

The /api/current/ endpoint returns the current values for all sensor types, as shown on the web page in the left column::

    $ http https://weather.monet.saao.ac.za/api/current/

    {
        "good": false,
        "sensors": {
            "dewpoint": {"good": null, "value": -5.22835},
            "humid": {"good": true, "value": 15.754024},
            "press": {"good": null, "value": 824.888119047619},
            "rain": {"good": true, "value": 0.0},
            "skytemp": {"good": false, "value": -19.43},
            "sunalt": {"good": false, "value": 71.0194850756101},
            "temp": {"good": null, "value": 25.7139337142857},
            "winddir": {"good": null, "value": 298.59207},
            "windspeed": {"good": true, "value": 32.2115857142857}
        },
        "time": "2020-02-13T10:44:29.302Z"
    }

Single station/sensor
^^^^^^^^^^^^^^^^^^^^^

A list of weather stations can be retrieved via /api/stations/::

    $ http https://weather.monet.saao.ac.za/api/stations/

    [
        {"code": "average", "name": "Average values"},
        {"code": "current", "name": "Current values"},
        {"code": "observer", "name": "Observer"},
        {"code": "lco", "name": "LCO"},
        {"code": "salt", "name": "SALT"},
        {"code": "suth", "name": "Sutherland Weather"},
        {"code": "monet_cur", "name": "MONET current"},
        {"code": "monet", "name": "MONET"}
    ]


Using one of the codes returned by /api/stations/, all its respective sensor values can be requested from
/api/stations/<station>/::

    $ http https://weather.monet.saao.ac.za/api/stations/monet/

    {
        "code": "monet",
        "name": "MONET",
        "sensors": [
            {
                "code": "temp",
                "name": "Temperature",
                "time": "2020-02-13T10:45:00Z",
                "value": 25.3448275862069
            },
            {
                "code": "press",
                "name": "Pressure",
                "time": "2020-02-13T10:45:00Z",
                "value": 824.786206896552
            },
            {
                "code": "winddir",
                "name": "Wind direction",
                "time": "2020-02-13T10:45:00Z",
                "value": 285.51724137931
            },
            {
                "code": "humid",
                "name": "Relative humidity",
                "time": "2020-02-13T10:45:00Z",
                "value": 16.8931034482759
            },
            {
                "code": "rain",
                "name": "Raining",
                "time": "2020-02-13T10:45:00Z",
                "value": 0.0
            },
            {
                "code": "windspeed",
                "name": "Wind speed",
                "time": "2020-02-13T10:45:00Z",
                "value": 35.0193103448276
            }
        ]
    }

Which also works just for single values using the code of a sensor at /api/stations/<station>/<sensor>/::

    $ http https://weather.monet.saao.ac.za/api/stations/monet/windspeed/

    {
        "code": "windspeed",
        "good": true,
        "name": "Wind speed",
        "since": "2020-02-05T19:00:06.769Z",
        "time": "2020-02-13T10:50:00Z",
        "unit": "km/h",
        "value": 36.2358620689655
    }


History
^^^^^^^

A list of sensor types, for which historic data is available, can be fetched using the /api/history/ endpoint::

    $ http https://weather.monet.saao.ac.za/api/history/

    [
        "humid",
        "skytemp",
        "dewpoint",
        "windspeed",
        "rain",
        "press",
        "temp",
        "winddir"
    ]

Finally, the actual history for one of these sensor types (for all stations) is available via /api/history/<type>/::

    $ http https://weather.monet.saao.ac.za/api/history/temp/

    [
        {
            "code": "average",
            "color": "rgba(0, 0, 0, 0.1)",
            "data": [
                {"time": "2020-02-13T10:50:00.011Z", "value": 25.7011098488768},
                {"time": "2020-02-13T10:45:00.012Z", "value": 25.6358858386895},
                [...]
                {"time": "2020-02-12T10:55:00.015Z","value": 28.482568678953}
            ],
            "name": "Average values"
        },
        {
            "code": "lco",
            "color": "rgba(0, 0, 0, 0.1)",
            "data": [
                {"time": "2020-02-13T10:54:01Z", "value": 27.0},
                {"time": "2020-02-13T10:53:00Z", "value": 26.9},
                [...]
                {"time": "2020-02-12T10:55:00Z", "value": 28.2758620689655}
            ],
            "name": "MONET"
        }
        [...]
    ]
