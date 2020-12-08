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

* ``BOOMERAMG``: settings for the (optional) :term:`AMG` solver
* ``GENERAL``: generic settings for the simulation
* ``OCCA``: backend device settings
* ``PRESSURE``: settings for the pressure solution
* ``SCALARXX``: settings for the ``XX``-th scalar
* ``TEMPERATURE``: settings for the temperature solution
* ``VELOCITY``: settings for the velocity solution

Each of the keys and value types are now described for these sections. Default settings
are shown in parentheses. Note that nekRS currently does not have any checking on valid
input parameters in the ``.par`` file (though this will be added in the future) - please
be sure that you type these (key, value) pairs correctly. nekRS also does not have a
method to register all valid keys, so this user guide may quickly become out of date
unless developers are careful to keep the keys listed here up to date.

nekRS uses just-in-time compilation to allow the incorporation of user-defined functions
into program execution. These functions can be written to allow ultimate flexibility on
the part of the user to affect the simulation, such as to define custom fluid properties,
specify spatially-dependent boundary and initial conditions, and apply post-processing
operations. Some of the parameters in the sections can be overridden through the use of
user-defined functions - see, for example, the ``viscosity`` parameter than is a key in
the ``VELOCITY`` section. This parameter is used to set a constant viscosity, whereas
for variable-property simulations, a user-defined function will override the ``viscosity``
input parameter. A full description of these user-defined functions on the host and
device are described in Sections :ref:`User-Defined Host Functions (.udf)` and
:ref:`User-Defined Device Functions (.oudf)`. So, the description of valid (key, value)
pairs here does not necessarily imply that these parameters reflect the full capabilities
of nekRS.

``BOOMERAMG`` section
^^^^^^^^^^^^^^^^^^^^^

**coarsenType**

**interpolationType**

**iterations** *<int>*

**nonGalerkinTol**

**smootherType**

**strongThreshold** *<real>*

``GENERAL`` section
^^^^^^^^^^^^^^^^^^^

**cubaturePolynomialOrder**

**dealiasing** *<bool>*

**dt** *<real>*
  Time step size

**endTime** *<real>*
  Final time at which to end the simulation, if using ``stopAt = endTime``

**extrapolation** *oifs, subcycling*

**filterCutoffRatio** *<real>*

**filtering** *(none), explicit, hpfrtm*

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

**subcyclingOrder** *<int>*

**subCyclingSteps** *<int>*

**targetCFL** *<real>*
  The target :term:`CFL` number when using adaptive time stepping with ``variableDT = true``

**timeStepper** *bdf1, bdf2, bdf3, tombo1, tombo2, tombo3*
  If you select any of the :term:`BDF` options, the time integrator is internally set to
  the :term:`TOMBO` time integrator of equivalent order.

**variableDT** *<bool>*
  Whether to enable a variable time step size; at the moment, nekRS only allows a fixed
  time step size, so this parameter is unused.

**verbose**

**writeControl** *(timeStep), runTime*
  Method to use for the writing of output files, either based on a time step interval with
  *timeStep* or a simulated time interval with *runTime*

**writeInterval** *<real>*
  Output writing frequency, either in units of time steps for ``writeControl = timeStep`` or
  in units of simulation time for ``writeControl = runTime``.

``OCCA`` section
^^^^^^^^^^^^^^^^

**backend** *CUDA*

**deviceNumber** *LOCAL-RANK*

``PRESSURE`` section
^^^^^^^^^^^^^^^^^^^^

**amgSolver** *paralmond*

**downwardSmoother** *ASM, jacobi, RAS*

**galerkinCoarseOperator** *<bool>*

**pMultigridCoarsening**

**preconditioner** *jacobi, multigrid*

**residualProjection** *<bool>*

**residualProjectionStart** *<int>*

**residualProjectionVectors** *<int>*

**residualTol** *<real>*

**smootherType** *additive, asm, chebyshev, chebyshev+ras, chebyshev+asm, ras*

**upwardSmoother** *ASM, JACOBI, RAS*

``SCALARXX`` section
^^^^^^^^^^^^^^^^^^^^

This section is used to define the transport parameters and solver settings for each
passive scalar. For instance, in a simulation with two passive scalars, you would have
two sections - ``SCALAR01`` and ``SCALAR02``, each of which represents a passive scalar.

**boundaryTypeMap** *<char[]>*

**diffusivity** *<real>*
  Although this is named ``diffusivity``, this parameter really represents the conductivity
  governing diffusion of the passive scalar. In other words, the analogue from the
  ``TEMPERATURE`` section (a passive scalar in its internal representation) is the
  ``conductivity`` parameter. If a negative value is provided, the
  conductivity is internally set to :math:`1/|k|`, where :math:`k` is the value of the
  ``conductivity`` key.

**residualProjection** *<bool>*

**residualProjectionStart** *<int>*

**residualProjectionVectors** *<int>*

**residualTol** *<real>*

**rho** *<real>*
  Although this is name ``rho``, this parameter really represents the coefficient on the
  total derivative of the passive scalar. In other words, the analogue from the
  ``TEMPERATURE`` section (a passive scalar in its internal representation) is the
  ``rhoCp`` parameter.

``TEMPERATURE`` section
^^^^^^^^^^^^^^^^^^^^^^^

**boundaryTypeMap** *<string[]>*
  Array of strings describing the boundary condition to be applied to each sideset, ordered
  by sideset ID.

**conductivity** *<real>*
  Constant thermal conductivity; if a negative value is provided, the thermal conductivity
  is internally set to :math:`1/|k|`, where :math:`k` is the value of the ``conductivity``
  key.

**residualProjection** *<bool>*

**residualProjectionStart** *<int>*

**residualProjectionVectors** *<int>*

**residualTol** *<real>*

**rhoCp** *<real>*
  Constant volumetric isobaric specific heat.

**solver** *none*
  You can turn off the solution of temperature by setting the solver to ``none``

``VELOCITY`` section
^^^^^^^^^^^^^^^^^^^^

**boundaryTypeMap** *<char[]>*
  Array of strings describing the boundary condition to be applied to each sideset, ordered
  by sideset ID.

**density** *<real>*
  Constant fluid density

**residualProjection** *<bool>*

**residualProjectionStart** *<int>*

**residualProjectionVectors** *<int>*

**residualTol** *<real>*

**solver** *none*
  You can turn off the solution of the flow (velocity and pressure) by setting the solver
  to ``none``.

**viscosity** *<real>*
  Constant dynamic viscosity; if a negative value is provided, the dynamic viscosity is
  internally set to :math:`1/|\mu|`, where :math:`\mu` is the value of the ``viscosity`` key.

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
