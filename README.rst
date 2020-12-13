AD7766DevBoard
================

.. csv-table::
    :widths: 25 25 25 25 25
    :file: bom.csv

Issues in Version 1
---------------------
- [FIXED v2] Forgot to put standoffs on the thing
- [FIXED v2] Selected diode package is not 0805, it's much too large
- [FIXED v2] Hand soldering of filters is very difficult, the out-going pads should be made larger.
- [FIXED v2] The TRIM pin should not be connected to ground, it shouldn't be connected to anything.
- [FIXED v2] a 100nF capacitor at the output of the voltage reference is more than sufficient for noise performance, it adds only 200nV of noise per sample. Remove the 10uF capacitor.
- [FIXED v2] R1 and R2 were switched - they should have the opposite values.
- [FIXED v2] The teensy socket does not actually fit standard header pins - the holes are too small. Need to swap out this part.
- [FIXED v2] Should add pin names for the other output headers.
- [FIXED v2] Remove "U1" and "U2" text
 
Issues in Version 2
---------------------
- [FIXED v3] The MCLK output is 3.3V, and the AD7766 is expecting 2.5V. Added a voltage divider. The datasheet says it's within the absolute maximum ratings, though, so it should work.
