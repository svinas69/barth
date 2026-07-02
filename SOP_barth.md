# SOP — Barth Project

**Date:** 2026-07-02
**Project:** barth (Barth STG-650 PLC controller)

## Hardware
- Controller: Barth STG-650 PLC
- Programming: vendor software (not plain-text source); no exported program file yet

## I/O Configuration

### Inputs
| Terminal | Label | Meaning |
|---|---|---|
| IN1 | ff | Forward push button |
| IN2 | fv | Backward push button |
| IN3 | pressure | Pressure sensor/switch |

### Outputs
| Terminal | Label | Meaning |
|---|---|---|
| OUT3 | valve1 | Valve 1 |
| OUT4 | valve2 | Valve 2 |

## Control Logic (current spec)

### Mode: MANUAL (default / idle)
- valve1 = ff (momentary — on only while ff is held)
- valve2 = fv (momentary — on only while fv is held)
- If ff AND fv are both held continuously for ≥2s → enter AUTO_CYCLE

### Mode: AUTO_CYCLE
Entered from MANUAL when ff+fv held together ≥2s. Alternates valve1/valve2, gated by the pressure switch, until stopped.

1. **Step A**: valve1 = ON, valve2 = OFF
   - Wait for pressure switch to go active
   - Wait 2s
   - → Step B
2. **Step B**: valve1 = OFF, valve2 = ON
   - Wait for pressure switch to change state to active again (next rising edge)
   - Wait 2s
   - → Step A (loop)

**Exit AUTO_CYCLE** (any of these → valve1 = OFF, valve2 = OFF, return to MANUAL):
- ff held alone (without fv) — abort
- fv held alone (without ff) — abort
- ff AND fv held together ≥2s — normal stop

## What was built/changed this session
- Created the `barth` project folder and git repo
- Defined I/O mapping (3 inputs, 2 outputs)
- Specified manual and auto-cycle control logic through Q&A, including exit/abort conditions and valve state on exit

## Known issues / next steps
- No program has been written in the Barth STG-650 vendor software yet — this is a spec only
- Power-up behavior and emergency-stop input not yet defined
- Need to confirm actual Barth STG-650 programming tool/file format so exported programs can be version-controlled here

## How to program
- No compile/upload toolchain — program is authored directly in Barth vendor software per the I/O and logic spec above
