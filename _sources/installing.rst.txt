.. _installing:

Installing pyobs
================

Installing *pyobs* is as simple as calling pip::

    pip3 install pyobs-core


.. _installing-ejabberd:

Setting up ejabberd
-------------------
In case you already have a working XMPP server, skip this step.

1. Download ejabberd from https://www.process-one.net/en/ejabberd/downloads/ and install it. Or on Linux use your
package manager to do so.

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
