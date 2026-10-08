===================
Running Simulations
===================

Stage MDP Files
===============
Every stage is built from the production ``prod.mdp`` below; the table lists only what changes.

.. code-block:: ini

    ; prod.mdp: NVT production, 2 fs, 30 ns
    integrator           = md
    dt                   = 0.002
    nsteps               = 15000000
    cutoff-scheme        = Verlet
    nstlist              = 20
    pbc                  = xyz
    coulombtype          = PME
    rcoulomb             = 1.2
    fourierspacing       = 0.12
    pme-order            = 4
    vdwtype              = Cut-off
    vdw-modifier         = Potential-shift
    rvdw                 = 1.2
    DispCorr             = EnerPres
    constraints          = h-bonds
    constraint-algorithm = lincs
    lincs-order          = 4
    lincs-iter           = 1
    tcoupl               = V-rescale
    tc-grps              = System
    tau-t                = 1.0
    ref-t                = 298.15
    pcoupl               = no
    gen-vel              = yes
    gen-temp             = 298.15
    gen-seed             = -1
    nstcalcenergy        = 100
    nstenergy            = 100
    nstlog               = 5000
    nstxout-compressed   = 2500    ; one frame every 5 ps

.. list-table::
   :widths: 22 78
   :header-rows: 1

   * - Stage (length)
     - Changes relative to ``prod.mdp``
   * - ``em`` (5000 steps)
     - ``integrator=steep``, ``nsteps=5000``, ``emtol=500``, ``emstep=0.005``; delete the constraint, thermostat and velocity lines
   * - ``npt_c`` (compression, 1 ns)
     - ``nsteps=500000``, ``tau-t=0.5``; ``pcoupl=C-rescale``, ``pcoupltype=isotropic``, ``tau-p=5.0``, ``nstpcouple=10``, ``nstcalcenergy=10``, ``ref-p=1.0``, ``compressibility=4.5e-5``
   * - ``nvt_hot`` (10 ns)
     - ``nsteps=5000000``, ``tau-t=0.5``, ``ref-t=450``, ``gen-temp=450``
   * - ``nvt_cool`` (1 ns)
     - ``nsteps=500000``, ``tau-t=0.5``, ``gen-vel=no``, ``continuation=yes``, ``annealing=single``, ``annealing-npoints=2``, ``annealing-time=0 1000``, ``annealing-temp=450 298.15``
   * - ``npt_eq`` (3 ns)
     - ``nsteps=1500000``, ``tau-t=0.5``, ``gen-vel=no``, ``continuation=yes``; barostat as in ``npt_c`` but ``tau-p=2.0``; ``nstxout-compressed=5000`` (10 ps per frame, for frame selection)
   * - ``prod`` (≥30 ns)
     - The full mdp above, started from the average-volume frame; discard the first 1 ns in analysis (``-b 1000``, see :doc:`/code/md/analysis`)

.. warning::
    Always set ``nstpcouple=10`` in the compression stage. The default (−1, i.e. 100 steps) with a short ``tau-p`` makes a 70%-density box oscillate and crash.

Equilibration Notes
===================
* **Anneal:** at room temperature Li⁺ first-shell exchange can be slower than the whole run; without the 450 K stage, production keeps Packmol's random initial coordination.
* **Check (every new system):** run from two starting points (random / ion pre-paired). It is equilibrated when, at the end of production, the associated fraction (CIP+AGG, see :doc:`/code/md/analysis`) differs by ≤0.05 and the anion coordination number by ≤0.1; otherwise run longer or anneal hotter.
* **Length:** ≥30 ns per replica at ≤2 M, 50–100 ns at 3–4 M or for viscous systems; at least 3 independent replicas (different Packmol and velocity seeds).
* **Transport (D, σ, η):** always 2 fs, no HMR, because HMR changes masses and therefore dynamics (Hopkins et al., JCTC 2015). 4 fs + HMR (``mass-repartition-factor=3``, GROMACS 2024+) is fine for structure/thermodynamics only.
* Keep ``nstenergy`` at ~100 steps; use 10–20 fs only for Green–Kubo viscosity (noticeably slower).

Run Chain
=========

