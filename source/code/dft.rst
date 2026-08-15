==================================================
Density Functional Theory (Gaussian) SOP
==================================================

Accessing and Navigating Midway
===============================

Terminal Access
---------------
To log into the RCC Midway3 cluster via terminal:

.. code-block:: bash

    ssh <CNET>@midway3.rcc.uchicago.edu

Enter your password and complete the 2FA verification prompt. Use ``Ctrl + d`` to terminate the SSH session and logout.

Browser Access (ThinLinc)
-------------------------
You can access Midway via a web browser at:

* **Midway 3:** ``midway3.rcc.uchicago.edu`` 
* **Midway 2:** ``midway2.rcc.uchicago.edu`` 

.. note::
    Using ThinLinc via a web browser allows you to launch Graphical User Interfaces (GUIs) like Jupyter Notebooks and GaussView directly on Linux remote nodes. On Windows and Mac terminals, GUIs cannot be rendered natively without X11 forwarding. Alternatively, download output files (e.g., ``.log``) to your local machine and run analysis scripts locally.

Running GaussView
-----------------
.. warning::
    The RCC holds a license for GaussView compatible **only with Gaussian 09**. ``.log`` and ``.chk`` files generated using Gaussian 16 cannot be opened directly in GaussView.

To launch an interactive session for GaussView:

.. code-block:: bash

    # Request an interactive compute node
    sinteractive --nodes=1 --cpus-per-task=4 --partition=broadwl --time=2:00:00 --account=pi-chibueze

    # Load module (Midway3)
    module load gaussian

    # Load module (Midway2 alternative)
    module load gaussian/09RevB.01

    # Launch GaussView
    gv <filename>

Mapping Midway Project Folders Locally (SAMBA)
----------------------------------------------
Mounting the network drive allows direct editing of cluster files on your local desktop:

* **Windows PC Setup**:
  
  1. Open File Explorer, navigate to *This PC*, click the three dots (``...``), and select **Map network drive**.
  2. Enter Folder: ``\\midwaysmb.rcc.uchicago.edu\project2`` 
  3. Enter Username: ``ADLOCAL\CNetID`` and your CNetID password.

* **Mac Setup**:
  
  1. Open Finder, select **Go** -> **Connect to Server...** (or ``Cmd + K``).
  2. Enter Server Address: ``smb://midway3smb1.rcc.uchicago.edu/project`` 
  3. Authenticate with ``ADLOCAL\CNetID`` and your CNetID password.

* **Lab Project Path:** ``/project2/chibueze/<CNET>`` (Always work out of the ``project2`` partition; do not store simulation runs in ``home`` or ``scratch``).

File Extensions
~~~~~~~~~~~~~~~
* ``.com`` / ``.gjf`` – Gaussian Input file (contains header keywords, charge, multiplicity, atomic coordinates).
* ``.log`` – Calculation output log file (contains energy values, optimization steps, thermodynamic data).
* ``.chk`` – Binary checkpoint file (contains wavefunctions, orbital densities, ESP maps).

Structure Generation & Input (.com) Setup
=========================================

Avogadro Input Preparation
--------------------------
1. Draw the desired molecular structure in Avogadro.
2. Select **Extensions** -> **Optimize Geometry** for initial molecular mechanics cleanup.
3. Select **Extensions** -> **Gaussian** to open the Gaussian input generator window.
4. Overwrite the top header section with standard lab parameters and click **Generate** to save the ``.com`` file.

Standard Coarse Header Specification
------------------------------------
.. code-block:: text

    %mem=64GB
    %nprocshared=32
    %chk=peg_1.chk
    # opt rb3lyp/6-31+g(d)

    Molecule description title line

    0 1
    C        -0.75112       1.15931       0.00000
    H        -1.10773       1.68331      -0.89010
    ...

Computational Resource Rules
----------------------------
* **Max CPU Cores:** Midway2 maximum is **28 CPUs**; Midway3 maximum is **48 CPUs**.
* **CPU Core Estimation:** Rules of thumb: CPUs = (number of atoms) / 2
* **Memory Allocation:** Calculate allocated memory as: mem = (CPUs x 2) + 1 GB. Ensure memory and CPU values match between the ``.com`` header and Slurm ``sbatch`` directives.

Functional & Basis Set Selection Guidelines
-------------------------------------------
* **Functional (Restricted vs. Unrestricted):**
  
  * ``rb3lyp`` – Closed-shell systems where all electrons are paired (neutral molecules).
  * ``ub3lyp`` – Open-shell systems with unpaired electrons (radicals, charged species).

