Tutorial
===========

.. toctree::2
   :maxdepth: 2
   :caption: Contents:

.. _compile_section:

Compile
---------------

For most systems, CaLES can be compiled from the root directory with the following commands ``make libs && make``, which will compile the 2DECOMP&FFT/cuDecomp libraries and CaLES. ``make clean`` clears the CaLES build files, ``make libsclean`` clears the 2DECOMP/cuDecomp builds, and ``make allclean`` clears both.

The ``Makefile`` in root directory is used to compiled the code, and is expected to work out-of-the-box for most systems. The ``build.conf`` file in the root directory can be used to choose the Fortran compiler (MPI wrapper), a few pre-defined profiles depending on the nature of the run (e.g., production vs debugging), and pre-processing options:

.. code-block:: text

    # compiler and compiling profile
    #
    FCOMP=GNU           # options: GNU, NVIDIA, INTEL
    FFLAGS_OPT=1        # for production runs
    FFLAGS_OPT_MAX=0    # for production runs (more aggressive optimization)
    FFLAGS_DEBUG=0      # for debugging
    FFLAGS_DEBUG_MAX=0  # for thorough debugging
    #
    # defines
    #
    DEBUG=1                    # best = 1 (no significant performance penalty)
    TIMING=1                   # best = 1
    IMPDIFF=0                  #
    IMPDIFF_1D=0               #
    PENCIL_AXIS=1              # = 1/2/3 for X/Y/Z-aligned pencils
    SINGLE_PRECISION=0         # perform the whole calculation in single precision
    #
    # GPU-related
    #
    GPU=0
    USE_NVTX=0

In this file, ``FCOMP`` can be one of **GNU** (**gfortran**), **INTEL** (**ifort**), **NVIDIA** (**nvfortran**), or **CRAY** (**ftn**); the predefined profiles for compiler options can be selected by choosing one of the ``FFLAGS_*`` option; finer control of the compiler flags may be achieved by building with, e.g., ``make FFLAGS+=[OTHER_FLAGS]``, or by tweaking the profiles directly under ``configs/flags.mk``. Similarly, the library paths (e.g., for FFTW) may need to be adapted in the ``Makefile`` (``LIBS`` variable) or by building with ``make LIBS+='-L[PATH_TO_LIB] -l[NAME_OF_LIB]'``. Finally, the following pre-processing options are available:

- **DEBUG**                    : performs some basic checks for debugging purposes
- **TIMING**                   : wall-clock time per time step is computed
- **IMPDIFF**                  : diffusion term of the N-S equations is integrated in time with an implicit discretization (thereby improving the stability of the numerical algorithm for viscous-dominated flows)
- **IMPDIFF_1D**               : same as above, but with implicit diffusion **only** along Z; for optimal parallel performance this option should be combined with ``PENCIL_AXIS=3``
- **PENCIL_AXIS**              : sets the default pencil direction, one of [1,2,3] for [X,Y,Z]-aligned pencils; X-aligned is the default and should be optimal for all cases except for Z implicit diffusion, where using Z-pencils is recommended
- **SINGLE_PRECISION**         : calculation will be carried out in single precision (the default precision is double)
- **GPU**                      : enable GPU accelerated runs (requires the ``FCOMP=NVIDIA``)
- **USE_NVTX**                 : enable `NVTX <https://https://docs.nvidia.com/nsight-visual-studio-edition/nvtx>`_ markers to tag certain code regions and assist with profiling

Typing ``make libs`` will build the 2DECOMP&FFT/cuDecomp libraries; then typing ``make`` will compile the code and create the executable ``cales`` in ``build/``.

Note that cuDecomp needs to be dynamically linked before performing a GPU run. To do this, one should update the ``LD_LIBRARY_PATH`` environment variable as follows (from the root directory):

.. code-block:: text

    export LD_LIBRARY_PATH=$PWD/dependencies/cuDecomp/build/lib:$LD_LIBRARY_PATH

Finally, the choice of compiler ``FCOMP`` (see ``configs/flags.mk``), and profile flags ``FFLAGS_*`` (see ``configs/flags.mk``) can easily be overloaded, for instance, as: ``make FC=ftn FFLAGS=-O2``. Linking and Include options can be changed in ``configs/libs.mk``.


.. _input_section:

Input
-------------

Setting `input.nml`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Consider the following input file  as an example of wall-modeled LES (WMLES) of a turbulent plane channel flow. The ``&dns`` namelist contains the necessary physical and computational parameters to set a case. The ``&les`` namelist includes parameters for the large-eddy simulation model. The ``&cudecomp`` namelist is optional and sets some runtime configurations for the cuDecomp library.

