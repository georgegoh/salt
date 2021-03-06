==============
Reactor System
==============

Salt version 0.11.0 introduced the reactor system. The premise behind the
reactor system is that with Salt's events and the ability to execute commands,
a logic engine could be put in place to allow events to trigger actions, or 
more accurately, reactions. 

This system binds sls files to event tags on the master. These sls files then
define reactions. This means that the reactor system has two parts. First, the
reactor option needs to be set in the master configuration file.  The reactor
option allows for event tags to be associated with sls reaction files. Second,
these reaction files use highdata (like the state system) to define reactions
to be executed.

Event System
============

A basic understanding of the event system is required to understand reactors.
The event system is a local ZeroMQ PUB interface which fires salt events. This
event bus is an open system used for sending information notifying Salt and
other systems about operations.

The event system fires events with a very specific criteria. Every event has a
:strong:`tag` which is comprised of a maximum of 20 characters. Event tags
allow for fast top level filtering of events. In addition to the tag, each
event has a data structure. This data structure is a dict, which contains
information about the event.

Mapping Events to Reactor SLS Files
===================================

Reactor SLS files and event tags are associated in the master config file.
By default this is /etc/salt/master, or /etc/salt/master.d/reactor.conf.

In the master config section 'reactor:' is a list of event tags to be matched
and each event tag has a list of reactor SLS files to be run.

.. code-block:: yaml

    reactor:                           # Master config section "reactor"
     
      - 'minion_start':                # Match tag "minion_start"
        - /srv/reactor/start.sls       # Things to do when a minion starts
        - /srv/reactor/monitor.sls     # Other things to do

      - 'salt/cloud/*/destroyed':     # Globs can be used to matching tags
        - /srv/reactor/decommision.sls # Things to do when a server is removed


Reactor sls files are similar to state and pillar sls files.  They are
by default yaml + Jinja templates and are passed familar context variables.

They differ because of the addtion of the ``tag`` and ``data`` variables.

- The ``tag`` variable is just the tag in the fired event.
- The ``data`` variable is the event's data dict.

Here is a simple reactor sls:

.. code-block:: yaml

    {% if data['id'] == 'mysql1' %}
    highstate_run:
      cmd.state.highstate:
        - tgt: mysql1
    {% endif %}

This simple reactor file uses Jinja to further refine the reaction to be made.
If the ``id`` in the event data is ``mysql1`` (in other words, if the name of
the minion is ``mysql1``) then the following reaction is defined.  The same
data structure and compiler used for the state system is used for the reactor
system. The only difference is that the data is matched up to the salt command
API and the runner system.  In this example, a command is published to the
``mysql1`` minion with a function of ``state.highstate``. Similarly, a runner
can be called:

.. code-block:: yaml

    {% if data['data']['overstate'] == 'refresh' %}
    overstate_run:
      runner.state.over
    {% endif %}

This example will execute the state.overstate runner and initiate an overstate
execution.

Fire an event
=============

To fire an event from a minion call ``event.fire_master``

.. code-block:: bash

    salt-call event.fire_master '{"overstate": "refresh"}' 'foo'

After this is called, any reactor sls files matching event tag ``foo`` will 
execute with ``{{ data['data']['overstate'] }}`` equal to ``'refresh'``.

See :py:mod:`salt.modules.event` for more information.

Understanding the Structure of Reactor Formulas
===============================================

While the reactor system uses the same data structure as the state system, this
data does not translate the same way to operations. In state files formula
information is mapped to the state functions, but in the reactor system
information is mapped to a number of available subsystems on the master. These
systems are the :strong:`LocalClient` and the :strong:`Runners`. The
:strong:`state declaration` field takes a reference to the function to call in
each interface. So to trigger a salt-run call the :strong:`state declaration`
field will start with :strong:`runner`, followed by the runner function to
call. This means that a call to what would be on the command line
:strong:`salt-run manage.up` will be :strong:`runner.manage.up`. An example of
this in a reactor formula would look like this:

.. code-block:: yaml

    manage_up:
      runner.manage.up

If the runner takes arguments then they can be specified as well:

.. code-block:: yaml

    overstate_dev_env:
      runner.state.over:
        - env: dev

Executing remote commands maps to the :strong:`LocalClient` interface which is
used by the :strong:`salt` command. This interface more specifically maps to
the :strong:`cmd_async` method inside of the :strong:`LocalClient` class. This
means that the arguments passed are being passed to the :strong:`cmd_async`
method, not the remote method. A field starts with :strong:`cmd` to use the
:strong:`LocalClient` subsystem. The result is, to execute a remote command, 
a reactor fomular would look like this:

.. code-block:: yaml

    clean_tmp:
      cmd.cmd.run:
        - tgt: '*'
        - arg:
          - rm -rf /tmp/*

The ``arg`` option takes a list of arguments as they would be presented on the
command line, so the above declaration is the same as running this salt
command:

.. code-block:: bash

    salt '*' cmd.run 'rm -rf /tmp/*'

Use the ``expr_form`` argument to specify a matcher:

.. code-block:: yaml

    clean_tmp:
      cmd.cmd.run:
        - tgt: 'os:Ubuntu'
        - expr_form: grain
        - arg:
          - rm -rf /tmp/*


    clean_tmp:
      cmd.cmd.run:
        - tgt: 'G@roles:hbase_master'
        - expr_form: compound
        - arg:
          - rm -rf /tmp/*
