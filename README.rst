AD7766DevBoard
================
Getting Started
-----------------
- Apply 7.5V to the "7.5V" pin, and gruond to the "GND" pin.

.. csv-table::
    :widths: 25 25 25 25 25
    :file: bom.csv

Debugging
-----------
- Make sure the data ready pin in the Teensy software corresponds to the pin (the layout changed from the Dev Borad)

Issues in Version 1
---------------------
- [FIXED v2] Added drill holes for standoffs 
- [FIXED v2] Selected diode package is not 0805, it's much too large
- [FIXED v2] Hand soldering of filters is very difficult, the out-going pads should be made larger.
- [FIXED v2] The TRIM pin should not be connected to ground, it shouldn't be connected to anything.
- [FIXED v2] a 100nF capacitor at the output of the voltage reference is more than sufficient for noise performance, it adds only 200nV of noise per sample. Remove the 10uF capacitor.
- [FIXED v2] R1 and R2 were switched - they should have the opposite values.
- [FIXED v2] The teensy socket does not actually fit standard header pins - the holes are too small. Need to swap out this part.
- [FIXED v2] Should add pin names for the other output headers.
- [FIXED v2] Removed "U1" and "U2" text
 
Issues in Version 2
---------------------

- [FIXED v3] The MCLK output is 3.3V, and the AD7766 is expecting 2.5V. Added a voltage divider. The datasheet says it's within the absolute maximum ratings, though, so it should work.
- [FIXED v3] The SYNC/PD pin is also 3.3V but the ADC is expecting 2.5V. 

Appendix 1 - Notes
--------------------
- Currently it looks like everything on the PCB is working except the ADC is not sending any data. The DRDY pin is oscillating at 125kHz, the MCLK and SCLK both look as expected, the power supplies are all correct, and yet the ADC is just sending all zeros to the Teensy. Very odd. I will ask the Analog Devices people potentially what's going on with the ADC, and try to fix this.
- They got back to me and asked if I have a 10uF cap between VREF+ and REFGND, which I currently do not.
- I tried adding both a 10uF cap and a 100uF cap, neither of which fixed the issue. However, now it looks like the SPI clock is not doing anything. I thought I checked it before, but perhaps I did not. It's completely silent, and would explain the zero-bytes back. Ah, well duh, of course it's not going off, my Teensy is waiting for the PC to tell it to start sampling, so I wouldn't expect the SPI clock to be going off. I also don't have the cable I need to tell the Teensy to take some data, so that will have to wait until tomorrow.

