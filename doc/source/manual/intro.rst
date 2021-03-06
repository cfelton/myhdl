.. currentmodule:: myhdl

.. _intro:

*********************
Introduction to MyHDL
*********************

.. _intro-basic:

A basic MyHDL simulation
========================

We will introduce MyHDL with a classic ``Hello World`` style example. All
example code can be found in the distribution directory under
:file:`example/manual/`.  Here are the contents of a MyHDL simulation script
called :file:`hello1.py`:

.. include-example:: hello1.py

When we run this simulation, we get the following output:

.. run-example:: hello1.py

The first line of the script imports a number of objects from the ``myhdl``
package. In Python we can only use identifiers that are literally defined in the
source file   [#]_.

Then, we define a function called :func:`HelloWorld`. In MyHDL, hardware
modules can be modeled using classic functions.  In particular, the parameter list
is then used to define the interface. In this first example, the interface is empty.

.. index:: single: decorator; always

Inside the top level function we declare a local function called
:func:`sayHello` that defines the desired behavior. This function is decorated
with an :func:`always` decorator that has a delay  object as its parameter.  The
meaning is that the function will be executed whenever the specified delay
interval has expired.

Behind the curtains, the :func:`always` decorator creates a Python *generator*
and reuses the name of the decorated function for it. Generators are the
fundamental objects in MyHDL, and we will say much more about them further on.

Finally, the top level function returns the local generator. This is the
simplest case of the basic MyHDL code pattern to define the contents of a
hardware module. We will describe the general case further on.

In MyHDL, we create an *instance* of a hardware module by calling the
corresponding function. In the example, variable ``inst`` refers to an instance
of :func:`HelloWorld`.  To simulate the instance, we pass it as an argument to a
:class:`Simulation` object constructor.  We then run the simulation for the
desired amount of timesteps.


.. _intro-conc:

Signals, ports, and concurrency
===============================

In the previous section, we simulated a design with a single generator and no
concurrency. Real hardware descriptions are typically
massively concurrent. MyHDL supports this by allowing an arbitrary number of
concurrently running generators.

With concurrency comes the problem of deterministic communication. Hardware
languages use special objects to support deterministic communication between
concurrent code. In particular, MyHDL  has a :class:`Signal` object which is
roughly modeled after VHDL signals.

We will demonstrate signals and concurrency by extending and modifying our first
example. We define two hardware modules, one that drives a clock signal, and one
that is sensitive to a positive edge on a clock signal::

   from myhdl import Signal, delay, always, now, Simulation


   def ClkDriver(clk):

       halfPeriod = delay(10)

       @always(halfPeriod)
       def driveClk():
           clk.next = not clk

       return driveClk


   def HelloWorld(clk):

       @always(clk.posedge)
       def sayHello():
           print "%s Hello World!" % now()

       return sayHello


   clk = Signal(0)
   clkdriver_inst = ClkDriver(clk)
   hello_inst = HelloWorld(clk)
   sim = Simulation(clkdriver_inst, hello_inst)
   sim.run(50)

.. index::
   single: VHDL; signal assignment
   single: Verilog; non-blocking assignment

The clock driver function :func:`ClkDriver` has a clock signal as its parameter.
This is how a *port* is modeled in MyHDL. The function defines a generator that
continuously toggles a clock signal after a certain delay. A new value of a
signal is specified by assigning to its ``next`` attribute. This is the MyHDL
equivalent of  the VHDL signal assignment and the  Verilog non-blocking
assignment.

.. index:: single: wait; for a rising edge

The :func:`HelloWorld` function is modified from the first example. It now also
takes a clock signal as parameter. Its generator is made sensitive to a rising
edge of the clock signal. This is specified by the ``posedge`` attribute of a
signal. The edge specifier is the argument of the ``always`` decorator. As a
result, the decorated function will be executed on every rising clock edge.

The ``clk`` signal is constructed with an initial value ``0``. When creating an
instance of each  hardware module, the same clock signal is passed as the
argument. The result is that the instances are now connected through the clock
signal. The :class:`Simulation` object is constructed with the two instances.

When we run the simulation, we get::

   % python hello2.py
   10 Hello World!
   30 Hello World!
   50 Hello World!
   _SuspendSimulation: Simulated 50 timesteps


.. _intro-hier:

Parameters and hierarchy
========================

We have seen that MyHDL uses functions to model hardware modules. We have also
seen that ports are modeled by using signals as parameters. To make designs
reusable we will also want to use other objects as parameters. For example, we
can change the clock generator function to make it more general and reusable, by
making the clock period parametrizable, as follows::

   from myhdl import Signal, delay, instance, always, now, Simulation

   def ClkDriver(clk, period=20):

       lowTime = int(period/2)
       highTime = period - lowTime

       @instance
       def driveClk():
           while True:
               yield delay(lowTime)
               clk.next = 1
               yield delay(highTime)
               clk.next = 0

       return driveClk

In addition to the clock signal, the clock period is a parameter, with a default
value of ``20``.

.. index:: single: decorator; instance

As the low time of the clock may differ from the high time in case of an odd
period, we cannot use the :func:`always` decorator with a single delay value
anymore. Instead, the :func:`driveClk` function is now a generator function with
an explicit definition of the desired behavior. It is decorated with the
:func:`instance` decorator.  You can see that :func:`driveClk` is a generator
function because it contains ``yield`` statements.

When a generator function is called, it returns a generator object. This is
basically what the :func:`instance` decorator does. It is less sophisticated
than the :func:`always` decorator, but it can be used to create a generator from
any local generator function.

The ``yield`` statement is a general Python construct, but MyHDL uses it in a
dedicated way.  In MyHDL, it has a similar meaning as the wait statement in
VHDL: the statement suspends execution of a generator, and its clauses specify
the conditions on which the generator should wait before resuming. In this case,
the generator waits for a certain delay.

Note that to make sure that the generator runs "forever", we wrap its behavior
in a ``while True`` loop.

Similarly, we can define a general :func:`Hello` function as follows::

   def Hello(clk, to="World!"):

       @always(clk.posedge)
       def sayHello():
           print "%s Hello %s" % (now(), to)

       return sayHello

.. index:: single: instance; defined

We can create any number of instances by calling the functions with the
appropriate parameters. Hierarchy can be modeled by defining the instances in a
higher-level function, and returning them. This pattern can be repeated for an
arbitrary number of hierarchical levels. Consequently, the general definition of
a MyHDL instance is recursive: an instance is either a sequence of instances, or
a generator.


As an example, we will create a higher-level function with four instances of the
lower-level functions, and simulate it::

   def greetings():

       clk1 = Signal(0)
       clk2 = Signal(0)

       clkdriver_1 = ClkDriver(clk1) # positional and default association
       clkdriver_2 = ClkDriver(clk=clk2, period=19) # named association 
       hello_1 = Hello(clk=clk1) # named and default association
       hello_2 = Hello(to="MyHDL", clk=clk2) # named association

       return clkdriver_1, clkdriver_2, hello_1, hello_2


   inst = greetings()
   sim = Simulation(inst)
   sim.run(50)

As in standard Python, positional or named parameter association can be used in
instantiations, or a mix of both [#]_. All these styles are demonstrated in the
example above. Named association can be very useful if there are a lot of
parameters, as the argument order in the call does not matter in that case.

The simulation produces the following output::

   % python greetings.py
   9 Hello MyHDL
   10 Hello World!
   28 Hello MyHDL
   30 Hello World!
   47 Hello MyHDL
   50 Hello World!
   _SuspendSimulation: Simulated 50 timesteps

.. warning::

   Some commonly used terminology has different meanings in Python versus hardware
   design. Rather than artificially changing terminology, I think it's best to keep
   it and explicitly describing the differences.

   .. index:: single: module; in Python versus hardware design

   A :dfn:`module` in Python refers to all source code in a particular file. A
   module can be reused by other modules by importing. In hardware design, a module
   is  a reusable block of hardware with a well defined interface. It can be reused
   in  another module by :dfn:`instantiating` it.

   .. index:: single: instance; in Python versus hardware design

   An :dfn:`instance` in Python (and other object-oriented languages) refers to the
   object created by a class constructor. In hardware design, an instance is a
   particular incarnation of a hardware module.

   Normally, the meaning should be clear from the context. Occasionally, I may
   qualify terms  with the words "hardware" or "MyHDL" to  avoid ambiguity.

.. _intro-python:

Some remarks on MyHDL and Python
================================

To conclude this introductory chapter, it is useful to stress that MyHDL is not
a language in itself. The underlying language is Python,  and MyHDL is
implemented as a Python package called ``myhdl``. Moreover, it is a design goal
to keep the ``myhdl`` package as minimalistic as possible, so that MyHDL
descriptions are very much "pure Python".

To have Python as the underlying language is significant in several ways:

* Python is a very powerful high level language. This translates into high
  productivity and elegant solutions to complex problems.

* Python is continuously improved by some very clever  minds, supported by a
  large and fast growing user base. Python profits fully from the open source
  development model.

* Python comes with an extensive standard library. Some functionality is likely
  to be of direct interest to MyHDL users: examples include string handling,
  regular expressions, random number generation, unit test support, operating
  system interfacing and GUI development. In addition, there are modules for
  mathematics, database connections, networking programming, internet data
  handling, and so on.

* Python has a powerful C extension model. All built-in types are written with
  the same C API that is available for custom extensions. To a module user, there
  is no difference between a standard Python module and a C extension module ---
  except performance. The typical Python development model is to prototype
  everything in Python until the application is stable, and (only) then rewrite
  performance critical modules in C if necessary.


.. _intro-summary:

Summary and perspective
=======================

Here is an overview of what we have learned in this chapter:

* Generators are the basic building blocks of MyHDL models. They provide the way
  to model massive concurrency and sensitivity lists.

* MyHDL provides decorators that create useful generators from local functions.

* Hardware structure and hierarchy is described with classic Python functions.

* :class:`Signal` objects are used to communicate between concurrent generators.

* A :class:`Simulation` object is used to simulate MyHDL models.

These concepts are sufficient to start modeling and simulating with MyHDL.

However, there is much more to MyHDL. Here is an overview of what can be learned
from the following chapters:

* MyHDL supports hardware-oriented types that make it easier to write
  typical hardware models. These are described in Chapter :ref:`hwtypes`.

* MyHDL supports sophisticated and high level modeling techniques. This is
  described in Chapter :ref:`model-hl`.

* MyHDL enables the use of modern software verification techniques, such as unit
  testing, on hardware designs. This is the topic of Chapter :ref:`unittest`.

* It is possible to co-simulate MyHDL models with other HDL languages such as
  Verilog and VHDL. This is described in Chapter :ref:`cosim`.

* Last but not least, MyHDL models can be converted to Verilog, providing a path
  to a silicon implementation. This is the topic of Chapter :ref:`conv`.

.. rubric:: Footnotes

.. [#] The exception is the ``from module import *`` syntax, that imports all the
   symbols from a module. Although this is generally considered bad practice, it
   can be tolerated for large modules that export a lot of symbols. One may argue
   that ``myhdl`` falls into that category.

.. [#] All positional parameters have to go before any named parameter.