.. code-block:: text

    &dns
    ng(1:3) = 128, 96, 64
    l(1:3) = 12.8, 4.8, 2.
    gtype = 6, gr = 0.
    cfl = 0.95, dtmin = 1.e5
    visci = 125000.
    inivel = 'poi'
    is_wallturb = T
    nstep = 100000, time_max = 100., tw_max = 0.1
    stop_type(1:3) = T, F, F
    restart = F, is_overwrite_save = T, nsaves_max = 0
    icheck = 10, iout0d = 10, iout1d = 100, iout2d = 500, iout3d = 10000, isave = 5000
    cbcvel(0:1,1:3,1) = 'P','P',  'P','P',  'D','D'
    cbcvel(0:1,1:3,2) = 'P','P',  'P','P',  'D','D'
    cbcvel(0:1,1:3,3) = 'P','P',  'P','P',  'D','D'
    cbcpre(0:1,1:3)   = 'P','P',  'P','P',  'N','N'
    cbcsgs(0:1,1:3)   = 'P','P',  'P','P',  'D','D'
    bcvel(0:1,1:3,1) =  0.,0.,   0.,0.,   0.,0.
    bcvel(0:1,1:3,2) =  0.,0.,   0.,0.,   0.,0.
    bcvel(0:1,1:3,3) =  0.,0.,   0.,0.,   0.,0.
    bcpre(0:1,1:3  ) =  0.,0.,   0.,0.,   0.,0.
    bcsgs(0:1,1:3)   =  0.,0.,   0.,0.,   0.,0.
    bforce(1:3) = 0., 0., 0.
    is_forced(1:3) = T, F, F
    velf(1:3) = 1., 0., 0.
    dims(1:2) = 2, 2
    
    &les
    sgstype = 'smag'
    lwm(0:1,1:3) = 0,0, 0,0, 1,1
    hwm = 0.1
    
    &cudecomp
    cudecomp_t_comm_backend = 0, cudecomp_is_t_enable_nccl = T, cudecomp_is_t_enable_nvshmem = T
    cudecomp_h_comm_backend = 0, cudecomp_is_h_enable_nccl = T, cudecomp_is_h_enable_nvshmem = T

Here is a tip for vim/nvim users. Consider adding the following lines in your ``.vimrc`` file for syntax highlighting of the namelist file:

.. code-block:: text

    # vim
    if has("autocmd")
        au BufNewFile,BufRead *.nml set filetype=fortran
        au BufNewFile,BufRead *.namelist set filetype=fortran
    endif

Namelist `&dns`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: text
    
    ng(1:3) = 128, 96, 64
    l(1:3) = 12.8, 4.8, 2.
    gtype = 6, gr = 0.

These lines set the computational grid.

``ng(1:3)`` and ``l(1:3)`` are the **number of points**  and **domain length** in each direction.

``gtype`` and ``gr`` are the **grid stretching type** and **grid stretching parameter** that tweak the non-uniform grid in the third direction; zero ``gr`` implies no stretching. See ``initgrid.f90`` for more details. The following options are available for ``gtype``:

- 1: grid clustered towards both ends (default)
- 2: grid clustered towards the lower end
- 3: grid clustered towards the upper end
- 4: grid clustered towards the middle
- 5: grid clustered towards both ends naturally
- 6: grid clustered towards both ends suitable for WMLES of wall-bounded flows

.. code-block:: text

    cfl = 0.95, dtmin = 1.e5

This line controls the simulation time step.

The time step is set to be equal to ``min(cfl*dtmax,dtmin)``, i.e. the minimum value between ``dtmin`` and ``cfl`` times the maximum allowable time step ``dtmax`` (computed every ``ickeck`` time steps; see below).
``dtmin`` is therefore used when a constant time step, smaller than ``cfl*dtmax``, is required. If not, it should be set to a high value so that the time step is dynamically adjusted to ``cfl*dtmax``.

.. code-block:: text

    visci = 125000.

This line defines the inverse of the fluid viscosity, ``visci``, meaning that the viscosity is ``visc = visci**(-1)``. For a setup defined with two unit reference lengths and velocity scales, ``visci`` is half of the flow Reynolds number, as in the given example. For a setup defined with unit reference length and velocity scales, ``visci`` has the same value as the flow Reynolds number.

.. code-block:: text

    inivel = 'poi'
    is_wallturb = T

These lines set the initial velocity field.

