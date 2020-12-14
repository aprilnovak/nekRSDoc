.. _input:

The nekRS Input Files
=====================

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

Note that each of the input files is described only in terms of its file extension (such as
``.par`` and ``.udf``). A full simulation "case" consists of all of these files with
a common prefix. For instance, in the ``nekRS/examples/eddyPeriodic`` directory, you will see
files named ``eddy.par``, ``eddy.re2``, ``eddy.udf``, and ``eddy.oudf``, where ``eddy`` is
referred to as the case "name." The only restrictions on the case name are:

1. It must be used as the prefix on all simulation files
2. Typical restrictions for naming files for your operating system

The scope of this page is merely to introduce the format and purpose of the four
files needed to set up a nekRS simulation. Much more detailed instructions are provided
on the :ref:`Detailed Usage <detailed>` page.

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
device are described in Sections :ref:`UDF Functions <udf_functions>` and
:ref:`OUDF Functions <oudf_functions>`. So, the description of valid (key, value)
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

The nekRS mesh file is provided in ``.re2`` format. This format can be
produced by either:

1. Converting a mesh made with commercial meshing software to ``.re2`` format
2. Directly creating an ``.re2``-format mesh with nekRS-specific scripts

There are three main limitations for the nekRS mesh:

1. nekRS is currently, and for the forseeable future, restricted to 3-D hexahedral meshes.
2-D and 1-D problems can be accommodated on these 3-D meshes by applying zero gradient
boundary conditions to all solution variables in directions perpendicular to the
simulation plane or line, respectively. All source terms and material properties in the
governing equations must therefore also be fixed in the off-interest directions.

2. nekRS assumes that the numeric IDs for the mesh boundaries are ordered contiguously
beginning from 1.

3. The ``.re2`` format only supports HEX8 (an eight-node
hexahedral element) and HEX20 (a twenty-node hexahedral element) elements.

The following two sections describe how to generate a mesh in ``.re2`` format.

Converting an Existing Commercial Mesh
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The most general and flexible approach for creating a mesh is to use commercial meshing software
such as Cubit or Gmsh. After creating the mesh, it must be converted to the ``.re2`` binary format. Depending
on the mesh format (such as Exodus II format or Gmsh format), a conversion script is used to
convert the mesh to ``.re2`` format.

:term:`Nek5000` is currently a dependency in the nekRS build system for some I/O tasks. Relevant
to the present meshing discussion, a number of scripts shipped with :term:`Nek5000` are used to
perform this mesh conversion process. These scripts are built as part of the nekRS build process
and are available at ``nekRS/build/_deps/nek5000_content-src/tools``, so no extra steps are
required on the user's end to access these scripts. You may want to add this directory to your
path to avoid typing out the full directory path for each usage.

See the :ref:`Converting a Mesh to .re2 Format <converting_mesh>` section for examples demonstrating conversion of Exodus
and Gmsh meshes into ``.re2`` format.

.. _nek5000_mesh:

Nek5000 Script-Based Meshing
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A number of meshing scripts ship with the :term:`Nek5000` dependency, which allow
you to directly create ``.re2`` format meshes without the need of commercial meshing
tools. These scripts, such as ``genbox``, take user input related to the desired
grid spacing to generate meshes for fairly simple geometries. Please consult the
:term:`Nek5000` documentation for more information on the use of these scripts.

