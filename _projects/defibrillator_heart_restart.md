---
layout: page
title: "Heart Restart: a dual-shock defibrillator"
description: A microcontroller-controlled AED (ECE 395) that measures thoracic bio-impedance before discharge and is built for two sequential biphasic shocks, with strict galvanic isolation between the high-voltage side and the STM32. I designed the five-rail power subsystem, the 4-layer mixed-signal PCB, and the isolation.
img: assets/img/defib_pcb_render.png
importance: 5
category: systems
related_publications: false
---

ECE 395 is the advanced digital projects lab, where the assignment is a full system rather than a single circuit, and over Spring 2026 three of us (Hussein Thahab, Brian Chiang, and me) built an automated external defibrillator from discrete parts. Most low-cost public-access AEDs deliver one fixed shock and ignore the patient in front of them. Ours measures thoracic bio-impedance with a MAX30009 before discharging, and the shock path was designed around a dual H-bridge so it could deliver two sequential biphasic shocks at independently set energy, all for a bill of materials under a thousand dollars. My share was the power distribution network, the 4-layer mixed-signal PCB layout, and the galvanic isolation between the shock side and the control logic, plus helping on the high-voltage subsystem.

<div class="row justify-content-sm-center">
    <div class="col-sm-11 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/defib_block_diagram.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    System block diagram. The power subsystem on the left feeds five rails to everything else, the STM32 sits in the middle, and the shocking subsystem on the right (flyback, transformer, capacitor, H-bridge, twice over) is kept electrically separate from the low-voltage side.
</div>

## The power subsystem

The board runs off 18 V, from two 9 V batteries in series, and the power subsystem turns that into five regulated rails: +5 V, +3.3 V, +1.8 V, +15 V, and -15 V. I split it into a digital path and an analog path on purpose, so switching activity on the logic supplies would not couple into the sensitive front-end. The digital side steps 18 V down to 5 V through a TPS5420D buck converter, then to 3.3 V and 1.8 V through LDOs, feeding the STM32, the MAX30009, and the display. The analog side uses an LM2574N-15 for +15 V and an LT1054 charge pump to invert that to -15 V, which is what the ECG instrumentation amplifiers and the op-amp filter stages need to swing both ways around ground.

I built each rail in LTspice and checked it for stability before committing anything to copper, since respinning a 4-layer board is slow and not free. On the bench the power section behaved close to the simulation. Its one real problem was switching noise riding on the rails, which I cleaned up with ferrite beads at the regulator outputs. That was the cheap fix the simulation did not warn me about, because the model did not carry the parasitics that were actually generating the noise.

## Isolation and the board

The safety constraint drives the whole layout. The shock pads can sit at hundreds of volts, and the STM32 and analog front-end cannot see any of that, so the two domains have to be galvanically separated and stay that way even under fault. The gate-drive signals cross into the high-voltage switching stage through UCC21520 isolated gate drivers, the voltage monitoring comes back through AMC1200 isolation amplifiers, and the high-voltage side gets its own power through isolated DC-DC converters. On the 4-layer KiCad layout I held more than 200 V of creepage and clearance around the high-voltage nets so nothing arcs across to the analog section, and routed the high-current power traces wide enough to carry the load without cooking.

<div class="row justify-content-sm-center">
    <div class="col-sm-11 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/defib_pcb_render.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    The dual-shock board. The two channels mirror each other, each with its own flyback charging stage, shock capacitor, gate drivers, and H-bridge, with the isolation barrier running between the high-voltage sections and the control side.
</div>

## What did not work on the bench

The honest summary is that the headline feature did not land in this revision. The H-bridges charged and discharged into a resistive load correctly, so the energy-delivery hardware works, but the PWM control that was supposed to shape the biphasic waveform stayed unresolved. Reversing the current mid-discharge under PWM, with energy set by duty cycle, is the genuinely hard part of the design, and we ran out of bench time before the variable-energy biphasic delivery worked end to end. It charges and it dumps, it does not yet shape.

The ECG front-end has a more annoying story. The signal chain (two instrumentation-amplifier stages for a total gain of 10,000, a 60 Hz notch that hit about -40 dB in simulation, a 15 Hz low-pass, and a full-wave rectifier) simulated correctly and picked up cardiac activity in LTspice. We could not validate it on hardware, because an assembly error on the board reversed the polarity of the supply pins on two of the op-amps, which kept that part of the analog section from ever powering up correctly. That one sits in my area, since it is a layout and assembly mistake, not a design flaw in the filter itself.

The bio-impedance channel met the IEEE limits on patient current but missed its accuracy target. It came in around 17% error against the actual impedance, where the goal was 5%. The cause is that the MAX30009's internal calibration routine is not implemented yet, so the raw measurement is running without the correction it is supposed to have. The power board also had a few footprint mismatches, on the catch diode, the electrolytics, and the power inductors, that we worked around by hand at assembly rather than respinning.

## What I would change

The first revision would start with a footprint audit of the power board against the actual parts, because hand-fitting components at assembly is how you lose an afternoon and introduce mistakes like the reversed op-amp supplies. After that, the order is clear enough: fix the supply polarity so the ECG chain can finally be checked on real hardware, get the MAX30009 calibration running to pull the impedance error down toward 5%, and spend the bulk of the effort on the PWM control so the biphasic waveform and variable energy actually work, since that is the part that makes this different from a fixed-shock unit.

<hr>

[**View the full writeup (PDF)**]({{ '/assets/pdf/defibrillator_heart_restart.pdf' | relative_url }}), with the full schematics, the PCB layout, the rail-by-rail power design, and the bench results.
