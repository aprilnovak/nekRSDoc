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

Converting an Existing Commericial Mesh to .re2 Format
------------------------------------------------------

The most general and flexible approach for creating a mesh is to use commercial meshing software
such as Cubit or Gmsh. After creating the mesh, it must be converted to the ``.re2`` binary format.
The following two sections describe how to convert Exodus and Gmsh meshes into ``.re2`` binary format
with scripts that ship with the :term:`Nek5000` dependency.

Converting an Exodus mesh
""""""""""""""""""""""""""""""""""

To convert from an Exodus format mesh
(for this case, named ``my_mesh.exo``) to the ``.re2`` format, use the ``exo2nek`` script:

.. code-block::

  user$ exo2nek

  Input (.exo) file name:
  my_mesh

While the ``.re2`` format supports both HEX8 and HEX20 elements, the ``exo2nek`` script
is currently limited to HEX20 elements. Therefore, all Exodus format meshes must be
generated with HEX20 elements. 

Example: Converting a Gmsh mesh
""""""""""""""""""""""""""""""""

To convert from a Gmsh format mesh (for this case, named ``my_mesh.msh``) to the
``.re2`` format, use the ``gmsh2nek`` script:

.. code-block::

  user$ gmsh2nek

  Enter mesh dimension: 3
  Input (.msh) file name: my_mesh

