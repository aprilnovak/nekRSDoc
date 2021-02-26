Plugins
=======

nekRS contains several "plugins" that provide both physics models and postprocessing
capabilities. nekRS's :term:`RANS` and low-Mach models, for instance, are provided as
plugins. While significant attention is not provided to most of the inner source code structure of nekRS,
these plugins require more in-depth explanation because their usage requires non-trivial
modifications to the ``.udf`` files. Before reading this page, first consult
:ref:`User-Defined Host Functions (.udf) <udf_functions>` so that you have the necessary
background on each of the ``.udf`` functions that will be discussed.

.. _rans_plugin:

RANS :math:`k`-:math:`\tau` Plugin
----------------------------------

The :term:`RANS` :math:`k`-:math:`\tau` plugin is available in the ``src/plugins/RANSktau.hpp``
header file. In order to add the :math:`k`-:math:`\tau` model to your case, you need
to include this file in your ``.udf`` file and manually add all the requisite parts of
the :math:`k`-:math:`\tau` methodology. Unless otherwise noted, all code snippets in
this section are placed in the ``.udf`` file.

First, add the necessary include file at the top
of your ``.udf`` file:

.. code-block:: cpp

  #include "udf.hpp"
  #include "plugins/RANSktau.hpp"

This is required in order to access the methods in the :term:`RANS` plugin. The
following sections then each describe a step in the :term:`RANS` model setup using the plugin.

.. _kernels:

Add the Physics Kernels
"""""""""""""""""""""""
The calculations performed to add contributions to the residuals occur within
:term:`OCCA` kernels. In order to add the :term:`RANS` equations, the corresponding
physics kernels must first be included. The :term:`RANS` kernels are added to by
calling the ``RANSktau::buildKernel`` function within ``UDF_LoadKernels``:

.. code-block:: cpp

  void UDF_LoadKernels(nrs_t * nrs)
  {
    RANSktau::buildKernel(nrs);
  }

The ``RANSKtau::buildKernel`` function performs two main actions -

  1. Propagate the constants in the :math:`k`-:math:`\tau` model in :ref:`RANS Coefficients <rans_coeffs>`
     to become available in :term:`OCCA` kernels with the approach as described in
     :ref:`Defining Variables to Access in Device Kernels <defining_variables_for_device>`.
  2. Add the :term:`OCCA` kernels that will perform the calculations needed to apply
     the :math:`k`-:math:`\tau` model.

Add the Closure Properties Calculation
""""""""""""""""""""""""""""""""""""""
Next, add the function that will update the properties used in the governing equations.
An example is shown in :ref:`Setting Custom Properties <custom_properties>` for setting
custom user-defined properties for a laminar flow scenario. The necessary steps to add
the material properties for the :term:`RANS` model is much simpler, however, but consists of the
same essential steps:

  1. Set the ``udf.properties`` function pointer to a function
     local to the ``.udf`` file that actually computes the properties
  2. Add that property function to the ``.udf``

For the first step, assign the ``udf.properties`` function pointer to a function in the
``.udf`` with signature ``void (nrs_t* nrs, dfloat time, occa::memory o_U, occa::memory o_S,
occa::memory o_UProp, occa::memory o_SProp)``. Based on the example shown in
:ref:`Setting Custom Properties <custom_properties>`, for illustration purposes we will
name this function ``material_properties``:

.. code-block:: cpp

  void UDF_Setup(nrs_t * nrs)
  {
    // other stuff unrelated to properties

    udf.properties = &material_properties;
  }

Then, for the second step, we need to add the following ``material_properties`` function
in the ``.udf`` file:

.. code-block:: cpp

  void material_props(nrs_t* nrs, dfloat time, occa::memory o_U, occa::memory o_S,
  occa::memory o_UProp, occa::memory o_SProp)
  {
    RANSktau::updateProperties();
  }

.. warning::

  nekRS's :math:`k`-:math:`\tau` implementation currently requires that
  the laminar dynamic viscosity and the density are constant. Therefore, you
  should not have any other material properties being set in this function
  like there were in :ref:`Setting Custom Properties <custom_properties>`.

The ``RANSktau::updateProperties`` function performs two main actions:

  1. Apply a limiter to :math:`k` and :math:`\tau` as described in
     :ref:`Closure Coefficients and Other Details <rans_details>`.
  2. Compute the turbulent viscosity as :math:`\mu_T\equiv\rho k\tau`
     and then set the diffusion coefficients in the momentum, :math:`k`,
     and :math:`\tau` equations to be :math:`\mu+\mu_T`,
     :math:`\mu+\mu_T/\sigma_k`, and :math:`\mu+\mu_T/\sigma_\tau`, respectively.

Add the Source Terms Calculation
""""""""""""""""""""""""""""""""
The same passive scalar infrastructure that is used to solve the energy conservation
equation is used to solve the :math:`k` and :math:`\tau` passive scalar equations.
However, these equations clearly have different forms - therefore, we need to explicitly
add these unique source terms to the :math:`k` and :math:`\tau` equations. While we
loaded the :term:`RANS` kernels in :ref:`Add Physics Kernels <kernels>`, we still
need to add those kernels to the governing equations. An example was provided in
:ref:`Setting Custom Source Terms <custom_sources>`, but the necessary steps to
add the :term:`RANS` source terms is much simpler, but consists of the
same essential steps:

  1. Set the ``udf.sEqnSource`` function pointer to a function
     local to the ``.udf`` file that actually computes the source terms
  2. Add that source term function to the ``.udf``

For the first step, assign the ``udf.sEqnSource`` function pointer to a function in the
``.udf`` with signature ``void (nrs_t *nrs, dfloat time, occa::memory o_S, occa::memory o_FS)``.
Based on the example shown in
:ref:`Setting Custom Source Terms <custom_sources>`, for illustration purposes we will
name this function ``user_q``:

.. code-block:: cpp

  void UDF_Setup(nrs_t * nrs)
  {
    // other stuff unrelated to the source terms

    udf.sEqnSource = &user_q;
  }

Then, for the second step, we need to add the following ``material_properties`` function
in the ``.udf`` file:

.. code-block:: cpp

  void user_q(nrs_t *nrs, dfloat time, occa::memory o_S, occa::memory o_FS)
  {
    RANSktau::updateSourceTerms();
  }

Initialize the RANS Solve
"""""""""""""""""""""""""

Finally, the last step to initialize the :term:`RANS` solve is to call the
``RANSktau::setup`` function. This function has signature
``void setup(nrs_t * nrs, dfloat mu, dfloat rho, int ifld)`` - ``nrs`` is the
flow simulation object, ``mu`` is the *constant* laminar viscosity, ``rho`` is
the *constant* density, and ``ifld`` is the integer corresponding to the
:math:`k` scalar. As mentioned previously, nekRS's :math:`k`-:math:`\tau` model
is currently restricted to constant laminar dynamic viscosity and constant density,
and the values passed into this ``setup`` function define those properties.

.. warning::

  For consistency, be sure that the viscosity and density passed in to
  ``RANSktau::setup`` are the same as the properties used in the mean flow equations.

Finally, ``ifld`` simply indicates where in the sequence of passive scalars the
:math:`k` scalar is positioned. For instance, if your problem has a temperature
passive scalar (scalar 0 by definition) and a chemical concentration passive
scalar (which you have indicated as ``SCALAR01`` in the ``.par`` file),
then the :math:`k` scalar should be positioned as the second scalar, and ``ifld = 2``.

.. warning::

  It is assumed that in the passive scalar list that ``ifld`` corresponds to the
  :math:`k` passive scalar and ``ifld + 1`` corresponds to the :math:`\tau` passive
  scalar. Be sure to order the scalars in the input file to respect this assumption.


Low-Mach Plugin
---------------

Turbulence Statistics Plugin
----------------------------

Velocity Recycling Plugin
-------------------------