``inivel`` **chooses the initial velocity field**. The following options are available:

- `zer`: zero velocity field
- `uni`: uniform velocity field equal to `uref`                                     ; streamwise direction in `x`
- `cou`: plane Couette flow profile with symmetric wall velocities equal to `uref/2`; streamwise direction in `x`
- `poi`: plane Poiseuille flow profile with mean velocity `uref`                    ; streamwise direction in `x`
- `tbl`: temporal boundary layer profile with wall velocity `uref`                  ; streamwise direction in `x`
- `pdc`: plane Poiseuille flow profile with constant pressure gradient              ; streamwise direction in `x`
- `log`: logarithmic profile with mean velocity `uref`                              ; streamwise direction in `x`
- `hcp`: half channel with plane Poiseuille profile and mean velocity `uref`        ; streamwise direction in `x`
- `hcl`: half channel with logarithmic profile and mean velocity `uref`             ; streamwise direction in `x`
- `hdc`: half plane Poiseuille flow profile with constant pressure gradient         ; streamwise direction in `x`
- `tgv`: three-dimensional Taylor-Green vortex
- `tgw`: two-dimensional   Taylor-Green vortex

``is_wallturb``, if true, **superimposes a high amplitude disturbance on the initial velocity field** that effectively triggers transition to turbulence in a wall-bounded shear flow.

See ``initflow.f90`` for more details.

.. code-block:: text

    nstep = 100000, time_max = 100., tw_max = 0.1
    stop_type(1:3) = T, F, F
    restart = F, is_overwrite_save = T, nsaves_max = 0

These lines set the simulation termination criteria and whether the simulation should be restarted from a checkpoint file.

``nstep`` is the **total number of time steps**.

``time_max`` is the **maximum physical time**.

``tw_max`` is the **maximum total simulation wall-clock time**.

``stop_type`` sets which criteria for terminating the simulation are to be used (more than one can be selected, and at least one of them must be `T`)

- ``stop_type(1)``, if true (`T`), the simulation will terminate after ``nstep`` time steps have been simulated
- ``stop_type(2)``, if true (`T`), the simulation will terminate after ``time_max`` physical time units have been reached
- ``stop_type(3)``, if true (`T`), the simulation will terminate after ``tw_max`` simulation wall-clock time (in hours) has been reached

a checkpoint file ``fld.bin`` will be saved before the simulation is terminated.

``restart``, if true, **restarts the simulation** from a previously saved checkpoint file, named ``fld.bin``.

``is_overwrite_save``, if true, overwrites the checkpoint file ``fld.bin`` at every save; if false, a symbolic link is created which makes ``fld.bin`` point to the last checkpoint file with name ``fld_???????.bin`` (with ``???????`` denoting the corresponding time step number). In the latter case, to restart a run from a different checkpoint one just has to point the file ``fld.bin`` to the right file, e.g.: ` ln -sf fld_0000100.bin fld.bin`.

``nsaves_max`` limits the number of saved checkpoint files, if ``is_over_write_save`` is false; a value of `0` or any negative integer corresponds to no limit, and the code uses the file format described above; otherwise, only ``nsaves_max`` checkpoint files are saved, with the oldest save being overwritten when the number of saved checkpoints exceeds this threshold; in this case, files with a format ``fld_????.bin`` are saved (with ``????`` denoting the saved file number), with ``fld.bin`` pointing to the last checkpoint file as described above; moreover, a file ``log_checkpoints.out`` records information about the time step number and physical time corresponding to each saved file number.

.. code-block:: text

    icheck = 10, iout0d = 10, iout1d = 100, iout2d = 500, iout3d = 10000, isave = 5000

These lines set the frequency of time step checking and output:

- every ``icheck`` time steps **the new time step size** is computed according to the new stability criterion and cfl (above)
- every ``iout0d`` time steps **history files with global scalar variables** are appended; currently the forcing pressure gradient and time step history are reported
- every ``iout1d`` time steps **1d profiles** are written (e.g. velocity and its moments) to a file
- every ``iout2d`` time steps **2d slices of a 3d scalar field** are written to a file
- every ``iout3d`` time steps **3d scalar fields** are written to a file
- every ``isave``  time steps a **checkpoint file** is written (``fld_???????.bin``), and a symbolic link for the restart file, ``fld.bin``, will point to this last save so that, by default, the last saved checkpoint file is used to restart the simulation

1d, 2d and 3d outputs can be tweaked modifying files ``out?d.h90``, and re-compiling the source. See also ``output.f90`` for more details.

