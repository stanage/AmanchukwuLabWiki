========
Analysis
========

Run analysis as cluster jobs and bring back only KB–MB JSON/CSV.

Commands
========

.. code-block:: bash

    echo Density | gmx energy -f npt_eq.edr -o density.xvg    # first check
    echo 0 | gmx trjconv -f prod.xtc -s prod.tpr -pbc nojump -b 1000 -o prod_nojump.xtc   # drop the first 1 ns
    # Self-diffusion. Fit window (lag time, ps) = where the log-log slope is ~1; here ~10-40% of the trajectory
    gmx msd -f prod_nojump.xtc -s prod.tpr -sel "resname LI" -o msd_li.xvg -beginfit 3000 -endfit 12000
    gmx msd -f prod_nojump.xtc -s prod.tpr -sel "resname FSI" -mol diff_fsi.xvg -o msd_fsi.xvg \
            -beginfit 3000 -endfit 12000                       # -mol: per-molecule COM, one group only
    # Li-donor RDF and cumulative CN: -sel donor atoms only; wildcards must be double-quoted
    gmx rdf -f prod.xtc -s prod.tpr -b 1000 -ref "resname LI" \
            -sel 'resname FSI and name "O*"' 'resname DME and name "O*"' \
            -bin 0.002 -rmax 1.0 -o rdf_li.xvg -cn cn_li.xvg

.. code-block:: text

    D_i      = slope(MSD_i) / 6
    σ_NE     = e² / (V k_B T) · Σ_i N_i z_i² D_i
    σ_EH     = e² / (6 V k_B T) · d/dt ⟨|Σ_j z_j Δr_j(t)|²⟩    (Einstein–Helfand, collective)
    ionicity = σ_EH / σ_NE      (<1: net cation–anion correlation)
    t+(NE)   = N₊D₊ / (N₊D₊ + N₋D₋)
    D_∞      = D_PBC + 2.837297 · k_B T / (6π η L)              (Yeh–Hummer, cubic box, corrects D only)

**Units:** plug in SI (e = 1.602×10⁻¹⁹ C, V in m³, D in m²/s) to get S/m; ×10 gives mS/cm. ``gmx msd`` reports D in 10⁻⁵ cm²/s = 10⁻⁹ m²/s.

Validity Checks
===============

* **Density:** expected ρ [g/cm³] ≈ (N_pair·M_salt + n_solv·M_solv) / (602.2·V [nm³]). 1 M Li salts: ethers ≈ 0.95–1.1 (1 M LiFSI/DME ≈ 0.99), carbonates ≈ 1.2–1.35 g/cm³. More than ~3–5% off experiment: suspect the box or topology first.
* **MSD:** in the fit window the log-log slope β must be 0.9–1.1 and the Li RMS displacement ≥ 1 nm, otherwise D is noise. Anions: use the COM D from ``-mol`` (``diff_fsi.xvg``); Li⁺/Na⁺ are single atoms, so use the ``-o`` fit.
* **Charge convention:** group convention is formal charge z = ±1. On q = 0.8 trajectories, using ±0.8 instead changes σ by a factor of 0.64; state the convention with the result.
* **Which conductivity:** always state σ_EH or σ_NE. σ_NE ignores ion–ion correlations and can exceed σ_EH several-fold in strongly associated systems. Compare σ_EH (mean of ≥ 3 replicas; much noisier) with experiment; report σ_NE as the uncorrelated reference. Check ionicity against the CIP/AGG fractions.
* **Errors:** standard error over ≥ 3 independent replicas. To block-average one trajectory, use blocks ≥ ~2× the fit-window upper limit (e.g. 30 ns in 2 blocks, fit 1.5–6 ns) and re-check β ≈ 1 in each block.
* **Yeh–Hummer:** at L = 5 nm, 298 K, η = 1 mPa·s the correction is ~+1.2×10⁻⁶ cm²/s, the same order as D. η must be the force field's own viscosity. Preferred: compute D at 2–3 box sizes (e.g. 4/5/6 nm) and extrapolate linearly in 1/L to D_∞ (the slope gives the model η); experimental η is an approximation and must be stated. Always compute σ_NE and ionicity from the uncorrected D_PBC.
* **Green–Kubo viscosity:** tens of ns usually do not converge; it needs multiple, longer replicas (Maginn et al. 2019). Treat η from short runs as noise.

