.. _detailed:

Detailed Code Usage
===================

This page describes how to perform a wide variety of user interactions with nekRS
for setting boundary conditions, converting between mesh formats, defining and
running device kernels, writing output files, and much more. Please first consult
the :ref:`Input File Syntax <input>` page for an overview of the purpose of each
nekRS input file to provide context on where the following instructions fit into
the overall code structure. Throughout this section, variables and data structures
in the nekRS source code are referenced - a list defining these variables and structures
is available on the :ref:`Commonly Used Variables <commonly_used_variables>` page
for reference.

.. _scripts:

Building the Nek5000 Tool Scripts
---------------------------------

Some user actions in nekRS require the use of scripts that ship with the :term:`Nek5000`
dependency. To build these scripts, you must first compile nekRS so that the Nek5000
dependency is pulled into the repository. Then, navigate to the ``tools`` directory
and run the makefile to compile all the scripts.

.. code-block::

  user$ cd build/_deps/nek5000_content-src/tools
  user$ ./maketools all

This should create binary executables in the ``build/_deps/nek5000_content-src/tools/bin``
directory. You may want to add this to your path in order to quickly access those scripts.

.. _converting_mesh:

Converting a Mesh to .re2 Format
--------------------------------

The most general and flexible approach for creating a mesh is to use commercial meshing software
such as Cubit or Gmsh. After creating the mesh, it must be converted to the ``.re2`` binary format.
The following two sections describe how to convert Exodus and Gmsh meshes into ``.re2`` binary format
with scripts that ship with the Nek5000 dependency. First build these scripts following
the instructions in the :ref:`Building the Nek5000 Tool Scripts <scripts>` section.

Converting an Exodus mesh
"""""""""""""""""""""""""

To convert from an Exodus format mesh
(for this case, named ``my_mesh.exo``) to the ``.re2`` format, use the ``exo2nek`` script:

.. code-block::

  user$ exo2nek

  Input (.exo) file name:
  my_mesh

While the ``.re2`` format supports both HEX8 and HEX20 elements, the ``exo2nek`` script
is currently limited to HEX20 elements. Therefore, all Exodus format meshes must be
generated with HEX20 elements. 

