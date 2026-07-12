===========================
Plotting NMR and Raman Data
===========================

This module provides standard protocols for converting raw NMR and Raman outputs into publication-ready figures using standardized baseline correction, calibration, and fitting procedures.

Plotting Raw NMR Data
=====================

**Goal:** Plot the Li shift and F shift peaks for electrolyte systems. 

.. note:: 
    Pure solvents are typically analyzed via NMR solely to confirm the complete absence of impurities). 

Required Software:
------------------
* **MestReNova:** Search Google for "UChicago Mestrenova" and utilize the institutional campus license to download the software to your personal computer. 
* **Origin and Microsoft Excel.**

Step-by-Step Processing:
------------------------
1. **Open the File:** Load your raw FID file directly into MestReNova. 
2. **Identify Reference Peaks:** Locate your target reference peaks based on these standard behaviors:
   
   * **Li Peak:** Located around -2.8 ppm ; this peak is typically stronger than the active signal peak. 
   * **F Peak:** Located around -62.5 ppm ; this peak is typically weaker than the active signal peak. 

3. **Validation:** The observed position should sit close to established literature values. 
4. **Calibrate Spectra:** Select the Reference Peak Picker tool (located in the upper left corner of the interface) and click on your reference peak. Type in the precise literature value, or click the button in the bottom right corner to automatically select a peak based on your specific NMR solvent (e.g., Acetonitrile). This calibrates the scale across your entire spectrum. 
5. **Export to CSV:** Right-click on the spectrum window and select See Full Spectrum to maximize the x-axis limits. Ensure the entire calibrated spectrum is active, navigate to Save As, and choose NMR CSV File (CSV, text). 
6. **Format in Excel:** Open the exported text file in Microsoft Excel. Select the data column, navigate to Data → Text to Columns, and convert the raw text string into two distinct data columns. 
7. **Plot in Origin:** Copy the formatted two-column dataset into an Origin worksheet to generate your final stacked spectra plots. 

Plotting Raw Raman Data
=======================

**Goal:** Observe the fraction of ion pairing and evaluate the solution structure of your electrolyte blends. Standard Deliverable Figures: (1) Wavenumber vs. Peak Intensity, (2) Salt Concentration vs. Change of Normalized Peak Area, and (3) Raman Shift profiles detailing raw data overlaid with fitted peaks. 

**Required Software:** Origin (used for raw data visualization, manual baseline subtraction, and peak deconvolution). 

Understanding File Labeling (e.g., Christina-01262021-PFTMCH3-1_2MFSA-1_01)
---------------------------------------------------------------------------
* **-1 Tag:** Indicates the primary region containing critical FSA and solvent peaks. Import these text files into Origin. 
* **-2 Tag:** Indicates the C-H stretching region. Typically, these files are omitted from standard workflows. 
* **broadscan Tag:** Indicates a quick scan across the entire spectrum at a lower resolution. Avoid using these for final quantitative peak fitting. 

Step-by-Step Processing & Peak Fitting:
---------------------------------------
1. **Import Raw Data:** Import your targeted -1 text file into a new sheet inside an Origin workbook. 
2. **Isolate the Region of Interest:** Generate a basic line plot from the dataset. Select the Data Selector tool from the primary toolbar and drag the boundary end markers to capture the 700–1000 cm⁻¹ window. This is the standard range used for baseline calibration and structural analysis. 
3. **Open Peak Analyzer:** Click directly on the plotted data line to select it, then navigate to Analysis → Peaks & Baseline → Peak Analyzer → Open Dialog. 
4. **Configure the Goal:** In the wizard window, select Goal: Fit Peaks and click Next. 
5. **Establish Baseline Points:** Set Baseline Mode = User Defined. Select the manual creation options and click three separate points along the clear valley bottoms of your spectrum to establish the baseline anchor points. Click Next. 
6. **Subtract Baseline:** For Create Baseline, select an interpolation connection method and set it to Line. Once applied, the software automatically subtracts this baseline from your raw data and generates a new tab in your spreadsheet named "Subtracted Baseline". 
7. **Deconvolve/Find Peaks:** In the Find Peaks window, manually add component markers to each peak hidden within the spectrum. Crucial Rule: Maintain strict consistency across all related electrolyte samples. If you apply 3 fitting points to the primary FSA curve on your first sample, use the exact same layout for subsequent samples. For enhanced accuracy, consider fitting the FSA and solvent peak regions as separate multi-curve clusters. 
8. **Execute Fit Control:** Under Fit Peaks, individual Gaussian curves will generate beneath your raw trace. Click Fit. If the (x,y) coordinates of your manual components shift erratically during iterations, open the Fit Control settings, click the center icon among the bottom three options, and select Lock Peaks to stabilize the optimization. 
9. **Export Properties:** Click Finish. The raw data sheet will update to display the final baseline-subtracted data, the cumulative fit curve traces, and a comprehensive Peak Properties data tab. 

Generating the 3 Core Graphs:
-----------------------------
1. **Graph 1 (Wavenumber vs. Peak Intensity):** Group and plot the baseline-subtracted data lines together for all evaluated electrolyte compositions. 
2. **Graph 2 (Salt Concentration vs. Normalized Peak Area):** Navigate to the Peak Properties tab and isolate the Area Fit column. Normalize the area of each individual sub-peak against the total cumulative area of all peaks within that specific spectrum. Plot these normalized areas against salt concentration to draw quantitative sample comparisons. 
3. **Graph 3 (Raman Shift Overlays):** Plot the baseline-subtracted raw trace data directly overlaid with the final cumulative Gaussian fit curves to visually demonstrate fitting accuracy and peak shifts. These plots can be effectively stacked vertically.