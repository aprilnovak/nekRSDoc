.. _theory:

Theory
======

This page introduces the governing equations and the numerical discretization
at a high level. nekRS includes models for incompressible flow, a partially
compressible low-Mach formulation, the Stokes equations, and the :math:`k`-:math:`\tau`
:term:`RANS` equations.

.. _ins_model:

Incompressible Flow Model
-------------------------

The governing equations for conservation of mass, momentum, and energy for
an incompressible fluid are

.. math::

  \nabla\cdot\mathbf u=0

.. math::

  \rho\left(\frac{\partial\mathbf u}{\partial t}+\mathbf u\cdot\nabla\mathbf u\right)=-\nabla P=\nabla\cdot\tau+\rho\ \mathbf f

.. math::

  \rho C_p\left(\frac{\partial T}{\partial t}+\mathbf u\cdot\nabla T\right)=\nabla\cdot\left(k\nabla T\right)+\dot{q}

In these equations, :math:`t` is time,
:math:`\mathbf u` is the velocity vector, :math:`\rho` is the density, :math:`P` is the pressure,
:math:`\tau` is the viscous stress tensor, :math:`\mathbf f` is a force vector, :math:`C_p` is the
isobaric specific heat capacity, :math:`T` is the temperature, :math:`k` is the thermal conductivity,
and :math:`\dot{q}` is a volumetric heat source. If the viscosity is constant, the viscous stress tensor
can be contracted to give

.. math::

  \nabla\cdot\tau=\nabla^2\mathbf u

This is referred to as the "no-stress" formulation. In the general case for non-constant viscosity,
the viscous stress tensor is given by the Navier-Stokes closure as

.. math::

  \nabla\cdot\tau=\nabla\cdot\left\lbrack\mu\left(\nabla\mathbf u+\nabla\mathbf u^T\right)\right\rbrack

.. _nondimensional_eqs:

