# miCon-L Function Block Wiring Guide — VALVE Logic

Reference for building the control logic from `CLAUDE.md` inside the miCon-L
graphical editor (`$STG-650_TASK` worksheet).

Block names/pins below are confirmed from Barth's own reference
(newsupport.barth-elektronik.com/810986-micon-l → Function Blocks):

| Block | Pins | Notes |
|---|---|---|
| `AND` | P0..P7 (2-8 inputs), Q | any input/output can be negated in the parameter dialog |
| `OR` | P0..P7 (2-8 inputs), Q | any input/output can be negated in the parameter dialog |
| `NOT` | P, Q | only needed when a negated signal must be reused elsewhere; prefer negating a pin on AND/OR directly |
| `On-Delay` | P, Tv (sec, param), Q | delays rising edge of P by Tv; P must stay 1 for the **whole** Tv or Q never fires |
| `Off-Delay` | P, Tv (sec, param), Q | delays falling edge of P by Tv |
| `Rising Edge` | P, Q | 1-scan pulse on P's 0→1 transition |
| `Falling Edge` | P, Q | 1-scan pulse on P's 1→0 transition |
| `Reset Dominant (FFR)` | S, R, Q | if S=R=1, Q→0 (Reset wins) |
| `Set Dominant (FFS)` | S, R, Q | if S=R=1, Q→1 (Set wins) |

**There is no toggle/impulse-relay/T-flip-flop block and no pulse (monoflop) timer** in miCon-L. Both are built below from the primitives above — this is the actual implementation, not a fallback.

## I/O tags
- `ff` = IN1, `fv` = IN2, `pressure` = IN3 (boolean)
- `valve1` = OUT3, `valve2` = OUT4 (boolean)

## Toggle-from-pulse pattern (used twice below)
To get a toggle output `Q` from a one-scan trigger pulse `Trig`, with a priority `Reset`:
- `S = AND(Trig, Q negated)` — feed the FFR's own `Q` back into this AND with that pin negated
- `R = OR(AND(Trig, Q), Reset)` — feed `Q` back into this AND un-negated
- `FFR: S=S, R=R → Q`

Reset-dominant (FFR) is used because `Reset` must always win over a coincident toggle pulse.

## Stage 1 — Start/stop detection (ff+fv held ≥2s)
| Block | Inputs | Output |
|---|---|---|
| AND1 | P0=ff, P1=fv | BothHeld |
| ONDELAY1 (On-Delay, Tv=2) | P=BothHeld | Held2s |
| REDGE1 (Rising Edge) | P=Held2s | StartStopPulse |

`BothHeld` is a real level signal (buttons physically held), so On-Delay works directly here — no latch needed.

## Stage 2 — Abort detection (either button held alone)
| Block | Inputs | Output |
|---|---|---|
| AND2 | P0=ff, P1=fv (negated) | FfAlone |
| AND3 | P0=fv, P1=ff (negated) | FvAlone |
| OR1 | P0=FfAlone, P1=FvAlone | AbortCondition |

## Stage 3 — Cycle-enable toggle (InAutoCycle)
Toggle-from-pulse pattern, Trig=StartStopPulse, Reset=AbortCondition:
| Block | Inputs | Output |
|---|---|---|
| AND4 | P0=StartStopPulse, P1=InAutoCycle (negated, feedback) | SetCycle |
| AND5 | P0=StartStopPulse, P1=InAutoCycle (feedback) | ResetCycleFromToggle |
| OR2 | P0=ResetCycleFromToggle, P1=AbortCondition | ResetCycle |
| FFR1 | S=SetCycle, R=ResetCycle | **InAutoCycle** |

## Stage 4 — Pressure-gated 2s delay pulse
No monoflop exists, so a fresh pressure edge is latched into a `Waiting` flag that holds
long enough for an On-Delay to time out, then the delay's rising edge produces the pulse
and clears the latch:
| Block | Inputs | Output |
|---|---|---|
| REDGE2 (Rising Edge) | P=pressure | PressureEdge |
| AND6 | P0=PressureEdge, P1=InAutoCycle | GatedPressureEdge |
| FFR2 | S=GatedPressureEdge, R=ValveSwitchPulse (feedback, see below) | Waiting |
| ONDELAY2 (On-Delay, Tv=2) | P=Waiting | DelayDone |
| REDGE3 (Rising Edge) | P=DelayDone | **ValveSwitchPulse** |

`ValveSwitchPulse` feeds back into `FFR2.R` to clear `Waiting` once used.

## Stage 5 — Step toggle (which valve is active)
Toggle-from-pulse pattern, Trig=ValveSwitchPulse, Reset=NotInCycle:
| Block | Inputs | Output |
|---|---|---|
| NOT1 | P=InAutoCycle | NotInCycle |
| AND7 | P0=ValveSwitchPulse, P1=StepB (negated, feedback) | SetStepB |
| AND8 | P0=ValveSwitchPulse, P1=StepB (feedback) | ResetStepBFromToggle |
| OR3 | P0=ResetStepBFromToggle, P1=NotInCycle | ResetStepB |
| FFR3 | S=SetStepB, R=ResetStepB | **StepB** |

