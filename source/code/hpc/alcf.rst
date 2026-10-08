==============================
ALCF (Polaris / Aurora / Crux)
==============================

Access & Common Commands
========================
* **Join the project:** on `my.alcf.anl.gov <https://my.alcf.anl.gov/>`__ (request an ALCF account there first if needed), find ``electrolyte-chibueze``, click **Request Membership** and wait for PI approval.
* **Login (MobilePASS+):** enter your PIN in the app, not in the SSH window; the one-time passcode it shows is the SSH password.

.. code-block:: bash

    ssh <ALCF username>@polaris.alcf.anl.gov      # or aurora.alcf.anl.gov / crux.alcf.anl.gov
    qsub job.sh
    qstat -u $USER ; qstat -f <jobid>             # "comment" field says why a job is not running
    qstat -x -u $USER                             # finished jobs (kept two weeks)
    qdel <jobid>
    qstat -Qf <queue>                             # queue limits
    sbank-list-allocations -p electrolyte-chibueze -r all   # allocation (node-hours)
    myquota ; myprojectquotas                     # home and project usage

Systems & Storage
=================
All three use PBS with ``-A electrolyte-chibueze``. ALCF charges whole nodes (node-hours), so pack as many simulations as possible into each node.

=========  =====================================  ====================================================
System     Working directory                      Node / best for
=========  =====================================  ====================================================
Polaris    ``/eagle/electrolyte-chibueze/$USER``  4× A100; 4 small-box MDs per node
Aurora     ``/flare/electrolyte-chibueze/$USER``  6 PVC GPUs (12 tiles); 12 MDs per node
Crux       ``/eagle/electrolyte-chibueze/$USER``  CPU only, 2×64 cores; CPU MD, box building, analysis
=========  =====================================  ====================================================