.. code-block:: text

    cbcvel(0:1,1:3,1) = 'P','P',  'P','P',  'D','D'
    cbcvel(0:1,1:3,2) = 'P','P',  'P','P',  'D','D'
    cbcvel(0:1,1:3,3) = 'P','P',  'P','P',  'D','D'
    cbcpre(0:1,1:3)   = 'P','P',  'P','P',  'N','N'
    cbcsgs(0:1,1:3)   = 'P','P',  'P','P',  'D','D'
    bcvel(0:1,1:3,1) =  0.,0.,   0.,0.,   0.,0.
    bcvel(0:1,1:3,2) =  0.,0.,   0.,0.,   0.,0.
    bcvel(0:1,1:3,3) =  0.,0.,   0.,0.,   0.,0.
    bcpre(0:1,1:3  ) =  0.,0.,   0.,0.,   0.,0.
    bcsgs(0:1,1:3)   =  0.,0.,   0.,0.,   0.,0.

These lines set the boundary conditions (BC).

The **type** (BC) for each field variable are set by a row of six characters, `X0 X1  Y0 Y1  Z0 Z1` where,

- `X0` `X1` set the type of BC the field variable for the **lower** and **upper** boundaries in `x`
- `Y0` `Y1` set the type of BC the field variable for the **lower** and **upper** boundaries in `y`
- `Z0` `Z1` set the type of BC the field variable for the **lower** and **upper** boundaries in `z`

The first five rows correspond to the three velocity components, pressure and eddy viscosoty, i.e. `u`, `v`, `w`, `p` and `visct`. The following options are available:

- `P` periodic
- `D` Dirichlet
- `N` Neumann

The next five rows follow the same logic but specify the boundary condition **values** (dummy for a periodic direction). When a wall model is applied to a wall, the boundary condition for that wall should be set to a no-slip condition as given in the example.

.. code-block:: text

    bforce(1:3) = 0., 0., 0.
    is_forced(1:3) = T, F, F
    velf(1:3) = 1., 0., 0.

These lines set the flow forcing.

``bforce``, is a constant **body force density term** in the direction in question (e.g., the negative of a constant pressure gradient) that can be added to the right-hand-side of the momentum equation. The three values correspond to three domain directions.

``is_forced``, if true in the direction in question, **forces the flow** with a pressure gradient that balances the total wall shear (e.g., for a pressure-driven channel). The three boolean values correspond to three domain directions.

``velf``, is the **target bulk velocity** in the direction in question (where ``is_forced`` is true). The three values correspond to three domain directions.

.. code-block:: text

    dims(1:2) = 2, 2

This line set the grid of computational subdomains.

``dims`` is the **processor grid**, the number of domain partitions along the first and second decomposed directions (which depend on the selected default pencil orientation). ``dims(1)*dims(2)`` corresponds therefore to the total number of computational subdomains. Setting ``dims(:) = [0,0]`` will trigger a runtime autotuning step to find the processor grid that minimizes transpose times. Note, however, that other components of the algorithm (e.g., collective I/O) may also be affected by the choice of processor grid.


Namelist `&les`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: text

    sgstype = 'smag'
    lwm(0:1,1:3) = 0,0, 0,0, 1,1
    hwm = 0.1

These lines set the subgrid-scale (SGS) model and wall model for the large-eddy simulation.

``sgstype`` is the type of **subgrid-scale model**. The following options are available:

- smag: classical Smagorinsky model with the van Driest damping function
- dsmag: dynamic Smagorinsky model with Lilly's modification

``lwm(0:1,1:3)`` sets the **types of wall model** for the LES. The first row corresponds to the lower wall, and the second row to the upper wall. The three columns correspond to the three domain directions. Different wall models can be applied to each wall and direction. The following options are available:

- 0: no wall model
- 1: logarithmic law wall model

``hwm`` is the **wall-modeled layer thickness**.

Namelist `&cudecomp`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This is an **optional** namelist to set some runtime configurations for the *cuDecomp* library. Consider the following ``&cudecomp`` namelist, which corresponds to the default options in case the file is not provided:

.. code-block:: text

    &cudecomp
    cudecomp_t_comm_backend = 0, cudecomp_is_t_enable_nccl = T, cudecomp_is_t_enable_nvshmem = T
    cudecomp_h_comm_backend = 0, cudecomp_is_h_enable_nccl = T, cudecomp_is_h_enable_nvshmem = T

The first line sets the configuration for the transpose communication backend autotuning. Here ``cudecomp_t_comm_backend`` can be one of:

