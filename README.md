# Audi A5 8T facelift: US → EU tail lights

Notes on fitting European LED tail lights to a US-market Audi A5 (8T facelift, B8.5): wiring, the module 46 channel map, and why adaptation coding alone did not hold on this car.

![Brake lights after the conversion](img/03-brake-after.jpg)

## Status: the coding does not survive a restart on this module

**Do not copy the values below onto a facelift car and call it done.** They made every lamp work correctly in the garage, but the module rejected them at its next restart.

What happened on this car (module 46, `8K0 907 064 KM`, BCM2 2.0, H15 0711):

- With the eight channels changed, the lights worked as intended until the module restarted (ignition off, bus asleep, ignition on).
- After the restart the module logged `03389 - Data Set Implausible`, lit the bulb warning in the cluster and fell back to a built-in failsafe light set. In that state brake and turn share the same LEDs again, as on US coding.
- The fault cannot be cleared while the changed values are stored.
- Five different sets were tried. Each one failed after a restart, including a set with a single channel changed to a value that the neighbouring channel already uses (197: `6 → 134`). The untouched factory set passes.
- Restoring every channel to its factory value and restarting the module removes the fault.

Conclusion: this module version checks the whole light data set against something like a checksum. Editing the light channels through adaptation (VCDS, OBDeleven and similar tools) is not enough. The data set has to be written as a whole by a tool that also writes a valid signature: ODIS parametrisation, VCP with the matching data set, or a coding dongle sold with the adapter looms. Older BCM2 versions (pre-facelift cars) appear not to have this check, which is why many forum guides work there.

The channel map and the function codes below are still correct and useful for understanding the module. The car is currently back on factory values.

### The values that were tested

| Channel (left / right) | US value (factory) | EU value tested |
|---|---|---|
| 195 / 196 | 134 / 166 | 132 / 164 (also 4 / 36) |
| 197 / 198 | 6 / 38 | 4 / 36 |
| 199 / 200 | 6 / 38 | 12 / 44 |
| 201 / 202 | 134 / 166 | 140 / 172 (also 12 / 44) |

## Contents