* **Basis Sets:**
  
  * ``6-31+g(d)`` – Standard for **coarse** geometry relaxation (balances accuracy and performance).
  * ``6-311++g(d,p)`` – Standard for **fine** geometry relaxation and single-point energy runs.
  * ``lanl2dz`` – Effective core potential basis set used for heavy atoms beyond Krypton (atomic number > 36).

Charge and Multiplicity Rules
-----------------------------
Charge (c) and Spin Multiplicity (m) are specified as two integers on line 6 of the input file: ``c m``.

Spin multiplicity is calculated using total electron spin (s):

m = 2s + 1

* **Neutral systems:** Typically s = 0, which means m = 1 (Singlet).
* **Radical / Charged (+/- 1) systems:** Typically s = 1/2, which means m = 2 (Doublet).
* **Exceptions:** Complex ions (e.g., NH4+) maintain s = 0, which means m = 1 due to atom hybridization.

.. note::
    To confirm spin multiplicity for complex or large molecules, open the input file in GaussView:
    Go to **Calculate** -> **Gaussian Calculation Setup** -> adjust charge value, and GaussView will automatically calculate the ground-state spin multiplicity.

Structural Relaxation Workflow
==============================

Stage 1: Coarse Structure Relaxation
------------------------------------
Runs an initial geometry optimization using ``6-31+g(d)``.

**Slurm Script (Multi-job submission using list):**

.. code-block:: bash

    #!/bin/bash
    #SBATCH --job-name=rough_opt
    #SBATCH --account=pi-chibueze
    #SBATCH --output=outPC.out
    #SBATCH --error=errPC.err
    #SBATCH --time=18:00:00
    #SBATCH --partition=broadwl  # Note: Use 'broadwl' for Midway2, 'caslake' for Midway3
    #SBATCH --cpus-per-task=28
    #SBATCH --mem=56gb

    module load gaussian/16RevA.03
    cd ${PWD}

    for i in $(cat list)
    do 
        g16 $i
    done

Stage 2: Fine Structure Relaxation
----------------------------------
Refines coarse geometry using the expanded ``6-311++g(d,p)`` basis set:

1. Open the converged ``coarse.log`` file in Avogadro.
2. Export Gaussian input with fine parameters:

.. code-block:: text

    %mem=64GB
    %nprocshared=32
    %chk=peg_1_fine.chk
    # opt rb3lyp/6-311++g(d,p)

    peg_1_fine

    0 1
    <Atom Coordinates from coarse run>

3. Save as ``filename_fine.com`` and submit to Slurm queue.

Stage 3: Implicit Solvation
---------------------------
Applies implicit dielectric continuum solvation corrections using Polarizable Continuum Model (PCM) or SMD, along with Grimme's D3 dispersion corrections.

1. Open ``_fine.log`` in Avogadro and export a new Gaussian input file.
2. Include dispersion (``empiricaldispersion=gd3bj``) and solvation keywords (``scrf=pcm``).
3. Append dielectric constant (eps) at the bottom of the input file.

.. warning::
    Line spacing is critical! You **must** include a blank line between the final atomic coordinates and the dielectric constant, and another blank line after the dielectric line.

.. code-block:: text

    %chk=3fthf_fine_imp.chk
    %mem=24GB
    %nprocshared=12
    # opt rb3lyp/6-311++g(d,p) empiricaldispersion=gd3bj scrf=pcm

    3FTHF_fine_imp

    0 1
    <Atom Coordinates>

    eps=7.20
    

Stage 4: Mixed Basis Sets for Heavy Elements
--------------------------------------------
When a molecule contains both light elements (atomic number <= 35) and heavy elements (atomic number > 35, e.g., Cesium):

.. code-block:: text

    %mem=21GB
    %nprocshared=10
    %chk=fsi-cs_imp_fine.chk
    #t b3lyp/gen pseudo=read opt freq empiricaldispersion=gd3bj scrf=pcm

    FSI-Cs_imp_fine

    0 1
    <Atom Coordinates>

    N,S,O,F 0
    6-311++g(d,p)
    ****
    Cs 0
    lanl2dz
    ****

    Cs 0
    lanl2dz

    eps=7.20
    

Property & Thermodynamic Calculations
======================================

Binding Energy Calculations (Eb)
-----------------------------------
Calculates the binding energy of electrolyte complexes (e.g., solvent-Li+):

Eb (eV) = E(solvent+Li+) - E(solvent) - E(Li+)

All three configurations must be calculated using identical levels of theory and solvation models.

**Analysis Script (Jupyter Notebook):**

