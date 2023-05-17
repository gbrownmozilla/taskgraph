.. _from_deps:

From Dependencies
=================

The :mod:`taskgraph.transforms.from_deps` transforms can be used to create
tasks based on the kind dependencies, filtering on common attributes like the
``build-type``.

These transforms are useful when you want to create follow-up tasks for some
indeterminate subset of existing tasks. For example, maybe you want to run
a signing task after each build task.


Usage
-----

Add the transform to the ``transforms`` key in your ``kind.yml`` file:

.. code-block:: yaml

   transforms:
     - taskgraph.transforms.from_deps
     # ...

Then create a ``from-deps`` section in your task definition, e.g:

.. code-block:: yaml

   kind-dependencies:
     - build
     - toolchain

   tasks:
     signing:
       from-deps:
         kinds: [build]

This example will split the ``signing`` task into many, one for each build. If
``kinds`` is unspecified, then it defaults to all kinds listed in the
``kind-dependencies`` key. So the following is valid:

.. code-block:: yaml

   kind-dependencies:
     - build
     - toolchain

   tasks:
     signing:
       from-deps: {}

In this example, a task will be created for each build *and* each toolchain.

Limiting Dependencies by Attribute
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

It may not be desirable to create a new task for *all* tasks in the given kind.
It's possible to limit the tasks given some attribute:

.. code-block:: yaml

   kind-dependencies:
     - build

   tasks:
     signing:
       from-deps:
         with-attributes:
           platform: linux

In the above example, follow-up tasks will only be created for builds whose
``platform`` attribute equals "linux". Multiple attributes can be specified,
these are resolved using the :func:`~taskgraph.util.attributes.attrmatch`
utility function.

Grouping Dependencies
~~~~~~~~~~~~~~~~~~~~~

Sometimes, it may be desirable to run a task after a group of related tasks
rather than an individual task. One example of this, is a task to notify when a
release is finished for each platform we're shipping. We want it to depend on
every task from that particular release phase.

To accomplish this, we specify the ``group-by`` key:

.. code-block:: yaml

   kind-dependencies:
     - build
     - signing
     - publish

   tasks:
     notify:
       from-deps:
         group-by:
           attribute: platform

In this example, tasks across the ``build``, ``signing`` and ``publish`` kinds will
be scanned for an attribute called "platform" and sorted into corresponding groups.
Assuming we're shipping on Windows, Mac and Linux, it might create the following
groups:

.. code-block::

   - build-windows, signing-windows, publish-windows
   - build-mac, signing-mac, publish-mac
   - build-linux, signing-linux, publish-linux

Then the ``notify`` task will be duplicated into three, one for each group. The
notify tasks will depend on each task in its associated group.

Custom Grouping
~~~~~~~~~~~~~~~

Only the default ``single`` and the ``attribute`` group-by functions are
built-in. But if more complex grouping is needed, custom functions can be
implemented as well:

.. code-block:: python

   from typing import List

   from taskgraph.task import Task
   from taskgraph.transforms.from_deps import group_by
   from taskgraph.transforms.base import TransformConfig

   @group_by("custom-name")
   def group_by(config: TransformConfig, tasks: List[Task]) -> List[List[Task]]:
      pass

This can then be used in a task like so:

.. code-block:: yaml

   from-deps:
     group-by: custom-name

It's also possible to specify a schema for your custom group-by function, which
allows tasks to pass down additional context (such as with the built-in
``attribute`` function):

.. code-block:: python

   from typing import List

   from taskgraph.task import Task
   from taskgraph.transforms.from_deps import group_by
   from taskgraph.transforms.base import TransformConfig
   from taskgraph.util.schema import Schema

   @group_by("custom-name", schema=Schema(str))
   def group_by(config: TransformConfig, tasks: List[Task], ctx: str) -> List[List[Task]]:
      pass

The extra context can be passed by turning ``group-by`` into an object
instead of a string:

.. code-block:: yaml

   from-deps:
     group-by:
       custom-name: foobar

In the above example, the value ``foobar`` is what must conform to the schema defined
by the ``group_by`` function.

Primary Kind
~~~~~~~~~~~~

Each task has a ``primary-kind``. This is the kind dependency in each grouping
that comes first in the list of supported kinds (either via the
``kind-dependencies`` in the ``kind.yml`` file, or via the ``from-deps.kinds``
key). Note that depending how the dependencies get grouped, a given group may
not contain a dependency for each kind. Therefore the list of kind dependencies
are ordered by preference. E.g, kinds earlier in the list will be chosen as the
primary kind before kinds later in the list.

The primary kind is used to derive the task's label, as well as copy attributes
if the ``copy-attributes`` key is set to ``True`` (see next section).

Each task created by the ``from_deps`` transforms, will have a
``primary-kind-dependency`` attribute set.

Copying Attributes
~~~~~~~~~~~~~~~~~~

It's often useful to copy attributes from a dependency. When this key is set to ``True``,
all attributes from the ``primary-kind`` (see above) will be copied over to the task. If
the task contain pre-existing attributes, they will not be overwritten.