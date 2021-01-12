.. _commonly_used_variables:

Commonly-Used Variables in nekRS
================================

To become a proficient user of nekRS requires some knowledge of the data structures
used to store the mesh, solution fields, and simulation settings. While many
commercial :term:`CFD<CFD>` codes have developed user interfaces that allow most user
code interactions to occur through a :term:`GUI<GUI>` or even a text-based format, nekRS
very much remains a research tool. As such, even "routine" actions such as setting
boundary and initial conditions requires an understanding of the source code structure in
nekRS. This requirement is advantageous from a flexibility perspective, however, because
almost any user action that can be written in C++ ``.udf`` or :term:`OKL<OKL>` in ``.oudf``
files can be incorporated into a nekRS simulation.

This page contains a summary of some of the most commonly-used variables and structures
used to interact with nekRS. For array-type variables, the size of the array is also listed
in terms of the length of each dimension of that array. For instance, if the size of an array
is ``Nelements * Np``, then the data is stored first by each element, and second by each
quadrature point. If the variable is not an array type, the size is shown as ``1``.

Some variables have an equivalent form that is stored on the device that can be accessed
in device kernels. All such device variables and
arrays that live on the device by convention are prefixed with ``o_``. That is, ``mesh->x``
represents all the :math:`x`-coordinates of the quadrature points, and is stored on the host.
The same data, but accessible on the device, is ``mesh->o_x``. Not all variables and arrays
are automatically available on both the host and device, but those that are available are
indicated with a :math:`\checkmark` in the "Device-Side Version?" table column.


Mesh
----

This section describes commonly-used variables related to the mesh, which are all stored
on data structures of type ``mesh_t``. nekRS uses an archaic approach for conjugate heat
transfer applications, i.e. problems with separate fluid and solid domains. For problems
without conjugate heat transfer, all mesh information is stored on the ``nrs->mesh`` object,
while for problems with conjugate heat transfer, all mesh information is stored on the
``nrs->cds->mesh`` object. More information is available in the
:ref:`Creating a Mesh for Conjugate Heat Transfer <cht_mesh>` section. To keep the following
summary table general, the variable names are referred to simply as living on the ``mesh``
object, without any differentiation between whether that ``mesh`` object is the object on
``nrs`` or ``nrs->cds``.

Some notable points of interest that require additional comment:

* The :term:`MPI<MPI>` communicator is stored on the mesh, since domain decomposition
  is used to divide the mesh among processes. *Most* information stored on the ``mesh`` object
  strictly refers to the portion of the mesh owned by the current process. For instance,
  ``mesh->Nelements`` only refers to the number of elements "owned" by the current process
  (``mesh->rank``), not the total number of elements in the simulation mesh. Any exceptions
  to this process-local information is noted as applicable.

================== ============================ ================== =================================================
Variable Name      Size                         Device?            Meaning
================== ============================ ================== =================================================
``comm``           1                                               MPI communicator
``device``         1                                               backend device
``dim``            1                                               spatial dimension of mesh
``elementInfo``    ``Nelements``                                   phase of element (0 = fluid, 1 = solid)
``EToB``           ``Nelements * Nfaces``       :math:`\checkmark` boundary ID for each face
``N``              1                                               polynomial order for each dimension
``NboundaryFaces`` 1                                               *total* number of faces on a boundary (rank sum)
``Nelements``      1                                               number of elements
``Nfaces``         1                                               number of faces per element
``Nfp``            1                                               number of quadrature points per face
``Np``             1                                               number of quadrature points per element
``rank``           1                                               parallel process rank
``size``           1                                               size of MPI communicator
``vmapM``          ``Nelements * Nfaces * Nfp`` :math:`\checkmark` quadrature point index for faces on boundaries
``x``              ``Nelements * Np``           :math:`\checkmark` :math:`x`-coordinates of quadrature points
``y``              ``Nelements * Np``           :math:`\checkmark` :math:`y`-coordinates of quadrature points
``z``              ``Nelements * Np``           :math:`\checkmark` :math:`z`-coordinates of quadrature points
================== ============================ ================== =================================================