Converting a Gmsh mesh
""""""""""""""""""""""

To convert from a Gmsh format mesh (for this case, named ``my_mesh.msh``) to the
``.re2`` format, use the ``gmsh2nek`` script:

.. code-block::

  user$ gmsh2nek

  Enter mesh dimension: 3
  Input (.msh) file name: my_mesh

.. _cht_mesh:

Creating a Mesh for Conjugate Heat Transfer
-------------------------------------------

Mesh generation for conjugate heat transfer requires an additional pre-processing
step before performing other steps of the mesh generation process such as those
described in the :ref:`Converting a Mesh to .re2 Format <converting_mesh>` section.
The nekRS approach for conjugate heat transfer is still dependent on legacy limitations
from Nek5000. Unfortunately, you cannot
simply use a standard commercial meshing tool and define fluid and solid
regions according to block IDs - you must individually create the mesh for the fluid and
the solid, and then merge them with the ``pretex`` script.


.. _setting_ICs:

Setting Initial Conditions with ``UDF_Setup``
---------------------------------------------

This section provides an example for setting initial conditions with the
``UDF_Setup`` user-defined function that was introduced on the :ref:`Input Files <input>` page.
The following code snippet sets initial conditions for all three components of
velocity, the pressure, and two passive scalars. You may not necessarily have all of these
variables in your model - this example is just intended to cover all possibilities.

For this example, the initial conditions are
:math:`V_x=sin(x)cos(y)cos(z)`, :math:`V_y=-cos(x)sin(y)cos(z)`, and :math:`V_z=0`
for the three components of velocity;
:math:`P=101325` for the pressure; and :math:`\phi_0=573` and :math:`\phi_1=100+z` for the
two passive scalars indicated generically as :math:`\phi_0` and :math:`\phi_1`.

.. note::

  If present, the temperature variable is represented internally in nekRS as a passive
  scalar, since the form of the equation is the same as those solver for other passive
  scalars such as chemical concentration.

Because these initial conditions will
be a function of space, we must first obtain the mesh information, for which we
use the ``nrs->mesh`` pointer. All solution fields are stored in nekRS in terms of the
quadrature points (also referred to as the :term:`GLL` points). So, we will apply
the initial conditions by looping over all of these quadrature points, which for
the current :term:`MPI` process is equal to ``mesh->Np``, or the number of quadrature
points per element, and ``mesh->Nelements``, the number of elements on this process.

Next, we can get the :math:`x`, :math:`y`, and :math:`z` coordinates for the current
quadrature point with the ``x``, ``y``, and ``z`` pointers on the ``mesh`` object.
Finally, we programmatically set initial conditions for the solution fields. ``nrs->U``
is a single array that holds all three components of velocity; the ``nrs->fieldOffset``
variable is used to shift between components in this array. ``nrs->P`` represents the
pressure. Finally, ``nrs->S`` is a single array that holds all of the passive scalars.
Similar to the offset performed to index into the velocity array, the
``nrs->cds->fieldOffset`` variable is used to shift between components in the ``nrs->S``
array.

.. code-block:: cpp

   void UDF_Setup(nrs_t* nrs)
   {
    mesh_t* mesh = nrs->mesh;
    int num_quadrature_points = mesh->Np * mesh->Nelements;

    for (int n = 0; n < num_quadrature_points; n++) {
      float x = mesh->x[n];
      float y = mesh->y[n];
      float z = mesh->z[n];

      nrs->U[n + 0 * nrs->fieldOffset] = sin(x) * cos(y) * cos(z);
      nrs->U[n + 1 * nrs->fieldOffset] = -cos(x) * sin(y) * cos(z);
      nrs->U[n + 2 * nrs->fieldOffset] = 0;

      nrs->P[n] = 101325.0;

      nrs->S[n + 0 * nrs->cds->fieldOffset] = 573.0;
      nrs->S[n + 1 * nrs->cds->fieldOffset] = 100.0 + z;
    }
   }

.. _grabbing_user:

Grabbing User .par Settings
---------------------------

nekRS conveniently allows the user to define their own parameters in the ``.par`` file
that can then be accessed in the ``.udf`` functions. This is useful for programmatically
setting boundary conditions, forcing terms, and many other simulation settings. For instance,
suppose that the inlet velocity will vary from run to run and is possibly used in several
places in the ``.udf`` functions. Rather than continually edit the ``.udf`` file (which
will require repeated just-in-time compilation), these settings can be set with user-defined
parameters in the ``.par`` file.

As an example, we will define a parameter named ``inletVelocity`` in the ``VELOCITY`` block.

.. code-block :: xml

   [VELOCITY]
     residualTol = 1e-6
     density = 1.5
     viscosity = 2.4e-4
     boundaryTypeMap = inlet, wall, wall, wall, wall, outlet
     inletVelocity = 1.5

To access this value in the ``.udf`` functions, call the ``extract(String key, String value, T & destination)``
function on ``nrs->par`` as follows.

.. code-block :: cpp

   void UDF_Setup(nrs_t* nrs)
   {
     double inlet_Vz;
     nrs->par->extract("velocity", "inletVelocity", inlet_Vz);

     mesh_t* mesh = nrs->mesh;
     int num_quadrature_points = mesh->Np * mesh->Nelements;

     for (int n = 0; n < num_quadrature_points; n++) {
       nrs->U[n + 0 * nrs->fieldOffset] = 0;
       nrs->U[n + 1 * nrs->fieldOffset] = 0;
       nrs->U[n + 2 * nrs->fieldOffset] = inlet_Vz;
     }
   }

The extracted user parameter can then be used throughout the ``.udf`` functions, as well
as propagated to the device kernels as described in Section
:ref:`Defining Variables to Access in Device Kernels <defining_variables_for_device>`.

.. _defining_variables_for_device:

Defining Variables to Access in Device Kernels
----------------------------------------------

The customization of a nekRS problem to a specific case is one with both the host-side
user functions in the ``.udf`` file, as well as device-side user functions in the ``.oudf``
file. For convenience purposes, nekRS supports setting non-pointer-type variables in the
``.udf`` file that are accessible in the device kernels in the ``.oudf`` file. This section
shows an example of this usage.

Suppose that a device kernel requires a parameter representing a pressure gradient, which
is then used to determine a forcing kernel. One option would be to pass that pressure gradient
to the device kernel through its function parameters. The kernel in the ``.oudf`` file
would look something like the following.

.. code-block::

    @kernel void myForcingKernel(const dfloat dp_dx, /* more parameters */)
    {
      double foo = 2 * dp_dx;

      // do something
    }

Alternatively, we can define a variable, ``p_dp_dx``, that we set from the ``.udf`` file.
While this variable propagation can be done in any of the user-defined functions that
has ``nrs`` as an input parameter, for consistency purposes we will use the ``UDF_LoadKernels``
function for this purpose.

.. note:

  The convention is to precede any of these host-side kernel variable
  definitions with a ``p_``.

To set ``p_dp_dx`` to 5.5 from the ``.udf`` file, write to the ``kernelInfo`` object
on the ``nrs`` object. The ``defines/<p_name>`` syntax indicates that a variable on
the device is being declared with a name ``p_name`` that will be accessible simply as
``p_name`` in the device kernels.

.. code-block::

   void UDF_LoadKernels(nrs_t * nrs)
   {
     occa::properties & kernelInfo = *nrs->kernelInfo;

     kernelInfo["defines/p_dp_dx"] = 5.5;

     // other stuff related to loading the kernels
   }

Then, the kernel would be simplified to the following. You will note that nothing needs
to be passed through the kernel function arguments - ``p_dp_dx`` is simply available as
if it were a local variable to the function.

.. code-block:: cpp

   @kernel void myForcingKernel(/* more parameters */)
   {
     double foo = 2 * p_dp_dx;

     // do something
   }

If you grep for ``kernelInfo["defines`` in the nekRS source code, you will see that
this variable propagation features is also used extensively throughout a normal problem
setup. For instance, the number of velocity fields to solve for is propagated to the device
in the ``nrsSetup`` function.

.. code-block:: cpp

   nrs_t* nrsSetup(MPI_Comm comm, occa::device device, setupAide &options, int buildOnly)
   {
     // ...

     kernelInfo["defines/p_NVfields"] = nrs->NVfields;

     // ...
   }

Again, the convention is to precede all such propagated variables with the ``p_`` prefix.
No list of all such variables propagated automatically within a nekRS simulation is
maintained, so always check if the information you'd like to propagate is perhaps
already automatically propagated.

.. _boundary_conditions:

Setting Boundary Conditions with Device Kernels
-----------------------------------------------

Because all nekRS solves are performed on the device, boundary conditions on the
solution (which may change from time step to time step and be arbitrary functions
of the solution itself) are also applied on the device. The types of boundary conditions
on each solution field are specified in the ``.par`` file with the ``boundaryTypeMap``
key. 

.. _custom_properties:

Setting Custom Properties with ``UDF_Setup``
--------------------------------------------

Custom material properties can be set for the flow and passive scalar equations
by assigning the ``udf.properties`` function pointer to a function with a signature
that takes the ``nrs`` pointer to the nekRS solution object, the simulation time
``time``, the velocity solution on the device ``o_U``, the passive scalar solution
on the device ``o_S``, the flow material properties on the device ``o_UProp``,
and the passive scalar material properties on the device ``o_SProp``.

This section provides an example of setting :math:`\mu` and :math:`\rho` for the flow
equations and :math:`k` and :math:`\rho C_p` for two passive scalars. Suppose our problem
contains velocity, pressure, temperature, and two passive scalars. The ``[VELOCITY]``,
``[PRESSURE]``, ``[TEMPERATURE]``, ``[SCALAR01]``, and ``[SCALAR02]`` sections of the
``.par`` file would be as follows. Because we will be setting custom properties for
the pressure, velocity, and first two passive scalars (temperature and ``SCALAR01``),
we can let nekRS assign the default values of unity to all properties for those
governing equations until we override them in our custom property function. We still
need to define the material properties for ``SCALAR02``, however, because we will not
be overriding those properties in our function.

.. code-block::

  [PRESSURE]
  residualTol = 1e-6

  [VELOCITY]
  boundaryTypeMap = v, O, W
  residualTol = 1e-8

  [TEMPERATURE]
  boundaryTypeMap = t, O, I
  residualTol = 1e-8

  [SCALAR01]
  boundaryTypeMap = t, O, I
  residualTol = 1e-8

  [SCALAR02]
  boundaryTypeMap = t, O, t
  residualTol = 1e-7
  conductivity = 3.5
  rhoCp = 2e5

Also suppose that our problem contains conjugate heat transfer, such that some of
the mesh is fluid while some of the mesh is solid.

In ``UDF_Setup``, we next need to assign an address to the ``udf.properties`` function
pointer to a function with the correct signature where we eventually assign our custom
properties. Our ``UDF_Setup`` function would be as follows.

.. code-block:: cpp

   void UDF_Setup(nrs_t* nrs)
   {
     udf.properties = &material_props;
   }

Here, ``material_props`` is our name for a function in the ``.udf`` file that sets the
material properties. Its name is arbitrary, but it must have the following signature.

.. code-block:: cpp

   void material_props(nrs_t* nrs, dfloat time, occa::memory o_U, occa::memory o_S,
     occa::memory o_UProp, occa::memory o_SProp)
   {
     // set the material properties
   }
 
This function is called *after* the solve has been performed on each time step, so the
material properties are lagged by one time step with respect to the simulation.
 
Suppose we would like to set :math:`\rho=1000.0` and :math:`\mu=2.1e-5 e^{-\phi_0/500}(1+z)` for
the flow equations; because only the fluid domain has flow, we do not need to set
these properties on the solid part of the domain. For the first passive scalar
:math:`\phi_0`, we would
like to set :math:`(\rho C_p)_f=2e3(1000+PV_x)` and :math:`k_f=2.5` in the fluid
domain, and :math:`(\rho C_p)_s=2e3(1000+PV_x)` and :math:`k_s=3.5` in the solid domain.
Here, :math:`P` is the thermodynamic pressure and :math:`V_x` is the :math:`x`-component velocity.
For the second passive scalar :math:`\phi_1`, we would like to set
:math:`\rho C_p=0` and :math:`k=5+\phi_0` in both the fluid and solid domains.
Our material property function would be as follows. Note that these boundary conditions
are selected just to be comprehensive and show all possible options for setting
constant and non-constant properties with dependencies on properties - they do not
necessarily represent any realistic physical case.

.. code-block:: cpp

   // declare all the kernels we will be writing
   static occa::kernel viscosityKernel;
   static occa::kernel constantFillKernel;
   static occa::kernel heatCapacityKernel;
   static occa::kernel conductivityKernel;

   void material_props(nrs_t* nrs, dfloat time, occa::memory o_U, occa::memory o_S,
     occa::memory o_UProp, occa::memory o_SProp)
   {
     mesh_t* mesh = nrs->mesh;

     // viscosity and density for the flow equations
     const occa::memory o_mue = o_UProp.slice(0 * nrs->fieldOffset * sizeof(dfloat));
     const occa::memory first_scalar = o_S.slice(0 * cds->fieldOffset * sizeof(dfloat));
     viscosityKernel(mesh->Nelements, first_scalar, mesh->o_z, o_mue);

     const occa::memory o_rho = o_UProp.slice(1 * nrs->fieldOffset * sizeof(dfloat));    
     constantFillKernel(nrs->mesh->Nelements, 1000.0, 0.0 /* dummy */, nrs->o_elementInfo, o_rho);

     // conductivity and rhoCp for the first passive scalar
     int scalar_number = 0;
     const occa::memory o_con = o_SProp.slice((0 + 2 * scalar_number) *
       cds->fieldOffset * sizeof(dfloat));
     constantFillKernel(mesh->Nelements, 2.5, 3.5, nrs->o_elementInfo, o_con);

     const occa::memory o_rhocp = o_SProp.slice((1 + 2 * scalar_number) *
       cds->fieldOffset * sizeof(dfloat));
     heatCapacityKernel(mesh->Nelements, o_U, nrs->o_P, o_rhocp);

     // conductivity and rhoCp for the second passive scalar
     scalar_number = 1;
     const occa::memory o_con_2 = o_SProp.slice((0 + 2 * scalar_number) *
       cds->fieldOffset * sizeof(dfloat));
     conductivityKernel(mesh->Nelements, first_scalar, o_con_2);

     const occa::memory o_rhocp_2 = o_SProp.slice((1 + 2 * scalar_number) *
       cds->fieldOffset * sizeof(dfloat));
     constantFillKernel(mesh->Nelements, 0.0, 0.0, nrs->o_elementInfo, o_rhocp_2);
   }

The ``o_UProp`` and ``o_SProp`` arrays hold all material
property information for the flow equations and passive scalar equations, respectively.
In this function, you see six "slice" operations performed on ``o_UProp`` and ``o_SProp``
in order to access the two individual properties (diffusive constant and time derivative constant)
for the three equations (momentum, scalar 0, and scalar 1). The diffusive constant
(:math:`\mu` for the momentum equations and :math:`k` for the passive scalar equations)
is always listed first in these arrays, while the coefficient on the time derivative
(:math:`\rho C_p` for the momentum equations and :math:`\rho C_p` for the passive scalar
equations) is always listed second in these arrays.

To further elaborate, :math:`\mu` and :math:`\rho` are accessed as slices on ``o_UProp``.
Because viscosity is listed before density, the offset in the ``o_UProp`` array to get
the viscosity is zero, while the offset to get the density is ``nrs->fieldOffset``.
:math:`k` and :math:`\rho C_p` are accessed as slices in ``o_SProp``. Because the
passive scalars are listed in order and the conductivity is listed first for each user,
the offset in the ``o_SProp`` array to get the conductivity for the first passive scalar
is zero, while the offset to get the heat capacity for the first passive scalar 
is ``cds->fieldOffset``. Finally, the offset in the ``o_SProp`` array to get the conductivity
for the second passive scalar is ``2 * cds->fieldOffset``, while the offset to get the
heat capacity for the second passive scalar is ``3 * cds->fieldOffset``.

The ``viscosityKernel``, ``constantFillKernel``, ``heatCapacityKernel``,
and ``conductivityKernel`` functions are all user-defined device kernels. These
functions must be defined in the ``.oudf`` file, and the names are arbitrary. For each
of these kernels, we declare them at the top of the ``.udf`` file. In order to link
against our device kernels, we must instruct nekRS to use its just-in-time compilation
to build those kernels. We do this in ``UDF_LoadKernels`` by calling the
``udfBuildKernel`` function for each kernel. The second argument to the ``udfBuildKernel``
function is the name of the kernel, which appears as the actual function name of
the desired kernel in the ``.oudf`` file.

.. code-block:: cpp

  void UDF_LoadKernels(nrs_t* nrs)
  {
    viscosityKernel = udfBuildKernel(nrs, "viscosity");
    constantFillKernel = udfBuildKernel(nrs, "constantFill");
    heatCapacityKernel = udfBuildKernel(nrs, "heatCapacity");
    conductivityKernel = udfBuildKernel(nrs, "conductivity");
  }

In order to write these device kernels, you will need some background in programming
with :term:`OCCA`. Please consult the `OCCA documentation <https://libocca.org/#/>`__
before proceeding [#f1]_.

First, let's look at the ``constantFill`` kernel. Here, we want to write a device kernel
that assigns a constant value to a material property. So that we can have a general
function, we will write this such that it can be used to set constant (but potentially
different) properties in the fluid and solid phases for conjugate heat transfer
applications.

.. note::
  
  Material properties for the flow equations (i.e. viscosity and density) do not
  *need* to be specified in the solid phase. If you define flow properties in solid
  regions, they are simply not used.

The ``constantFill`` kernel is defined in the ``.oudf`` file as follows [#f2]_. :term:`OCCA`
kernels operate on the device. As input parameters, they can take non-pointer objects
on the host (such as ``Nelements``, ``fluid_val``, and ``solid_val`` in this example),
as well as pointers to objects of type ``occa::memory``, or device-side memory. The
device-side objects are indicated with the ``@restrict`` tag. 

.. note::

  Device-side memory in nekRS is by convention preceded with a ``o_`` prefix in order
  to differentiate from the host-side objects. In the initialization of nekRS, most of
  the simulation data is copied over to the device. All calculations are done on the
  device. The device-side solution is then only copied back onto the host for the
  purpose of writing output files.

.. warning::

  Because nekRS by default only copies the device-side solution back to the host for
  the purpose of writing output files, if you touch any host-side objects in your
  user-defined functions, such as in ``UDF_ExecuteStep``, you must ensure
  that you only use the host-side objects after they have been copied from device back
  to the host. Otherwise, they would not be "up to date." You can ensure that the host-
  side objects reflect the real-time nekRS solution by either (a) only touching the
  host-side solution on output writing steps (which can be determined based on the
  ``nrs->isOutputStep`` variable), or (b) calling the appropriate routines in nekRS
  to force data to be copied from the device back to the host. For the latter option,
  please refer to the :ref:`Copying From Device to Host <copy_device_to_host>` section.

For this example, we
loop over all the elements. The ``eInfo`` parameter represents a mask, and takes a value
of zero for solid elements and a value of unity for fluid elements. Next, we loop over
all of the :term:`GLL` points on the element, or ``p_Np``. This variable is set within
nekRS to be the same as ``mesh->Np`` using the device variable feature described in
the :ref:`Defining Variables to Access in Device Kernels <defining_variables_for_device>`
section. This particular variable is always available, and you do not need to pass it
explicitly into device functions. Finally, we set the value of the ``property`` to the
value specified in the function parameters.

.. code-block:: cpp

   @kernel void constantFill(const dlong Nelements, const dfloat fluid_val,
             const dfloat solid_val, @restrict const dlong* eInfo, @restrict dfloat* property)
   {
     for (dlong e = 0; e < Nelements; ++e ; @outer(0))
     {
       const bool is_solid = eInfo[e];

       for (int n = 0; n < p_Np; ++n ; @inner(0))
       {
         const int id = e * p_Np + n;

         property[id] = fluid_val;

         if (is_solid)
           property[id] = solid_val;
       }
     }
   }

Now, let's look at the slightly more complex ``conductivity`` kernel. Here, our function
signature is very different from that of the ``constantFill`` kernel. While we still
pass the number of elements, we no longer need to check whether we are in a fluid element
or a solid element, since the conductivity for the second passive scalar is going to be
the same in both phases. All that we need to pass in is the coupled scalar ``scalar``, 
or :math:`\phi_0` in our material property correlation :math:`k=5+\phi_0` that we listed
earlier. The ``property`` passed in then should represent the conductivity we are setting.

.. code-block:: cpp

  @kernel void conductivity(const dlong Nelements, @restrict const dfloat* scalar,
            @restrict dfloat* property)
  {
     for (dlong e = 0; e < Nelements; ++e ; @outer(0))
     {
       for (int n = 0; n < p_Np; ++n ; @inner(0))
       {
         const int id = e * p_Np + n;
         const dfloat scalar = scalar[id];

         property[id] = 5.0 + scalar;
       }
     }
  }

A key aspect of writing device kernels is that the device kernel can only operate on
non-pointer objects or pointers to device memory. Whatever the form of your material properties,
you just need to be sure to pass in all necessary information. Now, let's look at the even
more complex ``viscosity`` kernel. Here, we need to pass in the scalar :math:`\phi_0` and the
:math:`z`-coordinate that appear in the viscosity model.

.. code-block:: cpp

  @kernel void viscosity(const dlong Nelements, @restrict const dfloat* scalar,
            @restrict const dfloat* z, @restrict dfloat* property)
  {
     for (dlong e = 0; e < Nelements; ++e ; @outer(0))
     {
       for (int n = 0; n < p_Np; ++n ; @inner(0))
       {
         const int id = e * p_Np + n;
         const dfloat scalar = scalar[id];
         const dfloat z = z[id];

         property[id] = 2.1E-5 * exp(-scalar / 500.0) * (1.0 + z);
       }
     }
  }

The final kernel that wraps up this example is the ``heatCapacity`` kernel.

.. _copy_device_to_host:

Copying From Device to Host
---------------------------

All solutions take place on the host, and data transfer of the solution back to the host
(where it can be accessed for writing output files or other tasks such as postprocessing
functions in the ``.udf`` file functions) is only performed when nekRS is about to write
an output file. The writing of output in nekRS is controlled by settings in the ``.par``
file. For instance, if ``writeControl = numSteps``, and ``writeInterval = 100``, then the solution
on the device is copied to the host once every 100 time steps to ensure that the
ensuing output file writing is performed with the most "up-to-date" solution. This is
important to consider, because attempting to access the solution arrays on the ``nrs``
struct will be "out-of-date" if they are read at, say, time step 88.

If you would like to access the solution on intervals that do not match the output file
writing interval, then you need to explicitly copy the device solution back to the host.
This is done with the ``nek_ocopyFrom(double time, int tstep)`` routine in the
``nekInterfaceAdapter.cpp`` file. This function copies
the nekRS solution from the nekRS device arrays to the nekRS host arrays - that is,
``nrs->o_U`` is copied to ``nrs->U``, and so on. This
allows you to access the solution on the host as ``nrs->U``, ``nrs->p``, ``nrs->S``, etc.

.. _writing_output:

Writing an Output File
----------------------

nekRS will automatically write output files according to the ``writeControl`` criterion
set in the ``.par`` file. However, it may be desirable to have finer-grained control of
output writing, such as if you want the solution at a specific time step, but that
time step is not an integer multiple of ``writeInterval``. In this case, you can force
the output file writing to occur by calling the ``outfld(double time, double outputTime)``
function in the ``nekrs`` namespace. This function performs the following actions:

1. Copy the nekRS solution from the nekRS device arrays directly to the backend
   Nek5000 arrays.
2. Write an output file.

Note that this function is slightly different from the ``nek_ocopyFrom`` function described
in the :ref:`Copying Device to Host <copy_device_to_host>` section. This function is
solely intended for writing output, so no effort is expended in copying the device
solution into the nekRS host arrays - that step is bypassed, and the device solution is
copied straight into the Nek5000 backend arrays. The ``nek_ocopyFrom`` routine should really
only be used if you require access to the nekRS solution arrays on the host, while the
``outfld`` routine should be used strictly for writing output files.

.. rubric:: Footnotes

.. [#f1] There are many different ways to write :term:`OCCA` kernels. The examples shown here are by no means the most optimal form, and are only intended for illustration.
.. [#f2] :term:`OCCA` kernels are programmed in OKL, a thin extension to C++. Unfortunately, the ``pygmentize`` Python syntax highlighter does not recognize OKL syntax, so these examples here lack syntax highlighting.

