The nekRS Computational Fluid Dynamics Code
===================================================

nekRS is a computational fluid dynamics code developed at :term:`ANL`, :term:`UIUC`, and :term:`PSU`.
nekRS aims to leverage the present trend in :term:`GPU`-based :term:`HPC` systems to perform
:term:`CFD` on :term:`GPU`-accelerated systems. By using the :term:`OCCA` library's unified
:term:`API`, nekRS can run on :term:`CPUs<CPU>` and on :term:`GPU`-accelerated :term:`CPUs<CPU>` that
support :term:`CUDA`, :term:`HIP`, or :term:`OpenCL`.

This guide is intended to help new users get started with using nekRS, as well as serve as a
reference for more advanced users. Because the :term:`Nek5000` code is somewhat of a predecessor to
nekRS, some aspects of the current nekRS design are selected to enable faster translation of
:term:`Nek5000` input files into nekRS input files. Throughout this documentation, all such
:term:`Nek5000`-oriented settings will be referred to as "legacy" settings. Because these
:term:`Nek5000`-oriented settings require proficiency in Fortran, structured text formats,
and several additional input files, all new users are encouraged to adopt the nekRS-based problem setup.

We recommend working through this user guide in the order below. At the very least, please
read :ref:`The nekRS Input Files <input>` page before reading the :ref:`Detailed Code Usage <detailed>`
page, as some necessary concepts are introduced in this order.

Table of Contents
-----------------

.. toctree::
   :maxdepth: 2

   input_files
   tutorials
   detailed_usage
   just_in_time_compilation
   glossary

.. Indices and tables
.. ==================

.. * :ref:`genindex`
.. * :ref:`modindex`
.. * :ref:`search`