- [The car and the parts](#the-car-and-the-parts)
- [The symptom](#the-symptom)
- [Wiring](#wiring)
- [Coding](#coding)
- [Worked example: one channel, start to finish](#worked-example-one-channel-start-to-finish)
- [Result](#result) (as seen before the restart)
- [How the values were found](#how-the-values-were-found)
- [What went wrong](#what-went-wrong)
- [Sources](#sources)

## The car and the parts

| | |
|---|---|
| Car | 2015 Audi A5 Coupé 2.0 TFSI quattro, US market, factory LED tail lights |
| Module | 46 Central Convenience (J393), `8K0 907 064 KM`, BCM2 2.0, H15 0711 |
| EU outer lamps (fender) | `8T0 945 095 H` left, `8T0 945 096 H` right, marked ECE |
| EU inner lamps (trunk lid) | `8T0 945 093 C` left, `8T0 945 094 C` right |
| US lamps removed | `...095 J / 096 J` outer, `...093 D / 094 D` inner, marked SAE |

The suffix tells the market: `H` and `C` are European, `J` and `D` are North American. The housing also carries an `ECE` or `SAE` mark. The EU lamps here came from a breaker, with the connector pigtails cut from the donor car.

## The symptom

US lamps use one red element for both brake and turn. EU lamps have a separate amber turn strip. With EU lamps on US coding:

- pressing the brake also lit the amber turn strips,
- the turn signal and the hazards also flashed part of the red brake LEDs.

The photo below shows it. The left lamp is already recoded, the right lamp is still on US values and lights its amber strip (top row) on brake:

![Brake: left recoded, right still on US values](img/02-brake-left-fixed-right-still-us.jpg)

## Wiring

No new wires were run on this car.

- The car-side connector on the outer lamp is grey, `4H0 973 713 B`. The EU lamp takes the green one, `4H0 973 713 C`. Same connector family, different keying.
- The wire colours on the donor pigtails matched the wire colours on the car harness, on the outer and the inner lamps. The pigtails were spliced on colour to colour.
- The inner lamp has 4 wires on both the car and the EU pigtail: ground, tail, turn, rear fog.

| EU connector (green, `...713 C`) | US connector (grey, `...713 B`) |
|---|---|
| ![EU connector](img/06-connector-eu-green.jpg) | ![US connector](img/07-connector-us-grey.jpg) |

Tips:

- Cut the car-side connector close to the plug and leave a few centimetres of wire on it, so the original can be soldered back.
- Before cutting, probe each car-side wire with a multimeter (tail on, brake pressed, turn signal on) and note colour → function. If the colours on your pigtail do not match, connect by function instead.
- Solder and heat-shrink the joints.

## Coding

Any VCDS-style tool that reaches module 46 adaptation works. Route: module 46 → Security Access `16017` → Adaptation. The tool's drop-down list does not show these channels, so type the channel number by hand.

Odd channels are the left side, even channels are the right side. The right-side value is the left-side value plus 32.

| Channel (L / R) | What it drives on the EU lamps | US (before) | EU (after) |
|---|---|---|---|
| 195 / 196 | outer lamp: small block of brake LEDs | 134 / 166 | **132 / 164** |
| 197 / 198 | outer lamp: main rows of brake LEDs | 6 / 38 | **4 / 36** |
| 199 / 200 | outer lamp: amber turn strip (top) | 6 / 38 | **12 / 44** |
| 201 / 202 | inner lamp: amber turn strip | 134 / 166 | **140 / 172** |
| 203 / 204 | outer lamp: tail strip | 30721 / 30753 | unchanged |
| 207 / 208 | inner lamp: tail strip | 51201 / 51233 | unchanged |
| 205 / 206 | not identified | 13 / 45 | unchanged |
| 209 / 210 | not identified | 73 / 105 | unchanged |

How to read a value:

| Part of the value | Meaning |
|---|---|
| low bits `4` | brake only |
| low bits `6` | brake + turn on one element (US) |
| low bits `12` | turn only |
| `+32` | right side |
| `+128` | extra flag on channels 195 and 201, kept as it was |

So every change is the same idea: `6 → 4` where the wire feeds brake LEDs, `6 → 12` where it feeds an amber strip.

Example: channel 202 is the right inner amber strip. `12` (turn only) `+ 32` (right) `+ 128` (flag) `= 172`.

## Worked example: one channel, start to finish

This is how channel 199 (left outer amber strip) was done. The same steps apply to every channel.

1. Engine running or a charger connected. Tail lights on.
2. Module 46 → Security Access → `16017`.
3. Adaptation → type `199` → Read. The stored value shows `6`. Write it down.
4. Type `12` as the new value and press **Test**. Nothing is saved yet.
5. Press the brake. The left amber strip stays dark, the right one (still on `38`) lights. That is the proof the value is right.
6. Press **Save**, confirm, then read channel 199 again. It must show `12`.
7. Repeat on the right side: channel 200, `38 → 44`.
8. Switch the hazards on and check that only the amber strips flash.

If a test value does something wrong, read the channel again. The test value is dropped and the stored value comes back.

## Result

These photos show the lights with the tested values, before the module restarted and rejected them.

| Function | Outer lamp (fender) | Inner lamp (trunk lid) |
|---|---|---|
| Tail | red strip | red strip |
| Brake | rows of red LEDs | none |
| Turn / hazard | amber strip | amber strip |
| Rear fog | none | 3 bright LEDs |
| Reverse | white, bottom | none |

Tail lights:

![Tail lights](img/01-tail-lights.jpg)

Brake after recoding both sides. Brake rows lit, amber strips dark:

![Brake after](img/03-brake-after.jpg)

Hazards after recoding, a burst of ten frames. Only the amber strips flash, on the outer and the inner lamps. The brake rows stay dark:

![Hazards after](img/04-hazard-burst-after.jpg)

Rear fog off (top) and on (bottom). It worked with no coding and no extra wire on this car. The light switch already had the rear fog position and the cluster shows the indicator:

![Rear fog off and on](img/05-rear-fog-off-vs-on.jpg)

Note on the photos: a phone camera blows bright red LEDs out to orange and yellow. In real life the brake rows and the tail strips are red, and only the top strips are amber.

**The inner lamps do not light on brake.** The inner lamp has four wires (ground, tail, turn, fog), so there is no wire for a brake element. Testing channel 207 with `51205`, the value a facelift Q5 uses for a tail strip that brightens on brake, made no visible difference. Values from `51209` upward switched the tail strip off. Brake light comes from the two outer lamps and the third brake light.

## How the values were found

Forum threads for this conversion give values for other cars or other lamp generations, and they disagree with each other. The channel map above was built on the car.

1. **Backup first.** Channels 179–247 were read one by one and logged to a file before any change.
2. **Test, not Save.** Every candidate value was tried with the Test button first.
3. **Switch one channel off and see what goes dark.** Testing a channel with `0` while the tail lights, brake or hazards are on shows which LED group it feeds.
4. **Compare left against right.** Only the left channel was changed while the right stayed on US values. Both lamps are in the same photo, so the untouched side is the reference.
5. **A phone as the eyes.** The owner was alone, and nobody can press the brake and look at the lamps at the same time. A phone on the floor behind the car sent its camera picture to a laptop. An AI coding assistant (Claude Code) drove the diagnostic tool on a second laptop by remote control: it set a test value, asked by voice for "brake" or "hazards", took the photos and compared them. The owner confirmed colours by eye where the camera could not.
6. **Save, then read back.** Each accepted value was saved alone and the channel was read again to confirm it.

## What went wrong

- **The camera lies about LED colour.** In video frames the red brake LEDs and the amber strip looked the same, and two wrong conclusions came from that. Full-resolution stills cropped to each lamp fixed it: the LED groups sit in different places, so position tells them apart even when colour does not.
- **Low battery voltage breaks the session.** With the ignition and lights on for a long time the battery fell to 10.8 V. The diagnostic adapter dropped off USB and the module stopped answering. Run the engine or connect a charger.
- **The module overwrote about twenty channels by itself.** After the first restart with changed values, channels 195–212 and the service channels 213, 214 and 230–245 held different numbers (for example 199–202 = `10248 / 10280`, 212 = `15`, 213 = `12336`, 230–237 = `2771, 2772, 2901, 2902, 3532, 3533, 3554, 3555`). This was first blamed on a scripted write. It is the module's failsafe set: a [Ross-Tech thread](https://forums.ross-tech.com/index.php?threads/9240/) shows the same numbers on another car. All of them must be restored from the backup, not only the light channels, or `03389` stays.
- **Scripted clicking in the diagnostic tool is risky.** A click sequence that assumed the wrong screen opened a different module. Check the screen after every step.
- **The first read after opening the adaptation screen can show a stale value.** Read the channel a second time before trusting it.
- **Brake lights need the ignition on.** With the ignition off they stay dark, which looks exactly like a coding fault.

## Sources

- [AudiWorld: post-facelift European spec tail lights coding for Q5](https://www.audiworld.com/forums/q5-sq5-mki-8r-discussion-129/post-facelift-european-spec-tail-lights-coding-q5-part-numbers-2883472/) (same module family, different channel layout)
- [motor-talk: US-Blinker auf EU-Blinker umcodieren](https://www.motor-talk.de/forum/us-blinker-auf-eu-blinker-umcodieren-apk-181-255-t3742740.html) (login 16017, channels 191–212)
- [a5oc: looking for standard VCDS codes for tail lights A5 2015](https://www.a5oc.com/threads/looking-for-standart-vcds-codes-for-tail-lights-a5-2015.147881/)
- [Audizine: USDM tail lamps conversion](https://audizine.com/threads/usdm-tail-lamps-conversion.937109/)

## Disclaimer

This is a record of one conversion, shared as is. Wrong adaptation values can switch lights off or leave them on. Check every light before you drive, and check what your local inspection requires.
