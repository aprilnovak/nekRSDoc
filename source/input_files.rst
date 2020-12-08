Input File Syntax
=================

This page describes the input file structure and syntax needed to run a nekRS simulation.
At a minimum, nekRS requires four files to run a problem - 

* Parameter file, with ``.par`` extension
* Mesh file, with ``.re2`` extension
* User-defined functions for the host, with ``.udf`` extension
* User-defined functions for the device, with ``.oudf`` extension

The next four sections describe the structure and syntax for each of these four files.
Because the :term:`Nek5000` code is somewhat of a predecessor to
nekRS, some aspects of the current nekRS input file design are selected to enable faster translation of
:term:`Nek5000` input files into nekRS input files. All such
:term:`Nek5000`-oriented settings will be referred to as "legacy" settings. Because these
legacy settings require proficiency in Fortran, the formation of several additional input
files, and in some cases, careful usage of structured text inputs, all new
users are encouraged to adopt the nekRS-based problem setup.

Parameter File (.par)
_____________________

Most information about the problem setup is defined in the parameter file. This file is organized
in a number of sections, each with a number of keys. Values are assigned to these keys in order to
control a number of aspects of the simulation.

The general structure of the ``.par`` file is as
follows, where ``FOO`` and ``BAR`` are both section names, with a number of (key, value) pairs.

.. code-block:: xml

  [FOO]
    key = value
    baz = bat

  [BAR]
    alpha = beta

The valid sections for setting up a nekRS simulation are

* ``BOOMERAMG``: 
* ``GENERAL``: generic settings
* ``OCCA``: backend device settings
* ``PRESSURE``: settings for the pressure solution
* ``SCALARXX``: settings for the ``XX``-th scalar
* ``TEMPERATURE``: settings for the temperature solution
* ``VELOCITY``: settings for the velocity solution

Each of the keys and value types are now described for these sections. Default settings
are shown in parentheses.

In the ``GENERAL`` section, valid (key, value) pairs are as follows.

**dt** *<real>*
  Time step size

**endTime** *<real>*
  Final time at which to end the simulation, if using ``stopAt = endTime``

**extrapolation**

**filtering** *(none), explicit, hpfrt*

**filterModes** *<int>*

**filterWeight** *<real>*

**numSteps** *<int>*
  Number of time steps to perform, if using ``stopAt = numSteps``

**polynomialOrder** *<int>*
  Polynomial order for the spectral element solution. An order of :math:`N` will result
  in :math:`N+1` basis functions for each spatial dimension.

**startFrom** *<string>*
  Absolute or relative path to a nekRS output file from which to start the simulation from.
  If the solution in the restart file was obtained with a different polynomial order,
  interpolation is performed to the current simulation settings. If this is omitted, the
  simulation is assumed to start based on the user-defined initial conditions at time zero.

**stopAt** *(numSteps), endTime*
  When to stop the simulation, either based on a number of time steps *numSteps* or a simulated
  end time *endTime*

**stressFormulation** *<bool>*

**subCyclingSteps** *<int>*

**timeStepper** *tombo2, tombo3*

**writeControl** *(timeStep), runTime*
  Method to use for the writing of output files, either based on a time step interval with
  *timeStep* or a simulated time interval with *runTime*

**writeInterval** *<real>*
  Output writing frequency, either in units of time steps for ``writeControl = timeStep`` or
  in units of simulation time for ``writeControl = runTime``.


Legacy Option (.rea)
^^^^^^^^^^^^^^^^^^^^

Mesh File (.re2)
________________

User-Defined Host Functions (.udf)
__________________________________

Legacy Option (.usr)
^^^^^^^^^^^^^^^^^^^^

User-Defined Device Functions (.oudf)
_____________________________________
