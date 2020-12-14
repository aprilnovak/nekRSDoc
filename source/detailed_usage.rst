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

   void UDF_Setup(nrs_t *nrs)
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