.. warning::
    Eagle and Flare are **not backed up**. Keep runs in the project directories, never in home. Aurora cannot see Eagle; move data with Globus (``alcf#dtn_flare`` ↔ ``alcf#dtn_eagle``).

PBS Notes & Queues
==================
* ``-A``, ``walltime`` and ``filesystems`` are required: ``filesystems=home:eagle`` on Polaris/Crux, ``filesystems=home:flare`` on Aurora. Use ``-l place=scatter``; ``-k doe`` is added automatically.
* Start scripts with ``#!/bin/bash -l``. Never put comments on or between ``#PBS`` lines. Submit Aurora jobs from ``/flare/...``, not from home.
* ``qsub: Job rejected by all possible destinations``: the node count or walltime fits no queue.

.. warning::
    Set ``-maxh`` about 0.5 h below the walltime (24 h → 23.5; ``debug`` 1 h → 0.9) and change both together, otherwise PBS kills mdrun before it writes its final checkpoint.

.. list-table::
   :widths: 12 28 60
   :header-rows: 1

   * - System
     - Test queue
     - Single-node production
   * - Polaris
     - ``debug``: 1–2 nodes, ≤1 h
     - ``preemptable``: 1–10 nodes, ≤72 h, can be killed anytime, needs ``-r y``; ``capacity``: 1–4 nodes, ≤168 h, 1 running job per user. ``prod`` needs ≥10 nodes
   * - Aurora
     - ``debug``: 1–2 nodes, 1 h, 1 job per user
     - ``capacity``: 1–16 nodes, ≤168 h, ≤2 running jobs per user. ``prod`` needs ≥256 nodes
   * - Crux
     - ``debug``: 1–4 nodes, ≤2 h
     - ``workq-route`` (routes to ``workq``): 1–176 nodes, ≤24 h

Building GROMACS
================
ALCF documents no GROMACS module for Aurora/Crux, and the Polaris page lists an older 2022.1 build. Run ``module avail gromacs``; if nothing suitable exists, build into the project directory. Minimum versions: ≥2025.3 on Polaris (CUDA 13), ≥2026.4 for Aurora's default oneAPI 2026.1. Download `gromacs-2026.4.tar.gz <https://ftp.gromacs.org/gromacs/gromacs-2026.4.tar.gz>`__ on a login node.

.. note::
    These commands follow the official docs and have not been verified on all three machines. Build on a compute node (``qsub -I ... -q debug``); compute nodes reach the internet only through ``http://proxy.alcf.anl.gov:3128``. Polaris and Crux share ``/home`` and ``/eagle``, so keep their install prefixes separate.

.. code-block:: bash

    # Polaris (CUDA, A100 = sm_80)
    module restore; module use /soft/modulefiles
    module swap PrgEnv-nvidia PrgEnv-gnu
    module load cuda/13.0
    module load craype-x86-milan craype-accel-nvidia80
    module load spack-pe-base cmake
    export http_proxy=http://proxy.alcf.anl.gov:3128 https_proxy=http://proxy.alcf.anl.gov:3128
    cmake .. -DCMAKE_C_COMPILER=cc -DCMAKE_CXX_COMPILER=CC -DGMX_MPI=ON -DGMX_OPENMP=ON \
      -DGMX_GPU=CUDA -DCMAKE_CUDA_ARCHITECTURES=80 -DGMX_BUILD_OWN_FFTW=ON \
      -DCMAKE_INSTALL_PREFIX=/eagle/electrolyte-chibueze/$USER/sw/gromacs-2026.4-polaris
    make -j 16 && make install

    # Aurora (SYCL, Intel PVC); default PE already loads oneAPI + MPICH
    module load cmake
    cmake .. -DCMAKE_C_COMPILER=icx -DCMAKE_CXX_COMPILER=icpx -DGMX_MPI=ON \
      -DGMX_GPU=SYCL -DGMX_SYCL=DPCPP -DGMX_GPU_NB_NUM_CLUSTER_PER_CELL_X=1 -DGMX_GPU_NB_CLUSTER_SIZE=8 \
      -DGMX_FFT_LIBRARY=mkl -DCMAKE_INSTALL_PREFIX=/flare/electrolyte-chibueze/$USER/sw/gromacs-2026.4-aurora
    make -j 32 && make install

    # Crux (CPU only, AMD EPYC 7742 = Zen2)
    module swap PrgEnv-cray PrgEnv-gnu
    module use /soft/modulefiles; module load spack-pe-base cmake
    export http_proxy=http://proxy.alcf.anl.gov:3128 https_proxy=http://proxy.alcf.anl.gov:3128
    cmake .. -DCMAKE_C_COMPILER=cc -DCMAKE_CXX_COMPILER=CC -DGMX_MPI=ON -DGMX_GPU=OFF \
      -DGMX_SIMD=AVX2_128 -DGMX_BUILD_OWN_FFTW=ON \
      -DCMAKE_INSTALL_PREFIX=/eagle/electrolyte-chibueze/$USER/sw/gromacs-2026.4-crux
    make -j 32 && make install

Job Templates
=============
Each template runs ``mdrun -deffnm md`` in every ``sim*`` directory, so copy the stage tpr in as ``md.tpr`` (see :doc:`/code/md/running`).

Polaris: 4 Simulations per Node (One per A100)
----------------------------------------------

.. code-block:: bash

    #!/bin/bash -l
    #PBS -A electrolyte-chibueze
    #PBS -N gmx-4x1
    #PBS -q preemptable
    #PBS -l select=1:system=polaris
    #PBS -l place=scatter
    #PBS -l walltime=24:00:00
    #PBS -l filesystems=home:eagle
    #PBS -k doe
    #PBS -j oe
    #PBS -r y

    # Layout: sim0..sim3, each with md.tpr; -r y requeues after preemption, -cpi resumes
    cd ${PBS_O_WORKDIR}
    module restore; module use /soft/modulefiles      # same modules as the build
    module swap PrgEnv-nvidia PrgEnv-gnu
    module load cuda/13.0
    module load craype-x86-milan craype-accel-nvidia80
    GMX=/eagle/electrolyte-chibueze/${USER}/sw/gromacs-2026.4-polaris/bin/gmx_mpi

    GPUS=(0 1 2 3)
    CPUS=(24-31 16-23 8-15 0-7)    # cores closest to each GPU (official topology)
    for i in 0 1 2 3; do
      ( cd sim${i}
        export CUDA_VISIBLE_DEVICES=${GPUS[$i]}
        mpiexec -n 1 --ppn 1 --cpu-bind list:${CPUS[$i]} --env OMP_NUM_THREADS=8 \
          ${GMX} mdrun -deffnm md -ntomp 8 -pin inherit -nb gpu -pme gpu -update gpu \
          -cpi md.cpt -maxh 23.5 > mdrun.out 2>&1 ) &
    done
    wait

* Testing: ``-q debug``, ``walltime=1:00:00``, ``-maxh 0.9``. Long runs: ``-q capacity`` (≤168 h).
* Job modules must match the build modules, or mdrun fails at startup; the error only appears in each ``mdrun.out``.
* ``gmx_mpi mdrun -multidir sim0 sim1 sim2 sim3`` (with ALCF's ``set_affinity_gpu_polaris.sh``) also works, but one failing simulation kills the whole job. For several simulations per GPU, use MPS as in the ALCF docs (``nvidia-cuda-mps-control -d``).

Aurora: 12 Simulations per Node (One per GPU Tile)
--------------------------------------------------

.. code-block:: bash

    #!/bin/bash -l
    #PBS -A electrolyte-chibueze
    #PBS -N gmx-12tile
    #PBS -q capacity
    #PBS -l select=1
    #PBS -l place=scatter
    #PBS -l walltime=24:00:00
    #PBS -l filesystems=home:flare
    #PBS -k doe
    #PBS -j oe

    # qsub from /flare/electrolyte-chibueze/...; layout: sim00..sim11, each with md.tpr
    cd ${PBS_O_WORKDIR}
    GMX=/flare/electrolyte-chibueze/${USER}/sw/gromacs-2026.4-aurora/bin/gmx_mpi
    export ZE_ENABLE_PCI_ID_DEVICE_ORDER=1
    export SYCL_CACHE_PERSISTENT=1
    export MPIR_CVAR_ENABLE_GPU=0

    TILES=(0.0 0.1 1.0 1.1 2.0 2.1 3.0 3.1 4.0 4.1 5.0 5.1)
    CPUS=(1-8 9-16 17-24 25-32 33-40 41-48 53-60 61-68 69-76 77-84 85-92 93-100)   # skip reserved cores 0 and 52
    for i in $(seq 0 11); do
      d=$(printf "sim%02d" ${i})
      ( cd ${d}
        export ZE_AFFINITY_MASK=${TILES[$i]}
        mpiexec -n 1 --ppn 1 --cpu-bind list:${CPUS[$i]} --env OMP_NUM_THREADS=8 \
          ${GMX} mdrun -deffnm md -ntomp 8 -pin inherit -nb gpu -pme gpu -bonded cpu \
          -cpi md.cpt -maxh 23.5 > mdrun.out 2>&1 ) &
    done
    wait
    grep -L Performance sim*/md.log      # list simulations that did not finish

* ``No VNIs available in internal allocator`` in ``mdrun.out``: add ``--no-vni`` to ``mpiexec`` (see `Aurora known issues <https://docs.alcf.anl.gov/aurora/known-issues/>`__).
* The best GPU/CPU placement of ``-pme``/``-bonded``/``-update`` on PVC depends on the system; compare ``Performance: ns/day`` in ``debug`` for each new system.

Crux: 8 CPU Simulations per Node (One per NUMA Domain)
------------------------------------------------------

.. code-block:: bash

    #!/bin/bash -l
    #PBS -A electrolyte-chibueze
    #PBS -N gmx-8x16
    #PBS -q workq-route
    #PBS -l select=1:system=crux
    #PBS -l place=scatter
    #PBS -l walltime=24:00:00
    #PBS -l filesystems=home:eagle
    #PBS -k doe
    #PBS -j oe

    # Layout: sim0..sim7; each sim uses one NUMA domain (16 cores) as 4 MPI ranks x 4 OpenMP threads
    cd ${PBS_O_WORKDIR}
    module swap PrgEnv-cray PrgEnv-gnu
    GMX=/eagle/electrolyte-chibueze/${USER}/sw/gromacs-2026.4-crux/bin/gmx_mpi
    for i in 0 1 2 3 4 5 6 7; do
      b=$(( i * 16 ))
      LIST="${b}-$((b+3)):$((b+4))-$((b+7)):$((b+8))-$((b+11)):$((b+12))-$((b+15))"
      ( cd sim${i}
        mpiexec -n 4 --ppn 4 --cpu-bind list:${LIST} \
          --env OMP_NUM_THREADS=4 --env OMP_PROC_BIND=spread --env OMP_PLACES=cores \
          ${GMX} mdrun -deffnm md -ntomp 4 -pin inherit -cpi md.cpt -maxh 23.5 > mdrun.out 2>&1 ) &
    done
    wait

* ``OMP_NUM_THREADS`` defaults to 256 on Crux; always set it explicitly.
* For one large system on a whole node, use the official layout: ``mpiexec -n 64 --ppn 64 --depth=2 --cpu-bind depth --env OMP_NUM_THREADS=2 ...``.

Additional Documentation & References
=====================================
* **ALCF user guides:** `Polaris <https://docs.alcf.anl.gov/polaris/running-jobs/>`__ · `Aurora <https://docs.alcf.anl.gov/aurora/running-jobs-aurora/>`__ · `Aurora known issues <https://docs.alcf.anl.gov/aurora/known-issues/>`__ · `Crux <https://docs.alcf.anl.gov/crux/queueing-and-running-jobs/running-jobs/>`__ · `Storage <https://docs.alcf.anl.gov/data-management/filesystem-and-storage/>`__ · `sbank <https://docs.alcf.anl.gov/account-project-management/allocation-management/allocation-management/>`__
* If the docs site does not load, read the source in `argonne-lcf/user-guides <https://github.com/argonne-lcf/user-guides>`__.
