Compiling nekRS
===============

This page describes how to build nekRS in a general sense, as well as providing
specific instructions for some of the more common HPC systems used by the nekRS
development team.

First, clone the repository from `github <https://github.com/Nek5000/nekRS>`__.
Next, set the ``NEKRS_HOME`` environment variable to a location in your file
system where you would like to place the executables and other build files.
For example, this can be:

.. code-block::

  user$ export NEKRS_HOME=$HOME/.local/nekrs

Then, be sure to add this directory to your path:

.. code-block::

  user$ export PATH=${NEKRS_HOME}:${PATH}

To avoid repeating these steps for every new shell, you may want to add these environment
variable settings in a ``.bashrc``.

Next, run the ``makenrs`` script in the main level of the repository as

.. code-block::

  user$ ./makenrs

You will be prompted as to whether you want to build the :term:`CUDA`, :term:`HIP`,
and :term:`OpenCL` backends. To avoid passing these settings from the standard input,
you can also run ``makenrs`` with boolean values following the ``makenrs`` command
that indicate the backend settings for each of these three backends. For example,
to build without :term:`CUDA`, :term:`HIP`, *and* :term:`OpenCL`, you can run:

.. code-block::

  user$ ./makenrs 0 0 0

Summit
------

Sawtooth
--------
