.. _detailed:

Detailed Code Usage
===================

This page describes how to perform a wide variety of user interactions with nekRS
for setting boundary conditions, converting between mesh formats, defining and
running device kernels, writing output files, and much more. Please first consult
the :ref:`Input File Syntax <input>` page for an overview of the purpose of each
nekRS input file to provide context on where the following instructions fit into
the overall code structure.

.. _converting_mesh:

Converting a Mesh to .re2 Format
--------------------------------

The most general and flexible approach for creating a mesh is to use commercial meshing software
such as Cubit or Gmsh. After creating the mesh, it must be converted to the ``.re2`` binary format.
The following two sections describe how to convert Exodus and Gmsh meshes into ``.re2`` binary format
with scripts that ship with the :term:`Nek5000` dependency.

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

.. _boundary_conditions:

Setting Boundary Conditions with Device Kernels
-----------------------------------------------

Initial conditions 

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
 
Suppose we would like to set :math:`\rho=1000.0` and :math:`\mu=2.1e-5 e^{-T/500}` for
the flow equations; :math:`\rho C_p=2e3(1000+\phi_0)` and :math:`k=2.5` for the first
passive scalar :math:`\phi_0`; and :math:`\rho C_p=0` and :math:`k=5+\phi_0` for the
second passive scalar :math:`\phi_1`. Our material property function would be as follows.

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

The ``viscosityCorrelationKernel``, ``constantFillKernel``, ``heatCapacityCorrelationKernel``,
and ``conductivityCorrelationKernel`` functions are all user-defined device kernels. These
functions must be defined in the ``.oudf`` file.

.. code-block:: cpp

   void material_props(nrs_t* nrs, dfloat time, occa::memory o_U, occa::memory o_S,
     occa::memory o_UProp, occa::memory o_SProp)
   {
     // viscosity and density for the flow equations
     const occa::memory o_mue = o_UProp.slice(0 * nrs->fieldOffset * sizeof(dfloat));
     viscosityCorrelationKernel(nrs->mesh->Nelements, o_mue);

     const occa::memory o_rho = o_UProp.slice(1 * nrs->fieldOffset * sizeof(dfloat));    
     constantFillKernel(nrs->mesh->Nelements, 1000.0, 0, nrs->o_elementInfo, o_rho);

     // conductivity and rhoCp for the first passive scalar
     int scalar_number = 0;
     const occa::memory o_con = o_SProp.slice((0 + 2 * scalar_number) *
       cds->fieldOffset * sizeof(dfloat));
     heatCapacityCorrelationKernel(nrs->mesh->Nelements, o_con);

     const occa::memory o_rhocp = o_SProp.slice((1 + 2 * scalar_number) *
       cds->fieldOffset * sizeof(dfloat));
     constantFillKernel(nrs->mesh->Nelements, 2.5, 0, nrs->o_elementInfo, o_rhocp);

     // conductivity and rhoCp for the second passive scalar
     scalar_number = 1;
     const occa::memory o_con_2 = o_SProp.slice((0 + 2 * scalar_number) *
       cds->fieldOffset * sizeof(dfloat));
     constantFillKernel(nrs->mesh->Nelements, 0.0, 0, nrs->o_elementInfo, o_con_2);

     const occa::memory o_rhocp_2 = o_SProp.slice((1 + 2 * scalar_number) *
       cds->fieldOffset * sizeof(dfloat));
     conductivityCorrelationKernel(nrs->mesh->Nelements, o_rhocp_2);
   }