Non-Dimensional Formulation
"""""""""""""""""""""""""""

It is often advantageous to solve these equations in non-dimensional form. Here, we introduce
the non-dimensional form in as general a manner as possible, assuming the use of variable
material properties for density, viscosity, specific heat capacity, and thermal conductivity
that are functions of temperature, :math:`\rho=\rho(T)`, :math:`\mu=\mu(T)`,
:math:`C_p=C_p(T)`, and :math:`k=k(T)`. For simplicity, the functional notation is
omitted throughout.

Introduce
the non-dimensional variables :math:`\mathbf x^\dagger=\frac{\mathbf x}{L}`,
:math:`\mathbf u^\dagger=\frac{\mathbf u}{U}`, :math:`t^\dagger=\frac{tU}{L}`,
and :math:`\mathbf f^\dagger=\frac{\mathbf f L}{U^2}`. For the material properties,
we non-dimensionalize based on some reference temperature :math:`T_0`, such that
:math:`\rho^\dagger=\frac{\rho}{\rho_0}`, :math:`\mu^\dagger=\frac{\mu}{\mu_0}`,
:math:`k^\dagger=\frac{k}{k_0}`, and :math:`C_p^\dagger=\frac{C_p}{C_{p,0}}`. Here,
a subscript of :math:`0` is shorthand notation that indicates that the property
is evaluated at :math:`T_0`, such that :math:`k_0\equiv k(T_0)`.
Finally, for convection-dominated flows,
the pressure is non-dimensionalized in terms of the dynamic pressure as
:math:`P^\dagger=\frac{P}{\rho_0 U^2}`.

Inserting these non-dimensional variables
into the conservation of mass, momentum, and energy equations gives

.. math::

  \frac{\partial u_i^\dagger}{\partial x_i^\dagger}=0

.. math::

  \rho^\dagger\left(\frac{\partial u_i^\dagger}{\partial t^\dagger}+u_j^\dagger\frac{\partial u_i^\dagger}{\partial x_j^\dagger}\right)=-\frac{\partial P^\dagger}{\partial x_i^\dagger}+\frac{1}{Re}\frac{\partial \tau_{ij}^\dagger}{\partial x_j^\dagger}+\rho^\dagger f_i^\dagger

In these equations, the :math:`\nabla` are expanded to explicitly show that all derivatives
are taken with respect to the nondimensional space variable :math:`\mathbf x^\dagger`. :math:`Re`
is the Reynolds number

.. math::

  Re\equiv\frac{\rho_0 UL}{\mu_0}

To non-dimensionalize the energy conservation equation, use the previous non-dimensional
variables in addition to a non-dimensional temperature, :math:`T^\dagger=\frac{T-T_0}{\Delta T}`,
where :math:`\Delta T` is a reference temperature rise relative to a baseline temperature
:math:`T_0`. The heat source is non-dimensionalized as :math:`\dot{q}^\dagger=\frac{\dot{q}}{\rho_0 C_{p,0} U\Delta T/L}`,
which arises naturally from the simple formulation of bulk energy conservation of
:math:`Q=\dot{m}C_p\Delta T`, where :math:`Q` is a heat source (units of Watts) and
:math:`\dot{m}` is a mass flowrate.

Inserting these non-dimensional variables into the energy conservation equation gives

.. math::

  \rho^\dagger C_p^\dagger\left(\frac{\partial T^\dagger}{\partial t^\dagger}+u_i^\dagger\frac{\partial T^\dagger}{\partial x_i^\dagger}\right)=\frac{1}{Pe}\frac{\partial}{\partial x_i^\dagger}\left(k^\dagger\frac{\partial T^\dagger}{\partial x_i^\dagger}\right)+\dot{q}^\dagger

where :math:`Pe` is the Peclet number,

.. math::

  Pe\equiv\frac{LU}{\alpha}

and :math:`\alpha` is the thermal diffusivity,

.. math::

  \alpha\equiv\frac{k_0}{\rho_0 C_{p,0}}

Low-Mach Partially-Compressible Model
-------------------------------------

Stokes Equations
----------------

RANS Models
-----------

The :term:`RANS` equations are derived from the conservation of mass, momentum, and energy
equations by expressing each field :math:`f` as the sum of a mean :math:`\overline{f}` and a fluctuation,
:math:`f'`,

.. math::

  f(\mathbf x, t)=\overline{f}(\mathbf x)+f'(\mathbf x,t)

Specifically, the mean is a time average of :math:`f`, of

.. math::

  \overline{f}=\lim_{t'\rightarrow\infty}\frac{1}{S}\int_{t}^{t+S}f(\mathbf x,t)dt

Inserting the above "Reynolds decomposition" for :math:`\mathbf u`, :math:`P`, and :math:`T`
into the governing equations then gives the :term:`RANS` equations. The discussion here focuses
on the incompressible flow equations in :ref:`Incompressible Flow Model <ins_model>`, for
which the :term:`RANS` mass and momentum equations are

.. math::

  \frac{\partial\overline{u_i}}{\partial x_i} = 0

.. math::

  \rho\left(\frac{\partial\overline{u_i}}{\partial t}+\overline{u_j}\frac{\partial\overline{u_i}}{\partial x_j}+\frac{\partial}{\partial x_j}\overline{u_i'u_j'}\right)=-\frac{\partial \overline{P}}{\partial x_i}+\frac{\partial}{\partial x_j}\left(2\mu \overline{S_{ij}}\right)+\rho\overline{\mathbf f}

where :math:`\overline{S_{ij}}` is the mean strain rate tensor,

.. math::

  \overline{S_{ij}}=\frac{1}{2}\left(\frac{\partial \overline{u_i}}{\partial x_j}+\frac{\partial\overline{u_j}}{\partial x_i}\right)

The mass and momentum conservation equations have the same form as the instantaneous flow
equations in :ref:`Incompressible Flow Model <ins_model>` except for the addition of another
stress tensor to the momentum equation - :math:`\rho \overline{u_i'u_j'}`;
this new stress term is referred to as the Reynolds stress tensor. The new term,
:math:`\rho\ \partial(\overline{u_i'u_j'})/\partial x_j` represents the time-averaged rate
of momentum transfer due to turbulence. The objective of :term:`RANS` models is to provide
a closure for the Reynolds stress tensor in terms of the mean flow such that the time-averaged
equations can be solved for the mean flow.

The :term:`RANS` models in nekRS are based on the Boussinesq eddy viscosity approximation,
which assumes that the momentum flux that induces the Reynolds stresses shares the same
functional form as the momentum flux that induces the molecular stresses. In other words,
the Navier-Stokes closure that was used to relate the deviatoric stress tensor
:math:`\tau_{ij}` to the strain rate tensor,

.. math::

  \tau_{ij}=\mu\left(\frac{\partial u_i}{\partial x_j}+\frac{\partial u_j}{\partial x_i}\right)-\underbrace{\frac{2}{3}\mu\frac{\partial u_i}{\partial x_i}\delta_{ij}}_\text{$=\ 0$ if incompressible}

is assumed applicable to the Reynolds stress tensor, but with instantaneous velocities replaced by
mean velocities and the molecular viscosity replaced by the turbulent eddy viscosity
:math:`\mu_T`,

.. math::

  \rho\overline{u_i'u_j'}=\mu_T\left(\frac{\partial\overline{u_i}}{\partial x_j}+\frac{\partial\overline{u_j}}{\partial x_i}\right)-\underbrace{\frac{2}{3}\mu\frac{\partial \overline{u_i}}{\partial x_i}\delta_{ij}}_\text{$=\ 0$ if incompressible}-\frac{2}{3}\rho k\delta_{ij}

Here, :math:`k` is the turbulent kinetic energy,

.. math::

  k\equiv\frac{1}{2}\left(\overline{u_1'u_1'}+\overline{u_2'u_2'}+\overline{u_3'u_3'}\right)

The final term in the Boussinesq approximation for the Reynolds stress tensor simply ensures that
the trace of the Reynolds stress tensor equals :math:`2k`, because otherwise, for incompressible flows,
the trace of the Reynolds stress tensor would be zero. Inserting the Boussinesq eddy viscosity
model for the Reynolds stress tensor into the incompressible flow mean momentum equation then gives

.. math::

  \rho\left(\frac{\partial\overline{u_i}}{\partial t}+\overline{u_j}\frac{\partial\overline{u_i}}{\partial x_j}\right)=-\frac{\partial \overline{P}}{\partial x_i}+\frac{\partial}{\partial x_j}\left\lbrack 2\left(\mu+\mu_T\right) \overline{S_{ij}}-\frac{2}{3}\rho k\delta_{ij}\right\rbrack+\rho\overline{\mathbf f}


.. note::

  Even if the molecular viscosity is constant, you must set ``stressFormulation = true`` in
  the input file because the total viscosity (molecular plus turbulent) will not be constant.


