Examples
============

The difference between DNS and LES input files lies in disabling the SGS model and the wall model. Specifically, in the following section:

.. code-block:: fortran

    &les
    sgstype = 'none' 
    lwm(0:1,1:3) = 0,0, 0,0, 0,0
    hwm = 0.

By setting ``sgstype`` to ``none`` and ``lwm`` to ``0``, DNS computation can be enabled.

All example input files can be found in the ``examples`` folder on GitHub: https://github.com/CaNS-World/CaLES.


.. toctree::
   :maxdepth: 2
   :caption: Contents:


   DNS
   LES