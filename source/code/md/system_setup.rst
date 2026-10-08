============
System Setup
============

Force-Field Parameters
======================

* **Solvents:** `LigParGen <https://jorgensenresearch.com/ligpargen/>`__ (OPLS-AA, organic molecules up to ~200 atoms). Use ``1.14*CM1A-LBCC`` for neutral molecules and select GROMACS output; if LBCC fails (e.g. amine-rich molecules), fall back to ``1.14*CM1A`` with 1–2 optimization rounds. CM1A charges keep only 4 decimals, so spread the residual over the atoms until the net charge is exactly 0, or hundreds of copies give a non-integer system charge.
* **Ions:** `CL&P <https://github.com/paduagroup/clandp>`__ + `fftool <https://github.com/paduagroup/fftool>`__ for FSI⁻, TFSI⁻, PF6⁻ and Li⁺/Na⁺. fftool reads the structure files and ``il.ff`` and writes the Packmol input; add ``--gmx`` for GROMACS files. Both use the OPLS convention ``[ defaults ] 1 3 yes 0.5 0.5``, so their atomtypes merge directly.
* **Scale ion charges by 0.8** (ECC), ion atoms only (Li⁺ +0.8, FSI⁻ −0.8; solvent unchanged). Full ±1 charges in a non-polarizable force field overestimate ion–ion electrostatics and slow the dynamics, so short-trajectory MSDs often never reach the diffusive regime. Never mix charge scales within one campaign; record ``charge_scale`` with every result.

.. warning::
    **Verify that each force-field file really is the intended molecule.** Do not trust file names, cache keys or recorded SMILES: for every itp/gro, rebuild the formula and molecular graph from elements and bonds, compare it with the SMILES, and record the file content with ``sha256sum``.

Merging Topologies
==================

* **Prefix atomtypes per molecule.** LigParGen numbers every molecule's types from ``opls_800``; merging two solvents silently overwrites same-named types with the other molecule's LJ parameters, and grompp does not warn. Rename only column 1 (name) of ``[ atomtypes ]`` (e.g. ``opls_800`` → ``dme_opls_800``) and column 2 (type) of ``[ atoms ]``. Leave column 2 (bond_type) of ``[ atomtypes ]``; LigParGen writes all bonded parameters explicitly.
* Keep only ``[ moleculetype ]`` and below in each molecule itp. Move ``[ defaults ]``/``[ atomtypes ]`` (including those from fftool/CL&P) and the trailing ``[ system ]``/``[ molecules ]`` into ``system.top``, one copy of each.
* The ``[ molecules ]`` order must match the Packmol ``structure`` order (the coordinate order in the gro).

.. code-block:: text

    [ defaults ]
    ; nbfunc comb-rule gen-pairs fudgeLJ fudgeQQ   (OPLS-AA)
      1      3         yes       0.5     0.5

    [ atomtypes ]
    #include "atomtypes_merged.itp"   ; all types, prefixed per molecule

    #include "dme.itp"
    #include "li.itp"                 ; Li charge +0.8
    #include "fsi.itp"                ; FSI net charge -0.8

    [ system ]
    1 M LiFSI in DME

    [ molecules ]
    DME   667
    LI     75
    FSI    75

Fixed-Concentration Packing (Packmol)
=====================================

Concentration always means mol/L of the solution **after NPT** (not molality or salt:solvent molar ratio). Use a box edge ≥5 nm with at least ~40 salt pairs; the edge must also exceed 2× the cutoff.

.. code-block:: text

    N_pair             = round(c[M] × 0.6022 × V[nm³])        1 M, (5 nm)³ → 75 pairs
    v_solv[nm³]        = M_w[g/mol] / ρ[g/cm³] / 602.2
    n_solv (1st pass)  = (V − N_pair × v_pair) / v_solv         v_pair: LiFSI ≈ 0.135 nm³, NaPF6 ≈ 0.110 nm³
    after NPT          n_solv += (V_target − V_NPT) / v_solv    one round is usually enough
    check              c = N_pair / (0.6022 × V_NPT)            tolerance 1.5–3%

* Without the salt-volume term the post-NPT concentration comes out systematically high (more so at higher concentration).
* Correct only after the volume has plateaued (mean volumes of the two halves of the tail differ by <0.5%); otherwise first extend ``npt_c`` (the 1 ns compression NPT after EM; see :doc:`/code/md/running`). An unconverged volume can push the solvent count the wrong way.

Example: 1 M LiFSI/DME in a 5 nm box is 75 LiFSI pairs and ~667 DME. Pack at ~70% density: packing edge = target edge × 1.12 (5.6 nm = 56 Å), with a 1 Å margin on each side. Use a different seed per replica (``seed -1`` derives it from the system time, or set an integer and record it).

.. code-block:: text

    tolerance 2.0
    filetype pdb
    output mixture.pdb
    seed -1
    nloop 200
    movebadrandom

    structure dme.pdb
      number 667
      inside box 1. 1. 1. 55. 55. 55.
    end structure
    structure li.pdb
      number 75
      inside box 1. 1. 1. 55. 55. 55.
    end structure
    structure fsi.pdb
      number 75
      inside box 1. 1. 1. 55. 55. 55.
    end structure

.. code-block:: bash

    packmol < pack.inp        # non-zero exit "ENDED WITHOUT PERFECT PACKING" is acceptable
    grep -cE '^(ATOM|HETATM)' mixture.pdb    # atom count must match: 667*16 + 75*1 + 75*9 = 11422
    gmx editconf -f mixture.pdb -o mixture.gro -box 5.6 5.6 5.6

Additional Documentation & References
=====================================

* **Software:** `LigParGen <https://jorgensenresearch.com/ligpargen/>`__ · `CL&P <https://github.com/paduagroup/clandp>`__ / `fftool <https://github.com/paduagroup/fftool>`__ · `Packmol <https://m3g.github.io/packmol/userguide.shtml>`__
* **Force fields:** Dodda et al., NAR 2017 (LigParGen, `doi:10.1093/nar/gkx312 <https://doi.org/10.1093/nar/gkx312>`__); Canongia Lopes & Pádua, JPCB 2004 (CL&P, `doi:10.1021/jp0362133 <https://doi.org/10.1021/jp0362133>`__)
* **Charge scaling:** Leontyev & Stuchebrukhov, PCCP 2011 (`doi:10.1039/c0cp01971b <https://doi.org/10.1039/c0cp01971b>`__); Doherty et al., JCTC 2017 (`doi:10.1021/acs.jctc.7b00520 <https://doi.org/10.1021/acs.jctc.7b00520>`__)
