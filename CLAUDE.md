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

## Status
- I/O mapping defined; control logic not yet written

## Conventions
- TBD as hardware and control logic details are established
