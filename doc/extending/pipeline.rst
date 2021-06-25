
.. _pipeline:

====================================
Overview of the compilation pipeline
====================================

The purpose of this page is to explain each step of defining and
compiling an Aesara function.


Definition of the computation graph
-----------------------------------

By creating Aesara :ref:`Variables <variable>` using
``aesara.tensor.lscalar`` or ``aesara.tensor.dmatrix`` or by using
Aesara functions such as ``aesara.tensor.sin`` or
``aesara.tensor.log``, the user builds a computation graph. The
structure of that graph and details about its components can be found
in the :ref:`graphstructures` article.



Compilation of the computation graph
------------------------------------

Once the user has built a computation graph, they can use
:func:`aesara.function` in order to make one or more functions that
operate on real data. function takes a list of input :ref:`Variables
<variable>` as well as a list of output :class:`Variable`\s that define a
precise subgraph corresponding to the function(s) we want to define,
compile that subgraph and produce a callable.

Here is an overview of the various steps that are done with the
computation graph in the compilation phase:


Step 1 - Create a :class:`FunctionGraph`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The subgraph given by the end user is wrapped in a structure called
:class:`FunctionGraph`. That structure defines several hooks on adding and
removing (pruning) nodes as well as on modifying links between nodes
(for example, modifying an input of an :ref:`apply` node) (see the
article about :ref:`libdoc_graph_fgraph` for more information).

:class:`FunctionGraph` provides a method to change the input of an :class:`Apply` node from one
:class:`Variable` to another and a more high-level method to replace a :class:`Variable`
with another. This is the structure that :ref:`Optimizers
<optimization>` work on.

Some relevant :ref:`Features <libdoc_graph_fgraphfeature>` are typically added to the
:class:`FunctionGraph`, namely to prevent any optimization from operating inplace on
inputs declared as immutable.


Step 2 - Execute main :class:`Optimizer`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Once the :class:`FunctionGraph` is made, an :term:`optimizer` is produced by
the :term:`mode` passed to :func:`function` (the :class:`Mode` basically has two
important fields, :attr:`linker` and :attr:`optimizer`). That optimizer is
applied on the :class:`FunctionGraph` using its :meth:`Optimizer.optimize` method.

The optimizer is typically obtained through :attr:`optdb`.


Step 3 - Execute linker to obtain a thunk
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Once the computation graph is optimized, the :term:`linker` is
extracted from the :class:`Mode`. It is then called with the :class:`FunctionGraph` as
argument to produce a ``thunk``, which is a function with no arguments that
returns nothing. Along with the thunk, one list of input containers (a
:class:`aesara.link.basic.Container` is a sort of object that wraps another and does
type casting) and one list of output containers are produced,
corresponding to the input and output :class:`Variable`\s as well as the updates
defined for the inputs when applicable. To perform the computations,
the inputs must be placed in the input containers, the thunk must be
called, and the outputs must be retrieved from the output containers
where the thunk put them.

Typically, the linker calls the ``toposort`` method in order to obtain
a linear sequence of operations to perform. How they are linked
together depends on the Linker used. The :class:`CLinker` produces a single
block of C code for the whole computation, whereas the :class:`OpWiseCLinker`
produces one thunk for each individual operation and calls them in
sequence.

The linker is where some options take effect: the ``strict`` flag of
an input makes the associated input container do type checking. The
``borrow`` flag of an output, if ``False``, adds the output to a
``no_recycling`` list, meaning that when the thunk is called the
output containers will be cleared (if they stay there, as would be the
case if ``borrow`` was True, the thunk would be allowed to reuse--or
"recycle"--the storage).

.. note::

    Compiled libraries are stored within a specific compilation directory,
    which by default is set to ``$HOME/.aesara/compiledir_xxx``, where
    ``xxx`` identifies the platform (under Windows the default location
    is instead ``$LOCALAPPDATA\Aesara\compiledir_xxx``). It may be manually set
    to a different location either by setting :attr:`config.compiledir` or
    :attr:`config.base_compiledir`, either within your Python script or by
    using one of the configuration mechanisms described in :mod:`config`.

    The compile cache is based upon the C++ code of the graph to be compiled.
    So, if you change compilation configuration variables, such as
    :attr:`config.blas__ldflags`, you will need to manually remove your compile cache,
    using ``Aesara/bin/aesara-cache clear``

    Aesara also implements a lock mechanism that prevents multiple compilations
    within the same compilation directory (to avoid crashes with parallel
    execution of some scripts).

Step 4 - Wrap the thunk in a pretty package
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The thunk returned by the linker along with input and output
containers is unwieldy. :func:`aesara.function` hides that complexity away so
that it can be used like a normal function with arguments and return
values.