Debugging Part 2
__________________
- Man, it would be nice to have some test points I can hook the scope probe into for the SPI channel since it's not working. Also, where is the software? I can't find it on this page, and it should be here. It was pretty easy to find in my 'Software' directory.
- God I think OS 11 fucked everything. It looks like comports which I was using to find available ports doesn't work anymore.
- Fixed the issue by updating pyserial using `pip install pyserial --upgrade`. This was in fact a backwards incompatibility introduced by OS X. Now code appears to be running just fine, but still not getting any valid data from the ADC. Checking serial clock now.
- SPI clock looks good, it's applied in bursts of 3 at a period of 125kHz with a clock period of about 10MHz. Maybe check the other voltages during the sampling? See if VDRIVE or VREF falls?
- Interestingly, the supply is using less current than I would expect.
- With the old PCB (AD7766 dev board) connected and the TIA disconnected, looks like the high-frequency PSD is 82nV/rtHz. Looks like two unit tests failed. No signal from the optical chopper was measured. I also didn't measure anything out of the motor, probably because it's not connected :p. There appears to be lots of low-frequency (60Hz) noise with extra harmonics, which is going to be a serious problem if what I want to measure is a response to 60Hz excitation.
- The high-frequency noise PSD is about 82nV/rtHz with the TIA disconnected, 
- The teensy seems reliable as hell. It's been months and the code still works after multiple restarts. Good work.
- There seems like a bunch of either 1/f noise or some 60Hz nonsense. This seemed to go down once the TIA got powered. When powered on, the TIA has a noise of 320nV/rtHz with a 1MOhm resistor. The expected noise is 260nV/rtHz. There is a significant interferer at ~50kHz I think I've seen before.
- Getting a wrong number of bytes error from the Teensy after plugging in the optical chopper. Now it looks like my unit tests aren't even running, and my code for the Teensy is no longer compiling. Getting two fails (one expected failure, one unexpected) and two errors, both unexpected.
- testMeasureSynchronizationData and testMeasureSynchronizationPoints are both failing even though the optical chopper is on. It looks like the Teensy is somehow getting stuck, and now I have to fucking figure out how my code got corrupted. It looks like the SCPIParser library I was using has been removed from the OS, or at least the OS can't find it anymore.
- I checked the chopper voltage and it all looks fine. A 3.2V square wave at 1kHz. Now a lot of my unit tests are failing. I'm no longer getting any data from the teensy after re-uploading code. This is fucking ridiculous, this is the downside of using multi-platform and multi-language non-standard interfaces. I appear to have successfully broken everything. After restarting the Teensy everything is still broken. Fucking hell, what on earth is going on now?
- Apparently the issue is with my Measure() function. The Teensy is not returning any data. I am only receiving back a single byte from the measured data. If it is the '#' character, then it means my Teensy is just not recording anything from the ADC. The ADC is in fact raising DRDY at 125kHz, so it doesn't appear to be an ADC problem. The device is responding to basic commands, such as identifying itself, so it's not a strictly software problem. I suspect the problem is in the Arduino code. I did not realize upgrading my OS would cause so much chaos.
- It appears the Sample() function is never getting executed, which means that an interrupt is never getting triggered or is never being attached properly. The signal IS pulsing at pin 21 of the Teensy, so this tells me the interrupt is never getting attached. Looks like my code is using the wrong data ready pin.
- Found the issue. The pins are different between the new dev board and the old PCB, my newly-written code was using the wrong pins for the old dev board, but the teensy was initially running the old code. Now I'm only getting 2 bytes back from a measurement.
- Now, after the code was fixed I am failing 4 tests and getting errors on an additional three. Looks like all hell has broken loose. The ADC is also measuring an implausibly large amount of noise, which tells me I might have screwed up the byte order that I am sending or reading stuff in. It's railing things out. However, it does look like the raw data is consistent with this, it's also all over the place. After resetting the device there's still a huge number of bytes in the buffer. I think the device is still sending data even though it should not be sending data. Bytes are accumulating in the buffer after the Measure() command is executed. I don't know why. This should also not be happening. Ah, I accidentally added an extra line. Now the PSD is back to what it was before, the ADC is still working. Now some more of my tests are failing. The PSD is 320nV/rtHz.
- I figured out at least part of the synchronization issue - was using the wrong pin. However, now my unit tests are throwing errors that they weren't before. testMeasureLargeByteCount isn't working. I suspect this has something to do with the failure of the synchronization tests. 
- For some reason on a long measurement, the device stops sending back data after 648533 measurements are taken. This seems suspiciously close to the max value for a 16-bit integer. However, it's declared in the Arduino as a 32-bit signed integer. After it fails to give back the rest of the data, it subsequently fails to communicate at all - it cannot identify itself or do anything. Performing a reset doesn't appear to do anything, which means the device must be stuck somewhere. DataPointsToSample is also an integer. Our device isn't even listening for our commands any more, and it's not sending us any data either. This is very odd. Code also isn't able to upload to the device, so it's being rendered unresponsive for some undetermined reason.
- The device does understand that it needs to read 1 million samples, so it's not an issue of the variable not being set appropriately.
- The device is sending back 1,943,551 bytes of data befor it fails and is subsequently unable to communicate. The second time I tried this it sent back 1,943,551 bytes of data. The failure is consistent and not stoichaistic. For some reason after it sends that number of bytes it just gives up.
- On the arduino sketch it fails at 648157. Second time 648157. It actually prints the next 3 characters "648" on the next line, and says "Done!" after 648157. This behavior is damn consistent. It is able to execute 500,000 measurements multiple times without any issue. I suspect the origin of this error is pretty damn obscure and hard to track down. For now, I'm just going to limit us to 500K measurements. If we want more than 500K, I'm going to need to implement a software fix to query for multiple blocks of 500K. All tests now passing.

New PCB
_________
- The new PCB has the same exact issues as the old one. This tells me there's something wrong with my connections or something. Time to debug the damn thing.
- Just as before, all the voltages seem correct and SCK is pulsing appropriately. The SPI clock frequency is 3MHz.
- Here is a potential issue - the MCLK signal needs to be voltage-divided before it gets to the AD7766, but it appears that it is not being voltage divided. It's not clear how it's getting to the ADC at all.
- I tried applying a 2.5V signal to the ADC MCLK pin. This did not appear to help anything. It's possible that the 3.3V MCLK signal fried some of the sampling circuitry while leaving the SPI circuitry and the DRDY circuitry intact. Implausible, but possible. In this case, I will either need to add the voltage divider at the output (requiring a new PCB revision) or add an on-board frequency synthesizer, like a crystal. I can try populating yet another PCB, but add a resistive divider at the output before I test it. A manual way to reset the ADC also might not be a bad idea. 
- I think The SYNC/PD pin is also 3.3V when it should be 2.5V and needs to be divided down to 2.5V. This is another potential problem. The AD7766 dev board applies a 2.5V signal independent of VDrive.
- Perhaps the teensy is driving the MISO pin? This is not likely, as it's not driving it on the ADC dev board. On the dev board, the MOSI pin isn't connected at all.
- Maybe there are safety features to prevent frying the ADC on the dev board? I should check.
- I have no fucking idea how to increase the sourceMeter compliance from 105uA. I did it before, but I can't seem to be able to do it again.
- The noise from the single-ended board is actually not much better with the keithley than the Tekpower source. I NEED to get my new PCB working so that everything can be on the same PCB and I don't have to deal with the pickup of these wires.

TO TEST:
_________
- Make sure the MISO pin isn't being unintentionally driven by the ADC. Try cutting the trace on the PCB or setting the pinMode to input.
