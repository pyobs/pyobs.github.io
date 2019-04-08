Basic Usage
===========

After installing pyobs, you have the new command ``pyobs``, which creates and
starts pyobs modules from the command line based on a configuration file, written
in YAML.

A simple configuration file (``standalone.yaml``) might look like this::

    class: pyobs.modules.test.StandAlone
    message: Hello world
    interval: 10

Basically you always define a ``class`` for a block together with its properties.

In this example, the module is of type :class:`pyobs.modules.test.StandAlone`, which is a trivial implementation
of a module that does nothing more than logging a given message continuously in a given interval::

    class StandAlone(pyObsModule):
        def __init__(self, *args, **kwargs):
            pyobsModule.__init__(self, thread_funcs=self.thread_func, *args, **kwargs)

        @classmethod
        def default_config(cls):
            cfg = super(StandAlone, cls).default_config()
            cfg['message'] = 'Hello world'
            cfg['interval'] = 10
            return cfg

        def open(self) -> bool:
            return pyobsModule.open(self)

        def close(self):
            pyobsModule.close(self)

        def thread_func(self):
            while not self.closing.is_set():
                log.info(self.config['message'])
                self.closing.wait(self.config['interval'])

The constructor just calls the constructor of :class:`pyobs.pyObsModule`, adding the ``thread_funcs``
parameter, that takes a method that is run in an extra thread. In this case, it is the method
thread_func(), that does some logging in a loop that runs until the
program quits.

The class method default_config() defines the default configuration for the module, and open() and close()
are called when the module is opened and closed, respectively.

If the configuration file is saved as ``standalone.yaml``, one can easily start it via the ``pyobs`` command::

    pyobs standalone.yaml

The program quits gracefully when it receives an interrupt, so you can stop it by simply pressing ``Ctrl+c``.

Environment
-----------

There is some functionality that is required in many modules, including those concerning the environment,
especially the location of the telescope and the local time. For this, the :class:`pyobs.Application` class
has support for an additional module of type :class:`pyobs.environment.Environment`, which can be
defined in the application's configuration like this::

    environment:
      class: pyobs.environment.Environment
      timezone: utc
      location:
        longitude: 20.810808
        latitude: -32.375823
        elevation: 1798.

Now an object of this type is automatically pushed into the module and can be accessed via the ``environment``
property, e.g.::

    def open(self) -> bool:
        print(self.environment.location)
        return pyobsModule.open(self)


Comm
----

In case the module is supposed to communicate with others, we need another module of type
:class:`pyobs.comm.Comm`, which can be defined in the application's configuration like this::

    comm:
      class: pyobs.comm.xmpp.XmppComm
      jid: some_module@my.domain.com

More details about this can be found in the :doc:`auxiliary/comm` section.
