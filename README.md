# ERC-2026-Power-Mux-Board-Xplore
This is the first version of this power mux board uses LTC4421 as a "brain".
It takes measurements, switched between power sources accordingly to their priority of quality.


The LTC4421 measures each channel’s current from the voltage across its 0.4 mΩ shunt (ΔV_SNS = I · R_SENSE), sensed through Kelvin connections. It uses this voltage for three things:

Current limiting. When ΔV_SNS reaches 25 mV (20–30 mV over tolerance), the controller reduces the MOSFET gate voltage so the current stays at I_LIM = 25 mV / 0.4 mΩ ≈ 62.5 A (50–75 A).
Overcurrent fault timing. While the channel is in current limit, the TMR capacitor charges. If the overcurrent lasts longer than t_TMR, the channel is turned off and flags FAULT.
Fast short-circuit response. If ΔV_SNS exceeds 50 mV (≈125 A), the gate is pulled down with 50 mA so the current is brought back under control within microseconds.

The TVS diodes protect the board from fast voltage transients. D7 and D8 sit on the inputs (cable and battery), D9 on the output.

In normal operation they do not conduct: below V_RWM = 33 V the leakage is only µA, and the maximum battery voltage is 29.4 V.
During a spike they break down (V_BR ≥ 36.7 V) and clamp the voltage, sending the surge current to ground. Typical spikes are:
inductive kicks when a cable is hot-plugged (cable inductance ringing with the ceramic capacitors);
motor regeneration and inductive kicks;
ESD.
Coordination with the LTC4421: V_BR,min (36.7 V) is above the LTC4421 OV threshold (34.9 V). Sustained overvoltage is therefore handled by the controller, which disconnects the source. The TVS only absorbs short transients (rated 1500 W for a 10/1000 µs pulse).
Reverse polarity: a unidirectional TVS conducts in the forward direction and acts as a crowbar, so the fuse opens.

Fuses (F1, F2 – MAXI 40 A)

The fuses protect against board failures, not against output short circuits.

Output shorts are handled by the LTC4421. It limits the current to ≤ 75 A for ≤ 1.25 ms: I²t ≈ 75² × 1.25 ms ≈ 7 A²s, which is negligible for a 40 A fuse.
The fuse opens when the controller cannot limit the current:
a shorted MOSFET;
a shorted TVS or input capacitor;
reverse polarity (TVS crowbar);
a wiring short before the controller.
In these cases the battery can deliver hundreds of amps, and the fuse clears in a fraction of a second.
Normal load: the 40 A / 2 s peak is 1× the rated current, so the fuse does not blow. MAXI fuses are rated 32 V DC, above the 29.4 V maximum.
Timers: TMR and QUAL

TMR – overcurrent fault timer (C10/C11 = 10 nF)

While a channel is in current limit, the TMR capacitor charges at 6 µA.
After t_TMR = 83 µs/nF × 10 nF ≈ 0.83 ms (0.55–1.25 ms), the channel turns off and FAULT is asserted.
A cool-down of 1024 × t_TMR (≈ 0.85 s) follows. Because FAULT is tied to DISABLE, the channel then retries automatically.
Lower bound: t_TMR must be longer than the time needed to charge the output capacitance at start-up:
t_TMR,min > 1.5 × C_OUT · V_IN / (I_LIM − I_LOAD)
Upper bound: the MOSFET SOA, since the MOSFET dissipates about V_IN × I_LIM for the whole t_TMR.

QUAL – input qualification timer (C9 = 3.3 nF)

A source must stay inside its valid UV/OV window continuously for t_VALID = 16 ms/nF × 3.3 nF ≈ 53 ms before it can be connected.
This rejects contact bounce while the XT60 connector is being inserted, as well as noise and short glitches.
Connecting is deliberately slow (53 ms). Disconnecting is fast (t_INVALID ≤ 8 µs).


Board logic
Two sources:
Channel 1 = 28 V cable PSU (priority);
Channel 2 = 7S Li-ion battery (21–29.4 V).
Each path goes through a fuse, TVS, input capacitor, a back-to-back N-MOSFET pair with a sense resistor, and then to the common output.

Input monitoring: a resistor divider per input sets three thresholds.
Source	UV rising	UV falling	OV
Cable	24.85 V		23.87 V		34.9 V
Battery	24.05 V		21.0 V		35.0 V

Priority: whenever the cable is valid, the output is fed from the cable and the battery is disconnected. Otherwise the battery feeds the output.

Switchover:
When the cable is unplugged or sags below its falling threshold, the cable channel turns off within ≤ 8 µs.
After a break-before-make time of ≤ 15 µs, the battery channel turns on.
The 220 µF output capacitor C6 holds the bus meanwhile, with a droop of about 2 V at 20 A.
When the cable returns and qualifies for 53 ms, the board switches back.
Back-to-back MOSFETs block current in both directions. The battery can never back-feed the PSU, and the two sources are never connected together.

Protection:
current limit with fault timer and auto-retry;
UV/OV lockout;
fast gate pull-down above 50 mV;
current-limit foldback (halved) when the output is hard-shorted (< 0.41 V);
TVS for transients;
fuses for catastrophic failures.
User interface:
LEDs show which inputs are valid and which channel is active;
the SHDN connector (J4) turns both channels off.

Start-up:
the charge-pump reservoir capacitor C5 (2.2 µF) charges first (~0.5 s);
the inputs are then qualified;
the highest-priority valid source is connected.