.. code-block:: python

    import glob
    from cclib.io import ccread  # Or custom lab GaussianOutput parser

    # Load completed output files
    log_list = glob.glob("*.log")
    done_list = [i for i in log_list if GaussianOutput(i).properly_terminated == True]

    # Extract final energies (Hartree / au)
    energy_free_li = GaussianOutput("li_fine_imp.log").final_energy
    energy_free_peg = GaussianOutput("peg_fine_imp.log").final_energy
    energy_peg_li = GaussianOutput("peg_li_fine_imp.log").final_energy

    # Compute binding energy and convert Hartrees to eV
    # Conversion factor: 1 au = 27.211324570273 eV
    bind_energy_au = energy_peg_li - energy_free_peg - energy_free_li
    bind_energy_ev = bind_energy_au * 27.211324570273

    print(f"Binding Energy: {bind_energy_ev:.4f} eV")

Redox Stability Analysis (Vox)
-------------------------------------------
Calculates the oxidation potential relative to the Li/Li+ reference electrode:

Vox (V) = -E0_system (eV) + E+_system (eV) - 1.4 = IE - E_ref(Li/Li+)

**Analysis Script:**

.. code-block:: python

    # Read orbital eigenvalues and total electron count
    energy_levels = GaussianOutput("peg_imp.log").eigenvalues
    num_elecs = GaussianOutput("peg_imp.log").electrons[0]

    # Index HOMO (num_elecs - 1) and LUMO (num_elecs)
    homo_ev = energy_levels[0][num_elecs - 1] * 27.211324570273
    lumo_ev = energy_levels[0][num_elecs] * 27.211324570273

    print(f"HOMO: {homo_ev:.4f} eV | LUMO: {lumo_ev:.4f} eV")

    # Calculate Oxidation Potential vs Li/Li+
    energy_peg0 = GaussianOutput("peg_neutral.log").final_energy
    energy_peg_plus1 = GaussianOutput("peg_oxidized.log").final_energy

    ionization_energy_ev = (-energy_peg0 + energy_peg_plus1) * 27.211324570273
    v_ox = ionization_energy_ev - 1.4

    print(f"Oxidation Potential (Vox): {v_ox:.4f} V vs Li/Li+")

Orbital & ESP Mapping Visualization
-----------------------------------
Molecular orbital rendering (HOMO/LUMO) and Electrostatic Potential (ESP) surfaces must be compiled using **Gaussian 09** to maintain compatibility with GaussView:

1. Modify ``.com`` header to append complete population analysis:

.. code-block:: text

    #P rb3lyp/6-311++g(d,p) empiricaldispersion=gd3bj pop=full

2. Execute using ``g09`` in Slurm script:

.. code-block:: bash

    module load gaussian/09RevD.01
    g09 input_orbital.com

3. **Visualizing orbitals in GaussView**:
   
   * Open output ``.chk`` file in GaussView.
   * Navigate to **Results** -> **Surfaces and Contours**.
   * Click **New Cube** -> Select **HOMO** or **LUMO** -> **New Surface**.

4. **Visualizing Electrostatic Potential (ESP)**:
   
   * Open ``.chk`` in GaussView.
   * Navigate to **Results** -> **Surfaces and Contours**.
   * Select **New Cube** (Type = Total Density).
   * Select **Surface** -> **New Mapped Surface** -> Select **ESP**.
   * Set display format to *Transparent* to view the underlying molecular skeleton.

Troubleshooting Common Errors
=============================

Diagnostic Commands
-------------------
* ``ls -ltr`` – Sorts files by modification time (helps identify newly created error logs).
* ``tail -f <filename.log>`` – Streams output updates in real-time.

SCF Non-Convergence Strategy
----------------------------
If a fine geometry optimization or solvent run fails to converge, initialize the initial guess wavefunction from a previously converged coarse step:

1. Copy the converged coarse checkpoint file to a new target checkpoint file:

.. code-block:: bash

    cp peg_coarse.chk peg_fine.chk

2. Modify the fine ``.com`` file header to read the wavefunctions from the copied checkpoint:

.. code-block:: text

    %chk=peg_fine.chk
    # opt guess=read rb3lyp/6-311++g(d,p)

Additional Documentation & References
=====================================
* **External Troubleshooting Reference:** `Gaussian Common Errors & Solutions <https://wongzit.github.io>`_ 

.. note::
    For a more detailed guide please refer to the `DFT training folder on Box <https://uchicago.box.com/s/s7w4ilk47rv0ap703k3zg2yxpq95i9jw>`_.