RDF, CN and Ion Pairing
=======================

* ``-sel`` donor atoms only (O, N, F; F for PF6). With the whole anion, g(r) is normalized by all anion atoms (contact peak ~2× too low for FSI) and ``-cn`` counts atoms rather than donors, so CN multiplies once the cutoff passes the S/N peak.
* Cutoff = first minimum after the contact peak, with g(r_min) < 0.5 × peak height.
* **SSIP/CIP/AGG** (anion view): for each anion, count distinct cations within the cutoff of any of its donor atoms: 0 = SSIP, 1 = CIP, ≥ 2 = AGG.

.. warning::
    Do not find the "first peak" with argmax. In highly dissociated systems the contact peak is lower than the second-shell peak, so the cutoff lands beyond the second shell and most SSIPs are misclassified as CIP/AGG. Always scan the cutoff ±0.5 Å for sensitivity.

The same quantities with SolvationAnalysis (built on MDAnalysis):

.. code-block:: python

    import MDAnalysis as mda
    from solvation_analysis import Solute
    u = mda.Universe("npt_selected.gro", "prod.xtc")    # use gro, not tpr
    li  = u.select_atoms("resname LI")
    fsi = u.select_atoms("byres resname FSI")
    dme = u.select_atoms("byres resname DME")
    solute = Solute.from_atoms(li, {"FSI": fsi, "DME": dme}, solute_name="Li",
                               radii={"FSI": 2.8, "DME": 2.8})   # Å, from the first RDF minimum
    solute.run(start=200)                                # skip the first 1 ns (5 ps/frame)
    print(solute.coordination.coordination_numbers)
    print(solute.speciation.speciation_fraction)

.. note::
    The MDAnalysis 2.10 TPR reader only supports tpr files up to the GROMACS 2025.0 format, so analysis scripts always read gro + xtc.

Common Pitfalls
===============

* **Skipping EM** gives LINCS errors, then a segfault; **default** ``nstpcouple`` **in compression** makes the box oscillate and crash (see :doc:`/code/md/running`).
* **Unprefixed atomtypes:** LJ parameters are silently overwritten in multi-solvent systems; single-solvent systems are unaffected, so it goes unnoticed for a long time (see :doc:`/code/md/system_setup`).
* **Non-integer net charge:** check grompp's ``System has non-zero total charge`` warning; never suppress it with ``-maxwarn``.
* **Mixing** ``charge_scale``, ``DispCorr``, timestep or HMR within one campaign: each shifts results systematically.
* ``gmx energy`` **with several terms:** xvg columns follow the edr's internal order, not the order you typed; parse the ``@ s<N> legend`` lines.
* **Job fails after ~5 s with exit 127:** almost always a module not loaded. GROMACS writes errors to stderr, so read ``.err`` / ``mdrun.out``.
* **Killed at walltime:** add ``-cpi xxx.cpt -maxh <slightly below walltime>`` and resubmit to continue.

Additional Documentation & References
=====================================

* Maginn et al., LiveCoMS 2019 (transport best practices): `doi:10.33011/livecoms.1.1.6324 <https://doi.org/10.33011/livecoms.1.1.6324>`__
* Yeh & Hummer, JPCB 2004 (finite-size correction of D): `doi:10.1021/jp0477147 <https://doi.org/10.1021/jp0477147>`__
* Fong et al., Macromolecules 2021 (ion correlations and conductivity): `doi:10.1021/acs.macromol.0c02545 <https://doi.org/10.1021/acs.macromol.0c02545>`__
* SolvationAnalysis: `documentation <https://solvation-analysis.readthedocs.io/en/latest/>`__; Cohen et al., JOSS 2023, `doi:10.21105/joss.05183 <https://doi.org/10.21105/joss.05183>`__
* MDAnalysis TPR format support: `TPRParser documentation <https://docs.mdanalysis.org/stable/documentation_pages/topology/TPRParser.html>`__
* Our group: Kumar, Vu, Ma, Amanchukwu, Chem. Mater. 2025, `doi:10.1021/acs.chemmater.4c03196 <https://doi.org/10.1021/acs.chemmater.4c03196>`__
