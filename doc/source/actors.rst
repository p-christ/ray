.. _actor-guide:

Using Actors
============

An actor is essentially a stateful worker (or a service). When a new actor is
instantiated, a new worker is created, and methods of the actor are scheduled on
that specific worker and can access and mutate the state of that worker.

Creating an actor
-----------------

You can convert a standard Python class into a Ray actor class as follows:

.. code-block:: python

  @ray.remote
  class Counter(object):
      def __init__(self):
          self.value = 0

      def increment(self):
          self.value += 1
          return self.value

Note that the above is equivalent to the following:

.. code-block:: python

  class Counter(object):
      def __init__(self):
          self.value = 0

      def increment(self):
          self.value += 1
          return self.value

  Counter = ray.remote(Counter)

When the above actor is instantiated, the following events happen.

1. A node in the cluster is chosen and a worker process is created on that node
   for the purpose of running methods called on the actor.
2. A ``Counter`` object is created on that worker and the ``Counter``
   constructor is run.

Actor Methods
-------------

Any method of the actor can return multiple object refs with the ``ray.method`` decorator:

.. code-block:: python

    @ray.remote
    class Foo(object):

        @ray.method(num_returns=2)
        def bar(self):
            return 1, 2

    f = Foo.remote()

    obj_ref1, obj_ref2 = f.bar.remote()
    assert ray.get(obj_ref1) == 1
    assert ray.get(obj_ref2) == 2

.. _actor-resource-guide:

Resources with Actors
---------------------

You can specify that an actor requires CPUs or GPUs in the decorator. While Ray has built-in support for CPUs and GPUs, Ray can also handle custom resources.

When using GPUs, Ray will automatically set the environment variable ``CUDA_VISIBLE_DEVICES`` for the actor after instantiated. The actor will have access to a list of the IDs of the GPUs
that it is allowed to use via ``ray.get_gpu_ids()``. This is a list of strings,
like ``[]``, or ``['1']``, or ``['2', '5', '6']``. Under some circumstances, the IDs of GPUs could be given as UUID strings instead of indices (see the `CUDA programming guide <https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#env-vars>`__).

.. code-block:: python

  @ray.remote(num_cpus=2, num_gpus=1)
  class GPUActor(object):
      pass

When an ``GPUActor`` instance is created, it will be placed on a node that has
at least 1 GPU, and the GPU will be reserved for the actor for the duration of
the actor's lifetime (even if the actor is not executing tasks). The GPU
resources will be released when the actor terminates.

If you want to use custom resources, make sure your cluster is configured to
have these resources (see `configuration instructions
<configure.html#cluster-resources>`__):

.. note::

  * If you specify resource requirements in an actor class's remote decorator,
    then the actor will acquire those resources for its entire lifetime (if you
    do not specify CPU resources, the default is 1), even if it is not executing
    any methods. The actor will not acquire any additional resources when
    executing methods.
  * If you do not specify any resource requirements in the actor class's remote
    decorator, then by default, the actor will not acquire any resources for its
    lifetime, but every time it executes a method, it will need to acquire 1 CPU
    resource.


.. code-block:: python

  @ray.remote(resources={'Resource2': 1})
  class GPUActor(object):
      pass


If you need to instantiate many copies of the same actor with varying resource
requirements, you can do so as follows.

.. code-block:: python

  @ray.remote(num_cpus=4)
  class Counter(object):
      def __init__(self):
          self.value = 0

      def increment(self):
          self.value += 1
          return self.value

  a1 = Counter.options(num_cpus=1, resources={"Custom1": 1}).remote()
  a2 = Counter.options(num_cpus=2, resources={"Custom2": 1}).remote()
  a3 = Counter.options(num_cpus=3, resources={"Custom3": 1}).remote()

Note that to create these actors successfully, Ray will need to be started with
sufficient CPU resources and the relevant custom resources.


Terminating Actors
------------------

Actor processes will be terminated automatically when the initial actor handle
goes out of scope in Python. If we create an actor with ``actor_handle =
Counter.remote()``, then when ``actor_handle`` goes out of scope and is
destructed, the actor process will be terminated. Note that this only applies to
the original actor handle created for the actor and not to subsequent actor
handles created by passing the actor handle to other tasks.

If necessary, you can manually terminate an actor by calling
``ray.actor.exit_actor()`` from within one of the actor methods. This will kill
the actor process and release resources associated/assigned to the actor. This
approach should generally not be necessary as actors are automatically garbage
collected. The ``ObjectRef`` resulting from the task can be waited on to wait
for the actor to exit (calling ``ray.get()`` on it will raise a ``RayActorError``).
Note that this method of termination will wait until any previously submitted
tasks finish executing and then exit the process gracefully with sys.exit. If you
want to terminate an actor forcefully, you can call ``ray.kill(actor_handle)``.
This will call the exit syscall from within the actor, causing it to exit
immediately and any pending tasks to fail. This will not go through the normal
Python sys.exit teardown logic, so any exit handlers installed in the actor using
``atexit`` will not be called.

Passing Around Actor Handles
----------------------------

Actor handles can be passed into other tasks. To illustrate this with a
simple example, consider a simple actor definition.

.. code-block:: python

  @ray.remote
  class Counter(object):
      def __init__(self):
          self.counter = 0

      def inc(self):
          self.counter += 1

      def get_counter(self):
          return self.counter

We can define remote functions (or actor methods) that use actor handles.

.. code-block:: python

  import time

  @ray.remote
  def f(counter):
      for _ in range(1000):
          time.sleep(0.1)
          counter.inc.remote()

If we instantiate an actor, we can pass the handle around to various tasks.

.. code-block:: python

  counter = Counter.remote()

  # Start some tasks that use the actor.
  [f.remote(counter) for _ in range(3)]

  # Print the counter value.
  for _ in range(10):
      time.sleep(1)
      print(ray.get(counter.get_counter.remote()))

Named Actors
------------

An actor can be given a globally unique name via ``.options(name="some_name")``,
which allows you to retrieve the actor from any job in the Ray cluster via
``ray.get_actor("some_name")``. This can be useful if you cannot directly
pass the actor handle to the task that needs it, or if you are trying to
access an actor launched by another driver.

Actor Lifetimes
---------------

Separately, actor lifetimes can be decoupled from the job, allowing an actor to
persist even after the driver process of the job exits.

.. code-block:: python

  counter = Counter.options(name="CounterActor", lifetime="detached").remote()

The CounterActor will be kept alive even after the driver running above script
exits. Therefore it is possible to run the following script in a different
driver:

.. code-block:: python

  counter = ray.get_actor("CounterActor")
  print(ray.get(counter.get_counter.remote()))

Note that the lifetime option is decoupled from the name. If we only specified
the name without specifying ``lifetime="detached"``, then the CounterActor can
only be retrieved as long as the original driver is still running.

Actor Pool
----------

The ``ray.util`` module contains a utility class, ``ActorPool``.
This class is similar to multiprocessing.Pool and lets you schedule Ray tasks over a fixed pool of actors.

.. code-block:: python

  from ray.util import ActorPool

  a1, a2 = Actor.remote(), Actor.remote()
  pool = ActorPool([a1, a2])
  print(pool.map(lambda a, v: a.double.remote(v), [1, 2, 3, 4]))
  # [2, 4, 6, 8]

See the `package reference <package-ref.html#ray.util.ActorPool>`_ for more information.