.. code-block:: bash

    set -euo pipefail
    GMX=gmx                     # MPI build: mpiexec -n 1 gmx_mpi, and drop -ntmpi
    MD="-ntmpi 1 -ntomp 8 -nb gpu -pme gpu -bonded gpu -update gpu -pin on"

    $GMX grompp -f em.mdp       -c mixture.gro  -p system.top -o em.tpr
    $GMX mdrun  -deffnm em -ntmpi 1 -ntomp 8
    $GMX grompp -f npt_c.mdp    -c em.gro       -p system.top -o npt_c.tpr
    $GMX mdrun  -deffnm npt_c $MD                # check concentration; if off, change solvent count and repack
    $GMX grompp -f nvt_hot.mdp  -c npt_c.gro    -p system.top -o nvt_hot.tpr
    $GMX mdrun  -deffnm nvt_hot $MD
    $GMX grompp -f nvt_cool.mdp -c nvt_hot.gro  -t nvt_hot.cpt  -p system.top -o nvt_cool.tpr
    $GMX mdrun  -deffnm nvt_cool $MD
    $GMX grompp -f npt_eq.mdp   -c nvt_cool.gro -t nvt_cool.cpt -p system.top -o npt_eq.tpr
    $GMX mdrun  -deffnm npt_eq $MD
    # select the average-volume frame -> npt_selected.gro (next section)
    $GMX grompp -f prod.mdp -c npt_selected.gro -p system.top -o prod.tpr
    $GMX mdrun  -deffnm prod $MD -cpi prod.cpt -maxh 23.5   # -maxh slightly below walltime
    grep Performance *.log

.. warning::
    Never skip EM. MD started directly from the Packmol box fails with LINCS/SETTLE errors, then segfaults.

* Run ``grompp`` with the same GROMACS version as ``mdrun``, on the cluster where the job runs: an older ``mdrun`` cannot read a tpr written by a newer ``grompp``.
* Judge speed only by ``Performance: ns/day`` in ``md.log``, not by ``nvidia-smi`` utilization.

Average-Volume Frame Selection
==============================
The same script run on ``npt_c.edr`` is the concentration check (to fix the solvent count, see :doc:`/code/md/system_setup`).

.. code-block:: bash

    echo Volume | gmx energy -f npt_eq.edr -o vol.xvg
    python3 - <<'EOF'
    import numpy as np
    t, v = np.loadtxt("vol.xvg", comments=("#", "@"), unpack=True)   # ps, nm^3
    tail = t >= 0.5 * t[-1]; vm = v[tail].mean()
    print("c =", 75 / (0.6022 * vm), "M")                             # 75 = number of ion pairs
    on_xtc = tail & (np.abs(t / 10 - np.round(t / 10)) < 1e-4)        # xtc has one frame per 10 ps
    i = np.argmin(np.abs(v[on_xtc] - vm)); print("dump t =", t[on_xtc][i], "ps")
    EOF
    echo 0 | gmx trjconv -f npt_eq.xtc -s npt_eq.tpr -dump <t> -o npt_selected.gro

Job Templates
=============
Batch templates are in :doc:`/code/hpc/alcf` and :doc:`/code/hpc/rcc_midway3`. They run ``mdrun -deffnm md`` in each run directory (``sim*`` on ALCF), so copy the stage tpr in as ``md.tpr`` (e.g. ``prod.tpr``).

Record Keeping
==============
* Record for every run: GROMACS version, sha256 of the mdp and force-field files, per-stage seeds, ``charge_scale``, job id, final density, and the conductivity definition (σ_EH / σ_NE).
* Archive trajectories as **copy, verify, then delete** (Eagle and Flare are not backed up). Ask the PI for the destination; keep Globus "verify file integrity after transfer" checked and delete the source only after the transfer shows SUCCEEDED.

Additional Documentation & References
=====================================
* `GROMACS manual <https://manual.gromacs.org/current/>`__
* Bussi, Donadio & Parrinello, JCP 2007 (V-rescale thermostat): `doi:10.1063/1.2408420 <https://doi.org/10.1063/1.2408420>`__
* Bernetti & Bussi, JCP 2020 (C-rescale barostat): `doi:10.1063/5.0020514 <https://doi.org/10.1063/5.0020514>`__
* Hopkins et al., JCTC 2015 (hydrogen mass repartitioning): `doi:10.1021/ct5010406 <https://doi.org/10.1021/ct5010406>`__
