===========
RCC Midway3
===========

Slurm job templates for GROMACS on Midway3, charged to ``--account=pi-chibueze``. For the MD workflow itself see :doc:`/code/md_index`. The templates run ``mdrun -deffnm md`` in each run directory, so copy the stage tpr in as ``md.tpr`` (e.g. ``prod.tpr``).

Login & Storage
===============

Log in with ``ssh <CNetID>@midway3.rcc.uchicago.edu`` (CNetID password, then Duo). Do not compute on login nodes; compute nodes have no internet access.

* ``/project/chibueze/$USER`` – group project space, snapshotted (7 daily + 4 weekly). Keep runs here, never in home.
* ``/project2/chibueze`` – old Midway2 space, if the group has one (check with ``rcchelp quota``); also mounted on Midway3.
* ``/scratch/midway3/$USER`` – temporary, 100 GB soft / 2 TB hard, no backup.
* ``/home/$USER`` – 30 GB; do not keep trajectories here.
* **Transfers:** Globus collection "UChicago RCC Midway3".

Partitions & Modules
====================

* ``gpu`` – 4 GPUs / 48 cores per node: V100 and RTX6000 nodes plus a single A100 node; select the type with ``--constraint=`` (e.g. ``--constraint=v100`` or ``rtx6000``). At most 12 jobs per user.
* ``caslake`` – 48 cores / 192 GB per node. Both partitions allow at most 36 h walltime.
* **GROMACS modules:** ``gromacs/2020.4`` (default), ``2021.5``, ``2022.4``, ``2022.4-colvars``. RCC says to use ``gromacs/2021.5`` (OpenMPI 4.1.2 + CUDA 11.2, provides ``gmx_mpi``) on GPU nodes. Check ``module avail gromacs`` for newer versions; RCC also encourages building the latest release in your own space.

.. warning::
    The newest module (2022.4) does not support ``mass-repartition-factor`` (GROMACS 2024+); to use HMR, build a newer GROMACS yourself. Run ``grompp`` on Midway3 with the same module as ``mdrun`` (see the tpr version note in :doc:`/code/md/running`).

GPU: Single Simulation
======================

.. code-block:: bash

    #!/bin/bash
    #SBATCH --job-name=gmx-gpu
    #SBATCH --account=pi-chibueze
    #SBATCH --partition=gpu
    #SBATCH --gres=gpu:1
    #SBATCH --nodes=1
    #SBATCH --ntasks=1
    #SBATCH --cpus-per-task=12
    #SBATCH --time=24:00:00
    #SBATCH --output=%x-%j.out
    #SBATCH --error=%x-%j.err

    module load gromacs/2021.5
    cd $SLURM_SUBMIT_DIR
    export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
    # OpenMPI binds to core by default for np<=2; without --bind-to none all 12 threads share 1 core
    mpirun -np 1 --bind-to none gmx_mpi mdrun -deffnm md -ntomp $SLURM_CPUS_PER_TASK \
        -nb gpu -pme gpu -bonded gpu -update gpu -cpi md.cpt -maxh 23.5

If ``-update gpu`` is not supported, use ``-update cpu``. After the run, ``md.log`` should report 12 OpenMP threads and no thread-affinity warnings.

CPU: Full caslake Node
======================

.. code-block:: bash

    #!/bin/bash
    #SBATCH --job-name=gmx-cpu
    #SBATCH --account=pi-chibueze
    #SBATCH --partition=caslake
    #SBATCH --nodes=1
    #SBATCH --ntasks-per-node=48
    #SBATCH --cpus-per-task=1
    #SBATCH --exclusive
    #SBATCH --time=24:00:00
    #SBATCH --output=%x-%j.out
    #SBATCH --error=%x-%j.err

    module load gromacs/<cpu-build>      # see notes below
    cd $SLURM_SUBMIT_DIR
    export OMP_NUM_THREADS=1
    mpirun -np $SLURM_NTASKS gmx_mpi mdrun -deffnm md -ntomp 1 -pin on -nb cpu -cpi md.cpt -maxh 23.5

* Do not add ``--ntasks=1``: it overrides ``--ntasks-per-node``, so you pay for the whole node but start only 1 rank.
* If a small box fails with a domain decomposition error, use ``--ntasks-per-node=12 --cpus-per-task=4``, ``OMP_NUM_THREADS=4`` and ``-ntomp 4``.
* ``gromacs/2021.5`` is a CUDA build for GPU nodes. For caslake, find a non-CUDA build with ``module show gromacs/2020.4`` (or ``2022.4``), or compile your own. Test a new build first with a short debug-QoS run (see Common Commands).

Job Array: Many Formulations at Once
====================================

``systems.txt`` lists one formulation directory per line; run ``mkdir -p logs`` before submitting. ``%12`` caps running tasks at 12 (the ``gpu`` job limit); check ``rcchelp qos`` for the running vs. submitted limit.

.. code-block:: bash

    #!/bin/bash
    #SBATCH --job-name=gmx-array
    #SBATCH --account=pi-chibueze
    #SBATCH --partition=gpu
    #SBATCH --gres=gpu:1
    #SBATCH --ntasks=1
    #SBATCH --cpus-per-task=12
    #SBATCH --time=24:00:00
    #SBATCH --array=1-40%12
    #SBATCH --output=logs/%x_%A_%a.out

    module load gromacs/2021.5
    cd $SLURM_SUBMIT_DIR
    SYS=$(sed -n "${SLURM_ARRAY_TASK_ID}p" systems.txt)
    cd "$SYS" || exit 1
    export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
    mpirun -np 1 --bind-to none gmx_mpi mdrun -deffnm md -ntomp $SLURM_CPUS_PER_TASK \
        -nb gpu -pme gpu -bonded gpu -update gpu -cpi md.cpt -maxh 23.5

Common Commands
===============

.. code-block:: bash

    rcchelp balance ; accounts usage -a pi-chibueze -byuser     # SU balance and usage
    rcchelp qos ; rcchelp quota                                 # partition limits, storage quotas
    sinteractive --account=pi-chibueze --partition=gpu --gres=gpu:1 --cpus-per-task=12 --time=01:00:00
    sinteractive --qos=debug --time=00:15:00 --ntasks=2 --account=pi-chibueze   # <=4 cores, 15 min, no SU charge
    sbatch job.sbatch ; squeue --user=$USER ; squeue --user $USER --start
    sacct -j <jobid> ; scancel <jobid> ; scancel <jobid>_[3-5]
    sbatch --dependency=afterok:<jobid> next.sbatch            # run after the previous job succeeds

Additional Documentation & References
=====================================
* **RCC user guide:** `Storage <https://docs.rcc.uchicago.edu/storage/main/>`__ · `Partitions <https://docs.rcc.uchicago.edu/slurm/partitions/>`__ · `GROMACS <https://docs.rcc.uchicago.edu/software/apps-and-envs/gromacs/>`__ · `Allocations <https://docs.rcc.uchicago.edu/allocations/>`__