`StepB=0` → Step A (valve1 active); `StepB=1` → Step B (valve2 active). Resetting on `NotInCycle` ensures every new cycle restarts at Step A/valve1.

## Stage 6 — Output logic
| Block | Inputs | Output |
|---|---|---|
| AND9 | P0=InAutoCycle, P1=StepB (negated) | CycleValve1 |
| AND10 | P0=InAutoCycle, P1=StepB | CycleValve2 |
| AND11 | P0=NotInCycle, P1=ff | ManualValve1 |
| AND12 | P0=NotInCycle, P1=fv | ManualValve2 |
| OR4 | P0=CycleValve1, P1=ManualValve1 | **valve1 (OUT3)** |
| OR5 | P0=CycleValve2, P1=ManualValve2 | **valve2 (OUT4)** |

Note: right after an abort (ff or fv held alone), `InAutoCycle` drops to 0 and manual passthrough re-engages on the same scan — if the button that caused the abort is still physically held, its valve turns on via manual mode. That's expected: "both off" applies to the auto-cycle's own latched state, not to a held button's normal manual action.

## Block Diagram

```mermaid
flowchart LR
    ff([ff / IN1])
    fv([fv / IN2])
    pressure([pressure / IN3])
    valve1([valve1 / OUT3])
    valve2([valve2 / OUT4])

    subgraph S1["Stage 1 — Start/stop detect"]
        AND1["AND"]
        ONDELAY1["On-Delay Tv=2"]
        REDGE1["Rising Edge"]
        AND1 --> ONDELAY1 --> REDGE1
    end
    ff --> AND1
    fv --> AND1
    REDGE1 -->|StartStopPulse| AND4
    REDGE1 -->|StartStopPulse| AND5

    subgraph S2["Stage 2 — Abort detect"]
        AND2["AND (fv negated)"]
        AND3["AND (ff negated)"]
        OR1["OR"]
        AND2 --> OR1
        AND3 --> OR1
    end
    ff --> AND2
    fv --> AND2
    fv --> AND3
    ff --> AND3
    OR1 -->|AbortCondition| OR2

    subgraph S3["Stage 3 — Cycle-enable toggle"]
        AND4["AND (InAutoCycle negated)"]
        AND5["AND (InAutoCycle)"]
        OR2["OR"]
        FFR1["FFR"]
        AND4 -->|SetCycle| FFR1
        AND5 --> OR2
        OR2 -->|ResetCycle| FFR1
    end
    FFR1 -.feedback.-> AND4
    FFR1 -.feedback.-> AND5
    FFR1 -->|InAutoCycle| AND6
    FFR1 -->|InAutoCycle| NOT1
    FFR1 -->|InAutoCycle| AND9
    FFR1 -->|InAutoCycle| AND10

    subgraph S4["Stage 4 — Pressure-gated delay"]
        REDGE2["Rising Edge"]
        AND6["AND"]
        FFR2["FFR"]
        ONDELAY2["On-Delay Tv=2"]
        REDGE3["Rising Edge"]
        REDGE2 --> AND6 -->|GatedPressureEdge| FFR2
        FFR2 -->|Waiting| ONDELAY2 --> REDGE3
    end
    pressure --> REDGE2
    REDGE3 -.feedback R.-> FFR2
    REDGE3 -->|ValveSwitchPulse| AND7
    REDGE3 -->|ValveSwitchPulse| AND8

    subgraph S5["Stage 5 — Step toggle"]
        NOT1["NOT"]
        AND7["AND (StepB negated)"]
        AND8["AND (StepB)"]
        OR3["OR"]
        FFR3["FFR"]
        AND7 -->|SetStepB| FFR3
        AND8 --> OR3
        NOT1 -->|NotInCycle| OR3
        OR3 -->|ResetStepB| FFR3
    end
    FFR3 -.feedback.-> AND7
    FFR3 -.feedback.-> AND8
    FFR3 -->|StepB| AND9
    FFR3 -->|StepB| AND10
    NOT1 -->|NotInCycle| AND11
    NOT1 -->|NotInCycle| AND12

    subgraph S6["Stage 6 — Output logic"]
        AND9["AND (StepB negated)"]
        AND10["AND"]
        AND11["AND"]
        AND12["AND"]
        OR4["OR"]
        OR5["OR"]
        AND9 --> OR4
        AND10 --> OR5
        AND11 --> OR4
        AND12 --> OR5
    end
    ff --> AND11
    fv --> AND12
    OR4 --> valve1
    OR5 --> valve2
```

## Not yet defined
- Power-up initial state (assume all FFR latches reset to 0, i.e. MANUAL, valve1/valve2 off)
- Emergency-stop input, if any
