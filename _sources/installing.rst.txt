.. _installing:

Installing pyobs
================

*pyobs* supports two different ways of installing it: while for a development system a local installation of the
Python packages and executables is preferable, on a productive system the Docker setup may have its advantages.

Furthermore, an XMPP server is required. We recommend ejabberd.

.. _installing-ejabberd:

Setting up ejabberd
-------------------
In case you already have a working XMPP server, skip this step.

1. Download ejabberd from https://www.process-one.net/en/ejabberd/downloads/ and install it.

2. Since the allowed packet sizes are by default a little too small, find the ejabberd config file **ejabberd.yml**
   and find and edit the "shaper" part::

    shaper:
      normal: 100000
      fast: 5000000

3. Start ejabberd server using::

    ejabberdctl start

4. Add a Shared Roster Group so that all clients are in each others roster (replace <host> with local hostname)::

    ejabberdctl srg_create all <host> all all all
    ejabberdctl srg_user_add @all@ <host> all <host>

5. Register users (may skip for now), e.g.::

    ejabberdctl register <name> <host> <password>

Local installation
------------------
If you want to install multiple *pyobs* repositories, it is easier to keep them all in one place::

    mkdir pyobs
    cd pyobs

Then clone the *pyobs-core* repository::

    git clone git@gitlab.gwdg.de:pyobs/pyobs-core.git pyobs-core
    cd pyobs-core

If you don't want to install *pyobs* system-wide, you can now set up a virtual environment::

    python3 -m venv venv
    source ./venv/bin/activate

Either way, this install the required Python packages::

    pip install -r requirements.txt

Finally, install *pyobs* itself::

    python3 setup.py install

You now have the :program:`pyobs` (see :ref:`cli-pyobs`) executable available to start *pyobs* modules.


Using the pyobsd tool
---------------------

*pyobs* comes with its own little tool called *pyobsd* for starting and stopping *pyobs* modules
(see :ref:`cli-pyobsd`). On Linux systems, you should create a new user "pyobs"::

    adduser pyobs --home /opt/pyobs

Note that we've set the user's home directory to /opt/pyobs.

Change into the new user, and create some directories::

    su pyobs
    mkdir -p /opt/pyobs/config
    mkdir -p /opt/pyobs/log
    mkdir -p /opt/pyobs/run

Every configuration YAML file in the *config* directory will now automatically show up in the *pyobsd* tool.
Logs will be written into the *log* directory, and PID files for each process into *run*.

Docker
------

TBD