Legacy Option (.rea)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The very early equivalent of the ``.par`` parameter file for :term:`Nek5000` was a file with extension
``.rea``. This file contained similar user settings for problem parameters that now are
set in the ``.par`` file, in addition to ASCII text format describing each node of the
mesh. See the ``Mesh File (.re2)`` section of the :term:`Nek5000`
`documentation <http://nek5000.github.io/NekDoc/problem_setup/case_files.html>`__ [#f1]_
for further details on the format for the ``.rea`` file.

The mesh section of the ``.rea`` file can be generated in two different manners -
either by specifying all the element nodes by hand, or with the :term:`Nek5000` mesh
generation scripts introduced in Section :ref:`Nek5000 Script-Based Meshing <nek5000_mesh>`.
Because the binary ``.re2`` format is preferred for very large meshes where memory may be
a concern, the ``.rea`` file approach is considered to be a legacy option.
The mesh portion of the legacy ``.rea``
file can be converted to the ``.re2`` format with the ``reatore2`` script, which also
ships with the :term:`Nek5000` dependency.

.. _udf_functions:

User-Defined Host Functions (.udf)
__________________________________

User-defined functions for the host are specified in the ``.udf`` file. These
functions can be used to perform virtually any action that can be programmed in C++.
Some of the more common examples are setting initial conditions, querying the solution
at regular intervals, and defining custom material properties and source terms. The
available functions that you may define in the ``.udf`` file are as follows. From the
examples shown on the :ref:`Detailed Usage <detailed>` page, you will see that usage
of these functions requires some proficiency in the C++
language as well as some knowledge of the nekRS source code internals.

``UDF_ExecuteStep(nrs_t* nrs, dfloat time, int tstep)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``UDF_LoadKernels(nrs_t*  nrs)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``UDF_Setup0(MPI_Comm comm, setupAide & options)``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This user-defined function is passed the nekRS :term:`MPI` communicator ``comm`` and a data
structure containing all of the user-specified simulation options, ``options``. This function is
called once at the beginning of the simulation *before* initializing the nekRS internals
such as the mesh, solvers, and solution data arrays. Because virtually no aspects of
the nekRS simulation have been initialized at the point when this function is called,
this function is primarily used to modify the user settings. For the typical user,
all relevant settings are already exposed through the ``.par`` file; any desired
changes to settings should therefore be performed by modifying the ``.par`` file.

This function is intended for developers or advanced users to overwrite any user
settings that may not be exposed to the ``.par`` file. For instance, setting
``timeStepper = tombo2`` in the ``GENERAL`` section triggers a number of other internal
settings in nekRS that do not need to be exposed to the typical user, but that perhaps
a developer may want to modify for testing purposes.

``UDF_Setup(nrs_t* nrs)``
^^^^^^^^^^^^^^^^^^^^^^^^^

This user-defined function is passed the nekRS simulation object ``nrs``. This function
is called once at the beginning of the simulation *after* initializing the mesh, solution
arrays, material property arrays, and boundary field mappings. This function is most
commonly used to:

1. Apply initial conditions to the solution
2. Assign function pointers to user-defined source terms and material properties

Any other additional setup actions that depend on initialization of the solution arrays
and mesh can of course also be placed in this function.

Specifying Initial Conditions
"""""""""""""""""""""""""""""

Initial conditions are specified by looping over all :term:`GLL` points and assigning
values based on the position and user-defined parameters. See the :ref:`Setting Initial Conditions <setting_ICs>`
section for an example on the use of this function for setting initial conditions.

Specifying Custom Source Terms and Properties
"""""""""""""""""""""""""""""""""""""""""""""

In addition to the ``UDF_Setup0``, ``UDF_Setup``, ``UDF_ExecuteStep``, and ``UDF_LoadKernels``
functions described in detail here, there are other user-defined functions. These functions
are handled in a slightly different manner - rather than be tied to a specific function name
like ``UDF_Setup0``, these functions are provided in terms of generic function pointers to
*any* function (provided the function parameters match those of the pointer). The four
function pointers are named as follows in nekRS:

================== ======================================================== ===================
Function pointer   Function signature                                       Purpose
================== ======================================================== ===================
``udf.uEqnSource`` ``f(nrs_t* nrs, float t, m o_U, m o_FU)``                momentum source
``udf.sEqnSource`` ``f(nrs_t* nrs, float t, m o_S, m o_SU)``                scalar source
``udf.properties`` ``f(nrs_t* nrs, float t, m o_U, m o_S, m o_Up, m o_Sp)`` material properties
``udf.div``        ``f(nrs_t* nrs, float t, m o_div)``                      thermal divergence
================== ======================================================== ===================

To shorten the syntax above, the type ``m`` is shorthand for ``occa::memory``, and ``f`` is the
name of the function, which can be *any* user-defined name. Other parameters that appear in the
function signatures are as follows:

* ``nrs`` is a pointer to the nekRS simulation object
* ``t`` is the current simulation time
* ``o_U`` is the velocity solution on the device
* ``o_S`` is the scalar solution on the device
* ``o_FU``
* ``o_SU``
* ``o_Up``
* ``o_Sp``
* ``o_div``

The ``udf.uEqnSource`` allows specification of a momentum source, such as a gravitational force, or
a friction form loss. The ``udf.sEqnSource`` allows specification of a source term for the passive
scalars. For a temperature passive scalar, this source term might represent a volumetric heat source,
while for a chemical concentration passive scalar, this source term could represent a mass
source.

The ``udf.properties`` allows specification of custom material properties for the flow,
which can be a function of the flow state as well as position and time. See the
:ref:`Setting Custom Properties <custom_properties>` section for an example of setting custom
properties.

Finally, ``udf.div``
allows specification of the thermal divergence term needed for the low Mach formulation.
See the :ref:`Detailed Usage <detailed>` page for an example of each of these use cases.

Legacy Option (.usr)
^^^^^^^^^^^^^^^^^^^^

.. _oudf_functions:

User-Defined Device Functions (.oudf)
_____________________________________

This file contains all user-defined functions that are to run on the device. These functions include
all functions used to apply boundary conditions that are built in to nekRS, as well as any other
problem-specific device functions.

Specifying Boundary Conditions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The type of condition to apply for each boundary is specified by the ``boundaryTypeMap`` parameter
in the ``.par`` file. However, this single line only specifies the *type* of boundary condition.
If that boundary condition requires additional information, such as a value to impose for
a Dirichlet velocity condition, or a flux to impose for a Neumann temperature condition, then
a device function must be provided in the ``.oudf`` file. The names of these functions, and the
types of boundary conditions for which they are called, are described as follows.

=========================================== ============= =======================================
Function                                    Character Map Purpose
=========================================== ============= =======================================
``pressureDirichletConditions(bcData* bc)``               Dirichlet condition for pressure
``scalarDirichletConditions(bcData* bc)``   ``t``         Dirichlet condition for passive scalars
``scalarNeumannConditions(bcData* bc)``     ``f``         Neumann condition for passive scalars
``velocityDirichletConditions(bcData* bc)`` ``v``         Dirichlet condition for velocity
``velocityNeumannConditions(bcData* bc)``                 Neumann condition for velocity
=========================================== ============= =======================================

Each function has the same signature, and takes as input the ``bc`` object. This object contains
all information needed to apply a boundary condition - the position, unit normals, and solution
components. The "character map" refers to the character in the ``boundaryTypeMap`` key in the
``.par`` file that will trigger this boundary condition. The ``scalar``-type boundary conditions
are called for boundary conditions on passive scalars, while the ``pressure``- and ``velocity``-type
conditions are called for the boundary conditions on the flow.

Each of these functions is *only* called on boundaries that contain that boundary. For instance,
if only boundaries 3 and 4 are primitive conditions on velocity, then ``velocityDirichletConditions``
is only called on boundaries 3 and 4. See the :ref:`Setting Boundary Conditions <boundary_conditions>`
section for several examples on how to set boundary conditions with device functions.

.. rubric:: Footnotes

.. [#f1] While the heading for ``Mesh File (.re2)`` seems to suggest that the contents refer only to the ``.re2`` format, the actual text description still points to the legacy ``.rea`` format.
