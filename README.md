# Sportbike-CDI-development-
Sportbike CDI development with programable advance spark curve

Digital electronic ignition units are commonly in use in cars and motorcycles. The most important feature of a digital igniter is the processor accuracy on timing calculations, the programmable spark advance curve and the over RPMs cutoff for engine protection.

There are two main types of electronic igniter's on the market: TCI (transistor conmuted ignition) and CDI (capacitive discharge ignition). In TCI ignition a low voltage is conmuted (typically the 12V battery voltage) and applied to the spark coil at ignition time. In the case of CDI igniters a capacitor is charged at high voltage using a DC-DC converter stage (typically 200 to 400 volts) and discharged at ignition time. CDI igniters are much better in performance because coil high output voltage remains at high RPMs but one disadvantage is the need of an additional DC-DC inverter stage. 

I have build the CDI igniter unit in assembler language for my old Kawasaki (GPX600) sportbike because the factory igniter unit started to fail. In the picture above the external black transformer forms part of the low frequency DC-DC inverter stage that increase the voltage from 12 volts to approximately 300 volts. For testing purposes and initial development phase it worked as expected using a transformer but the selected power rate is to small to sustain the spark energy at high RPMs. To reach a better spark and improve the efficiency the right choice on next design would be to implement a high frequency switching DC-DC converter stage. Anyway the CDI project worked well on my Kawasaki GPX600 bike !!!

The entire project was developed in assembler language for 16F84 microcontrollers. The advance timing table is user programmable and follows GPX600 manufacturer specifications: 10º before TDC at 1000 to 40º before TDC at 4000 rpm. Over RPMs spark cut-off is set at 12500 rpm. For the RPM instruments panel a tachometer output is available.

Before running the CDI unit on the bike I tested the circuit board externally measuring the advance timings with an oscilloscope to avoid damages in case of failures.  A common CDI failure is the triggering of the spark out of ignition time and such events can lead to mechanical failures. Testing phase is very important and more on the development of a CDI unit.

Below is the assembler code used on the project. Please be aware that I don't have maintained this program after Nov 2004, so in case of any damages on the engine is your own responsibility. The advance timing table is calculated for a 8Mhz oscillator following original Kawasaki GPX600 values (10° at 1000 RPMs to 40° at 4000).
