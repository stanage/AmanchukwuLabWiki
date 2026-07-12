=================================
Load Cell Operation & Calibration
=================================

This page covers the initial wiring, parameter configuration, and calibration sequence for the laboratory load cell equipped with an LED digital indicator. 

.. warning::
   **Equipment Rating Notice:**
   Our laboratory load cell is strictly rated up to a maximum capacity of 100 kg. Do not exceed this weight limit.

1. Wiring & Power Setup
=======================

1. Wire the load cell into connections 9, 10, 11, and 12 on the indicator unit.
2. Wire the primary power cord into connections 1 and 3.
3. Plug the unit into an approved laboratory power source.

2. Parameter Menu Overview
==========================

To enter the parameter configuration interface, hold down the **SET** button until the parameter menu opens (the display will first show ``AL1``). While you can ignore many of the factory defaults, the following parameters are critical for accurate operation:

* **USP:** The specific mass of the known physical object you intend to use for calibration.
* **dP:** The physical position of the decimal point displayed on the PV screen.
* **Unt:** The measurement unit indicator.
  
  * ``g`` = Grams
  * ``Backwards y g`` = Kilograms (kg) — *Set to kg as our default standard.*
  * ``gamma`` = Tons

How to Edit Parameters
---------------------

* **Enter Editing Mode:** Press the **<</M** button to enter editing mode for a highlighted parameter.
* **Save & Advance:** Press **SET** to save changes and advance to the next parameter in the menu sequence.
* **Modifying Values (USP / dP):** Use the **<</M** key to shift leftwards between digits. Press the **^** (Up) or **v** (Down) keys to change the active flashing number. 
  
  .. note::
     **Tip:** Shifting all the way to the far-left digit and reducing the value to zero unlocks additional right-side decimal resolution.

* **Modifying Units (Unt):** Press the **^** or **v** keys to toggle between units.
* **Exit Menu:** Hold down the **SET** button to return to the main operation display.

3. Step-by-Step Calibration Sequence
====================================

Follow this sequence exactly to calibrate the system:

.. Sequence
   This is a strict step-by-step procedure; missing or reordering steps will result in an incorrect calibration baseline.

1. **Zero the Scale:** Ensure the load cell is completely clear of all loads. Hold down the **v** key until the PV screen reads "Oy", then release it. The PV screen should now stabilize at "0".
2. **Apply Calibration Weight:** Carefully place your known calibration weight onto the load cell. The true mass of this object must exactly match the value previously configured in your **USP** parameter.
3. **Configure the SV Screen:**
   
   a. Hold down the **^** key until the SV LED screen begins to flash, then release it.
   b. Edit the value on the flashing SV screen using the **<</M**, **^**, and **v** keys until it explicitly reflects the true mass of the object sitting on the load cell (this must match your USP setting).
   c. Press the **SET** key to finalize, store the calibration path, and exit.

4. Verification & Routine Testing
=================================

* **Pre-Use Verification:** A standard 1 kg calibration weight is stored directly with the equipment kits. This weight must be used to test the load cell's accuracy before each experimental run. If the display fails to register exactly 1 kg, repeat the calibration sequence detailed in Section 3.
* **Historical Baseline:** Initial calibration validation was verified using a sand-and-metal filled beaker (mass: 0.57 kg) and cross-checked against standard weights of 0.1 kg, 0.4 kg, and human scale footprints—confirming robust linearity across the operational spectrum.

..

.. note::
   For a more comprehensive SOP, please consult the full `Load Cell Operation SOP on Box <https://uchicago.box.com/s/7c36lpzxupwu0mgonnkdt6xbxlkbhxwd>`_.