- 1: CUDECOMP_TRANSPOSE_COMM_MPI_P2P
- 2: CUDECOMP_TRANSPOSE_COMM_MPI_P2P_PL
- 3: CUDECOMP_TRANSPOSE_COMM_MPI_A2A
- 4: CUDECOMP_TRANSPOSE_COMM_NCCL
- 5: CUDECOMP_TRANSPOSE_COMM_NCCL_PL
- 6: CUDECOMP_TRANSPOSE_COMM_NVSHMEM
- 7: CUDECOMP_TRANSPOSE_COMM_NVSHMEM_PL
- any other value: enable runtime transpose backend autotuning

The other two boolean values, enable/disable the NCCL (``cudecomp_is_t_enable_nccl``) and NVSHMEM (``cudecomp_is_t_enable_nvshmem``) options for **transpose** communication backend autotuning.

The second line is analogous to the first one, but for halo communication backend autotuning. Here ``cudecomp_h_comm_backend`` can be one of:

- 1: CUDECOMP_HALO_COMM_MPI
- 2: CUDECOMP_HALO_COMM_MPI_BLOCKING
- 3: CUDECOMP_HALO_COMM_NCCL
- 4: CUDECOMP_HALO_COMM_NVSHMEM
- 5: CUDECOMP_HALO_COMM_NVSHMEM_BLOCKING
- any other value: enable runtime halo backend autotuning

The other two boolean values, enable/disable the NCCL (``cudecomp_is_h_enable_nccl``) and NVSHMEM (``cudecomp_is_h_enable_nvshmem``) options for **halo** communication backend autotuning.

Finally, it is worth recalling that passing ``dims(1:2) = [0,0]`` under ``&dns`` will trigger the processor grid autotuning, so there is no need to provide that option in the ``&cudecomp`` namelist.


.. _visualization_section:

Visualization
--------------------

In addition to the binary files for visualization, CaLES now generates a log file that contains information about the saved data (see ``out2d.h90`` and ``out3d.h90`` for more details); this new approach uses that log file to generate the ``Xdmf`` visualization file.

The steps are as follows:

1. After the simulation has run, copy the contents of ``utils/visualize_fields/gen_xdmf_easy/write_xdmf.py`` to the simulation ``data`` folder.
2. Run the file with ``python write_xdmf.py``.
3. Load the generated Xdmf (``*.xmf``) file using paraview/visit or other visualization software.


3D fields
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When running the script ``write_xdmf.py`` we get the following prompts:

.. code-block:: bash

    # python write_xdmf.py
    Name of the log file written by CaLES [log_visu_3d.out]:
    Name to be appended to the grid files to prevent overwriting []:
    Name of the output file [viewfld_DNS.xmf]:

- The first value is the name of the file that logged the saved data.
- The second is a name to append to the grid files that are generated, which should change for different log files to prevent conflicts.
- The third is the name of the visualization file.

by pressing <kbd>enter</kbd> three times, the default values in the square brackets are assumed by the script; these correspond to the default steps required for visualizing 3D field data.

2D fields
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The procedure for visualizing 2D field data that is saved by CaLES in ``out2d.h90`` is exactly the same; it is just that the correct log file should be selected. CaLES saves by default field data in a plane of constant `y=ly/2`, and logs the saves to a file named ``log_visu_2d_slice_1.out``. If more planes are saved, the user should make sure that one log file per plane is saved by CaLES (e.g. if another plane is saved, the log file written in ``out2d.h90`` could be named ``log_visu_2d_slice_2.out``); see ``out2d.h90`` for more details. The corresponding steps to generate the Xdmf file would be, for instance:

.. code-block:: bash

    # python write_xdmf.py
    Name of the log file written by CaLES [log_visu_3d.out]: log_visu_2d_slice_1.out
    Name to be appended to the grid files to prevent overwriting []: 2d
    Name of the output file [viewfld_DNS.xmf]: viewfld_DNS_2d.xmf

Checkpoint files
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A similar script also located in ``utils/visualize_fields/gen_xdmf_easy``, named ``write_xdmf_restart.py``, can be used to generate metadata that allows to visualize the field data contained in all saved checkpoint files:

.. code-block:: bash

    # python write_xdmf_restart.py
    Name of the pattern of the restart files to be visualized [fld?*.bin]:
    Names of stored variables [VEX VEY VEZ PRE]:
    Name to be appended to the grid files to prevent overwriting [_fld]:
    Name of the output file [viewfld_DNS_fld.xmf]:


 
 
 
 

