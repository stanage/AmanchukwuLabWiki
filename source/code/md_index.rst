Molecular Dynamics (GROMACS)
============================

These pages cover classical MD of liquid Li/Na electrolytes (salt + solvent) with GROMACS, from formulation to transport and solvation analysis. For where and how jobs run (Polaris, Aurora, Crux, Midway3), see :doc:`/code/hpc_index`.

Overall workflow:

1. Define the formulation: salt/solvent SMILES, concentration, temperature.
2. Generate force fields (LigParGen for solvents, CL&P for ions) and confirm each file really is the intended molecule (:doc:`/code/md/system_setup`).
3. Merge topologies (prefix atomtypes, scale ion charges ×0.8) and pack with Packmol at ~70% density (:doc:`/code/md/system_setup`).
4. Run EM (``em``), then NPT compression (``npt_c``); if the concentration is out of tolerance, change the solvent count and repack (:doc:`/code/md/running`).
5. Anneal with NVT at 450 K for 10 ns (``nvt_hot``), cool down (``nvt_cool``), then equilibrate with NPT (``npt_eq``) (:doc:`/code/md/running`).
6. Run NVT production (``prod``) from the frame whose volume is closest to the NPT average (:doc:`/code/md/running`).
7. Compute density, D, σ and solvation structure (:doc:`/code/md/analysis`).
8. Archive trajectories: copy, verify, then delete (:doc:`/code/md/running`).

.. note::
    Residue and atom names (``LI``, ``FSI``, ``DME``, ``O*``) in these pages are examples; match them to your own itp files.

.. toctree::
   :maxdepth: 1

   md/system_setup
   md/running
   md/analysis
