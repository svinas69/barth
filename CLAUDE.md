# Barth Project

## Controller
- Hardware: Barth STG-650 PLC controller
- Programming: vendor software (not plain-text source) — exported/config files go here as they're produced

## I/O

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

## Control Logic

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
   - Wait for pressure switch to change state to active again (i.e. next rising edge)
   - Wait 2s
   - → Step A (loop)

**Exit AUTO_CYCLE** (any of these → valve1 = OFF, valve2 = OFF, return to MANUAL):
- ff held alone (without fv) — abort
- fv held alone (without ff) — abort
- ff AND fv held together ≥2s — normal stop

## Status
- I/O mapping and control logic defined; not yet implemented in Barth vendor software

## Conventions
- TBD as hardware and control logic details are established
