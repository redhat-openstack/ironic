.. _architecture:

===================
System Architecture
===================

High Level description
======================

An Ironic deployment will be composed of the following components:

- An admin-only RESTful `API service`_, by which privileged users, such as
  cloud operators and other services within the cloud control plane, may
  interact with the managed bare metal servers.
- A `Conductor service`_, which does the bulk of the work. Functionality is
  exposed via the `API service`_.  The Conductor and API services communicate via
  RPC.
- A Database and `DB API`_ for storing the state of the Conductor and Drivers.
- A Deployment Ramdisk or Deployment Agent, which provide control over the
  hardware which is not available remotely to the Conductor.  A ramdisk should be
  built which contains one of these agents, eg. with `diskimage-builder`_.
  This ramdisk can be booted on-demand.

    - **NOTE:** The agent is never run inside a tenant instance.

Drivers
=======

The internal driver API provides a consistent interface between the
Conductor service and the driver implementations. A driver is defined by
a class inheriting from the `BaseDriver`_ class, defining certain interfaces;
each interface is an instance of the relevant driver module.

For example, a fake driver class might look like this::

    class FakePower(base.PowerInterface):
        def get_properties(self):
            return {}

        def validate(self, task):
            pass

        def get_power_state(self, task):
            return states.NOSTATE

        def set_power_state(self, task, power_state):
            pass

        def reboot(self, task):
            pass

    class FakeDriver(base.BaseDriver):
        def __init__(self):
            self.power = FakePower()


There are three categories of driver interfaces:

- `Core` interfaces provide the essential functionality for Ironic within
  OpenStack, and may be depended upon by other services. All drivers
  must implement these interfaces. The Core interfaces are `power` and `deploy`.
- `Standard` interfaces provide functionality beyond the needs of OpenStack,
  but which have been standardized across all drivers and becomes part of
  Ironic's API.  If a driver implements this interface, it must adhere to the
  standard. This is presented to encourage vendors to work together with the
  Ironic project and implement common features in a consistent way, thus
  reducing the burden on consumers of the API. The Standard interfaces are
  `management`, `console`, `boot`, `inspect`, and `raid`.
- The `Vendor` interface allows an exemption to the API contract when a vendor
  wishes to expose unique functionality provided by their hardware and is
  unable to do so within the `Core` or `Standard` interfaces. In this case,
  Ironic will merely relay the message from the API service to the appropriate
  driver.

Driver-Specific Periodic Tasks
------------------------------

Drivers may run their own periodic tasks, i.e. actions run repeatedly after
a certain amount of time. Such task is created by decorating a method on the
driver itself or on any interface with driver_periodic_task_ decorator, e.g.

::

    class FakePower(base.PowerInterface):
        @base.driver_periodic_task(spacing=42)
        def task(self, manager, context):
            pass  # do something

    class FakeDriver(base.BaseDriver):
        def __init__(self):
            self.power = FakePower()

        @base.driver_periodic_task(spacing=42)
        def task2(self, manager, context):
            pass  # do something


Here the ``spacing`` argument is a period in seconds for a given periodic task.
For example 'spacing=5' means every 5 seconds.

.. note::
    The ``parallel`` argument may be passed to driver_periodic_task_.
    If it's set to False, this task will be run in the periodic task loop,
    rather than a separate greenthread.

    This is deprecated as of Liberty release, and the parallel argument will be
    ignored starting in the Mitaka cycle, as such task would prevent all other
    periodic tasks from starting while it is is running.

.. note::
    By default periodic task names are derived from method names,
    so they should be unique within a Python module.
    Use ``name`` argument to driver_periodic_task_ to override
    automatically generated name.


Message Routing
===============

Each Conductor registers itself in the database upon start-up, and periodically
updates the timestamp of its record. Contained within this registration is a
list of the drivers which this Conductor instance supports.  This allows all
services to maintain a consistent view of which Conductors and which drivers
are available at all times.

Based on their respective driver, all nodes are mapped across the set of
available Conductors using a `consistent hashing algorithm`_. Node-specific
tasks are dispatched from the API tier to the appropriate conductor using
conductor-specific RPC channels.  As Conductor instances join or leave the
cluster, nodes may be remapped to different Conductors, thus triggering various
driver actions such as take-over or clean-up.


.. _API service: ../webapi/v1.html
.. _BaseDriver: ../api/ironic.drivers.base.html#ironic.drivers.base.BaseDriver
.. _Conductor service: ../api/ironic.conductor.manager.html
.. _DB API: ../api/ironic.db.api.html
.. _diskimage-builder: https://github.com/openstack/diskimage-builder
.. _consistent hashing algorithm: ../api/ironic.common.hash_ring.html
.. _driver_periodic_task: ../api/ironic.drivers.base.html#ironic.drivers.base.driver_periodic_task
