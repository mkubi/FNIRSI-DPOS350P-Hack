# Stock-Like Minimal Runtime Environment

## Purpose

This document tracks the staged replacement of the DPOS350P stock acquisition runtime with minimal custom implementations while keeping a hardware-confirmed stock-like oracle.

The objective is not to clone the complete stock firmware. The stock runtime is used as a reference environment to separate:

- real hardware requirements,
- stock software bookkeeping,
- optional/default initialization,
- analog/frontend configuration,
- CSI transport requirements,
- and functions that can be replaced by small custom equivalents.

## Evidence labels

- **HW-CONFIRMED**: tested on the DPOS350P hardware.
- **STATIC-CONFIRMED**: established directly from `exe2-from-backup.bin` disassembly/literal pools.
- **WORKING HYPOTHESIS**: plausible interpretation not yet isolated by hardware tests.
- **INVALID EXPERIMENT**: result invalidated by instrumentation or an uncontrolled side effect.
- **DEFERRED**: relevant later, but intentionally outside the current digital acquisition minimum.

## Experimental rule: REQUIRED is baseline-specific

`REQUIRED` always means required for the exact active baseline used by the test.

It must never be interpreted as globally required for every future runtime state.

Whenever the prerequisite chain changes substantially, older dependency conclusions may need to be retested.

A valid SKIP dependency test must not introduce additional FPGA/GPIO transactions. In particular, diagnostic FPGA reads, selector writes, bus-direction changes, or `fpga_bus_safe_idle()` calls can invalidate a dependency result.

---

# Golden digital acquisition oracle

The current oracle proves the complete digital path from a prepared FPGA acquisition state through CSI into raw RAM Bank0.

## Bank0

```text
physical address  0x42DF76D4
logical slot      0x2134 = 8500 bytes
active payload    0x1800 = 6144 bytes
```

## PASS criteria

```text
ACQ READY      YES
HW DONE        YES
RAM CHANGED    YES
CHANGED BYTES  0x00001800
FIRST CHANGE   0x00000000
LAST CHANGE    0x000017FF
```

Natural stock acquisition tracing independently showed changes in exactly 24 consecutive 256-byte blocks:

```text
24 * 256 = 6144 = 0x1800 bytes
```

## Stream-configuration breakthrough

Calling only the low-level per-bank stock primitive:

```text
0x400406C8(0)
```

produced only one changed byte in Bank0.

Adding:

```text
0x400405E0()
0x400406C8(0)
```

changed the result to the complete contiguous `0x1800` payload.

**HW-CONFIRMED:** `0x400405E0` is a higher-level stream-configuration step required by the current hybrid capture path. It configures CSI capture quantity and FPGA stream quantity before the per-bank transfer.

**STATIC-CONFIRMED + HW-CONFIRMED:** `0x400405E0` consumes two unsigned 16-bit application-state fields and coordinates the CSI receiver geometry with FPGA selector `0x22`:

```text
A = APP_STATE +0x640
B = APP_STATE +0x6F2

CSI geometry  = (A + 2*B) & 0xFFFF
FPGA quantity = (20 + 2*(A+B)) & 0xFFFF
```

For the confirmed fixed profile, stock SC1.3 produces `A=0x17FA` and `B=0x0003`, therefore `GEOM=0x1800` and FPGA quantity `0x300E`.

---

# Current substitution status

## FPGA_Start

The stock FPGA initialization was decomposed and progressively substituted.

The custom path is hardware-confirmed for acquisition.

Important F1 CSI compatibility work included:

```text
original custom F1                 FAIL
Descriptor A +0x10 only            FAIL
controller base + CSI +0x00 state  PASS
```

The stock descriptor work was needed to interoperate with stock CSI software. It must not automatically be treated as a requirement of a future fully custom CSI driver.

**HW-CONFIRMED:** custom FPGA_Start is sufficient for the current acquisition baseline.

## SMEC_Init

Dependency isolation produced:

```text
S1  NOT REQUIRED
S2  NOT REQUIRED
S3  NOT REQUIRED
S4  REQUIRED
```

The custom S4/job implementation completes all seven jobs and is sufficient for the current acquisition baseline.

**HW-CONFIRMED:** custom minimal SMEC path passes.

---

# HorizontalInit substitution

Stock:

```text
HorizontalInit @ 0x4003A62C
```

Top-level decomposition:

```text
H1  0x4003A908
H2  0x4003A680
H3  0x4005DB9C
H4  0x4005D27C
H5  0x4003C008
H6  0x4003BEB4
```

## Hardware dependency matrix

```text
H1  NOT REQUIRED
H2  REQUIRED
H3  NOT REQUIRED
H4  NOT REQUIRED
H5  NOT REQUIRED for current digital ACQ oracle
H6  NOT REQUIRED
```

### Important H5 boundary

`H5 @ 0x4003C008` is relevant to stock CSI/capture state, but its omission does not break the current hybrid digital oracle because later stock CSI/stream operations remain active.

Therefore:

```text
H5 SKIP PASS
```

must not be interpreted as proof that H5-like state is unnecessary for a future fully custom CSI implementation.

## Custom H2

`H2 @ 0x4003A680` initializes the stock horizontal/timebase lookup state in RAM without direct FPGA or CSI I/O.

A stock-like custom H2 implementation was created.

A/B result:

```text
H2 STOCK   -> PASS, 0x1800 payload
H2 SKIP    -> FAIL
H2 CUSTOM  -> PASS, 0x1800 payload
```

**HW-CONFIRMED:** the stock HorizontalInit prerequisite has been reduced to custom H2 for the current digital acquisition baseline.

Snapshot:

```text
commit 2836402
experiment: replace HorizontalInit prerequisite with custom H2
```

---

# SysConfigExec investigation

Stock top level:

```text
SysConfigExec @ 0x4003E708
```

The entry checks a guard byte in the structure at `0x42E68BB0`.

When the guard permits execution, the function performs three large groups:

```text
SC1  0x4003A654
SC2  0x4004A510
SC3  0x40063328
```

`AllInfoInit` clears the 24-byte structure beginning at `0x42E68BB0`, including the `+0x17` guard byte, so normal startup allows the SysConfig body to execute.

## Invalid early SKIP result

An early SysConfig SKIP wrapper appeared to pass acquisition.

That wrapper also performed FPGA `0x0A` reads and bus cleanup operations before/after the skipped function.

A direct replacement of the original SysConfig call with a single ARM NOP produced:

```text
ACQ READY  NO
```

Therefore the wrapper result is classified as:

```text
INVALID EXPERIMENT: instrumentation-induced false PASS
```

**HW-CONFIRMED:** whole SysConfigExec remains required in the custom FPGA + custom SMEC + custom Horizontal H2 baseline when tested with a side-effect-free direct NOP.

---

# SysConfig large-group matrix

A side-effect-free dispatcher now tests only the three stock groups and performs no diagnostic FPGA accesses.

```text
G1 / SC1  0x4003A654
G2 / SC2  0x4004A510
G3 / SC3  0x40063328
```

## Group results

### Reference

```text
SC1 STOCK
SC2 STOCK
SC3 STOCK

PASS
```

### SC1 skipped

```text
SC1 SKIP
SC2 STOCK
SC3 STOCK

relay click  YES
0x0A READY   NO
```

**HW-CONFIRMED:** SC1 is required.

### SC2 skipped

```text
SC1 STOCK
SC2 SKIP
SC3 STOCK

relay click  NO
0x0A READY   YES
CSI          PASS
payload      0x1800
```

**HW-CONFIRMED:** SC2 is not required for the current digital acquisition oracle.

**HW-CONFIRMED:** the observed relay click belongs to the SC2 path.

**WORKING HYPOTHESIS:** SC2 is primarily analog frontend/channel/relay application logic. It may still be essential for correct calibrated oscilloscope measurements even though the raw digital transport works without it.

### SC3 skipped

```text
SC1 STOCK
SC2 STOCK
SC3 SKIP

relay click  YES
0x0A READY   YES
CSI          PASS
payload      0x1800
```

**HW-CONFIRMED:** SC3 is not required for the current digital acquisition oracle.

### SC1 only

```text
SC1 STOCK
SC2 SKIP
SC3 SKIP

relay click  NO
0x0A READY   YES
CSI          PASS
payload      0x1800
```

**HW-CONFIRMED:** SC1 alone is sufficient among the three top-level SysConfig groups for the current digital acquisition oracle.

Therefore the current minimal SysConfig prerequisite is:

```text
SC1 @ 0x4003A654
```

### FPGA 0x0A status-profile transitions across SysConfig groups

Further A/B testing showed that SC2 and SC3 are not required for the current digital READY/CSI/0x1800 oracle, but both materially change the higher status bits of FPGA register `0x0A`.

Observed progression:

```text
SC1 only
SC2 SKIP
SC3 SKIP
    -> 0x0A PRE/POST profile C0 -> C1

SC1 + SC2
SC3 SKIP
    -> relay click YES
    -> 0x0A profile moves from C0-class state to B6-class state

SC1 + SC2 + SC3
    -> relay click YES
    -> 0x0A pre-ready state advances from B6 to B8
    -> READY state B9
```

**HW-CONFIRMED:** SC2 causes the observed relay click and also changes the FPGA `0x0A` status profile even though SC2 is not required for the digital acquisition oracle.

**HW-CONFIRMED:** SC3 further changes the FPGA `0x0A` status profile from the SC2-produced B6-class state to the stock-like B8/B9 profile, while remaining unnecessary for READY + CSI + the full `0x1800` payload.

**IMPORTANT:** `NOT REQUIRED` for SC2/SC3 means only that the current transport oracle still passes without them. These functions are hardware-relevant configuration stages and must not be discarded from future analog/frontend or measurement-quality work.

The READY test must therefore continue to use bit 0 rather than equality with a fixed complete `0x0A` value:

```text
READY = (FPGA_0A & 1) != 0
```

The upper bits are now treated as a useful configuration fingerprint rather than as the READY condition itself.

---

# Static decomposition of SC1

```text
SC1 @ 0x4003A654

SC1.1  0x4003BE78
SC1.2  0x4003A908
SC1.3  0x4003A984
SC1.4  0x4003F948
SC1.5  0x4003F894
```

Current interpretation:

```text
SC1.1  derives internal horizontal mode/state
SC1.2  builds mode-dependent runtime fields
SC1.3  major horizontal/timebase compute path
SC1.4  applies a computed value to FPGA selector 0x0E
SC1.5  final horizontal/apply chain
```

SC1.4 is not a plain `void(void)` helper. Stock SC1 loads the input from the application-state area around `0x40544E68 + 0x30` before calling `0x4003F948`.

A future SC1 subphase dispatcher must reproduce this argument exactly and must not introduce additional FPGA transactions.

---

# SC1 hardware reduction and custom substitution status

## SC1 dependency matrix

```text
SC1.1  0x4003BE78   NOT REQUIRED
SC1.2  0x4003A908   REQUIRED      -> CUSTOM PASS
SC1.3  0x4003A984   REQUIRED      -> CUSTOM T6 PASS (fixed profile)
SC1.4  0x4003F948   NOT REQUIRED
SC1.5  0x4003F894   REQUIRED
```

**HW-CONFIRMED:** the cumulative minimal SC1 chain is `SC1.2 + SC1.3 + SC1.5`. With SC1.1 and SC1.4 skipped, READY, CSI completion, and the complete `0x1800` Bank0 payload are preserved.

## Custom SC1.2

```text
SC1.2 STOCK   -> PASS
SC1.2 SKIP    -> FAIL
SC1.2 CUSTOM  -> PASS
```

**HW-CONFIRMED:** custom SC1.2 is complete and preserves the minimal `0x0A` fingerprint `C0 -> C1`.

## SC1.5 decomposition and dependency matrix

```text
SC1.5.1  0x4003F2F4   REQUIRED
SC1.5.2  0x4003F160   REQUIRED   -> CUSTOM PASS
SC1.5.3  0x40063F04   REQUIRED
SC1.5.4  0x40063CD0   REQUIRED
SC1.5.5  0x40063CB0   NOT REQUIRED
```

**HW-CONFIRMED:** cumulative SC1.5 minimum `SC1.5.1 + SC1.5.2 + SC1.5.3 + SC1.5.4`, with SC1.5.5 skipped, passes the full digital oracle.

## Custom SC1.5.2 and ABI finding

The first custom replacement of `SC1.5.2 @ 0x4003F160` failed with FPGA `0x0A` stuck at `C0`. Reproducing the 32-bit shadow window in `r2` alone was insufficient.

Static inspection of stock helper `0x40049EC4` established its relevant call ABI:

```text
r0 = parameter index
r1 = width/mode = 1
r2 = 32-bit shadow window
r3 = source byte
```

The helper selects its effective value as:

```text
effective_value = (r1 < 4) ? r3 : r2
```

Thus the source byte in `r3` is required for the SC1.5.2 calls using parameters `0x21`, `0x22`, and `0x23`. The original three-argument custom wrapper left `r3` undefined and prevented READY.

After correcting the wrapper to preserve the stock `r0-r3` semantics:

```text
SC1.5.2 STOCK          -> C0 -> C1, READY YES, 0x1800 PASS
SC1.5.2 SKIP           -> C0 remains C0, FAIL
SC1.5.2 CUSTOM pre-fix -> C0 remains C0, FAIL
SC1.5.2 CUSTOM fixed   -> C0 -> C1, READY YES, 0x1800 PASS
```

**HW-CONFIRMED:** custom SC1.5.2 is complete for the current minimal runtime.

**STATIC-CONFIRMED:** future custom code calling `0x40049EC4` must preserve its stock register-level argument semantics rather than modeling the relevant calls as a simple three-argument function.

## Current confirmed implementation baseline

```text
CUSTOM
  FPGA_Start
  SMEC S4/job minimum
  Horizontal H2
  SysConfig SC1.2
  SysConfig SC1.3 T6 fixed-profile state
  SysConfig SC1.5.1
  SysConfig SC1.5.2
  SysConfig SC1.5.3
  SysConfig SC1.5.4

STOCK / retained helpers
  0x4004C544       retained inside current SC1.3 T6
  0x4003FA10       acquisition prepare
  0x400405E0       stream geometry + FPGA selector 0x22 configuration
  0x400406C8(0)    per-bank Bank0 capture

SKIP
  SC1.1
  SC1.4
  SC1.5.5
  SC2
  SC3
```

Current hardware fingerprint:

```text
FPGA 0x0A PRE    C0
FPGA 0x0A READY  C1
READY bit        bit 0
HW DONE          YES
stream GEOM      0x1800
FPGA 0x22 qty    0x300E
Bank0 payload    0x1800 bytes
FIRST CHANGE     0x0000
LAST CHANGE      0x17FF
```

A later substitution that changes `C0 -> C1`, `GEOM=0x1800`, `QTY=0x300E`, or the contiguous `0x1800` payload should trigger an A/B investigation even if READY remains set.

# Custom SC1.3 fixed-profile milestone

SC1.3 @ `0x4003A984` has now been substituted far enough to reproduce the complete digital oracle for the currently supported fixed horizontal profile without calling the full stock SC1.3 implementation.

## T1..T5 partial pass

T1 through T5 progressively restored the confirmed primary timing profile, normalized timing, signed bounds/current position, final timing limits, and the RAM-only helper `0x4004C544`. All five variants reached acquisition READY and CSI DONE but transferred only six bytes:

```text
FPGA 0x0A PRE    0xC0
FPGA 0x0A READY  0xC1
READY            YES
HW DONE          YES
CHANGED BYTES    0x00000006
FIRST CHANGE     0x00000000
LAST CHANGE      0x00000005
```

**HW-CONFIRMED:** READY and CSI DONE are necessary but insufficient success criteria for the bulk sample path.

## Stock `0x400405E0` consumer

Static analysis showed that the relevant stream-size inputs are only:

```text
A = APP_STATE +0x640  (u16)
B = APP_STATE +0x6F2  (u16)
```

and that stock computes:

```text
CSI geometry  = (A + 2*B) & 0xFFFF
FPGA quantity = (20 + 2*(A+B)) & 0xFFFF
```

`0x400405E0` first calls `0x40065FA8(CSI geometry)`, selects FPGA command `0x22`, and writes the computed quantity as a big-endian 16-bit value.

A stock-SC1.3 reference run captured immediately before `0x400405E0`:

```text
APP_STATE +0x640 = 0x17FA
APP_STATE +0x6F2 = 0x0003
GEOM             = 0x1800
FPGA quantity    = 0x300E
```

The arithmetic matches exactly:

```text
0x17FA + 2*0x0003         = 0x1800
20 + 2*(0x17FA + 0x0003) = 0x300E
```

## T6 full pass

T6 added only the two golden halfwords above to the custom fixed-profile SC1.3 state. The downstream stock `0x400405E0` remained unchanged.

```text
FPGA 0x0A PRE    0xC0
FPGA 0x0A READY  0xC1
READY            YES
HW DONE          YES
GEOM             0x1800
FPGA quantity    0x300E
CHANGED BYTES    0x00001800
FIRST CHANGE     0x00000000
LAST CHANGE      0x000017FF
```

**STATIC-CONFIRMED + HW-CONFIRMED:** the fixed-profile missing link between custom SC1.3 and the full Bank0 transfer is `+0x640/+0x6F2 -> 0x400405E0 -> CSI geometry / FPGA selector 0x22`.

The current SC1.3 implementation is deliberately profile-specific. General derivation of `+0x640/+0x6F2` for other timebases, channel counts, and acquisition modes remains open.

The T6 implementation still calls stock `0x4004C544`. T5 proved that this RAM-only helper by itself did not restore the full payload, but it must still be custom-replaced before SC1.3 can be called fully stock-independent.

# Deferred SysConfig groups

## SC2

```text
SC2 @ 0x4004A510
```

Static analysis shows extensive channel/frontend configuration and FPGA/application apply helpers.

Because skipping SC2 removes the relay click while preserving the digital oracle, SC2 is deferred until analog frontend/channel bring-up.

Potential internal phases already identified:

```text
SC2.1   0x4004A7C4
SC2.2   0x4004A608
SC2.3   0x4004C544
SC2.4   0x4004AA04
SC2.5   0x4004AA70
SC2.6   0x4004AAC0
SC2.7   0x4004AC04
SC2.8   0x4004AB60
SC2.9   0x4004AB84
SC2.10  0x4004ACA4
SC2.11  0x4004ABA8
SC2.12  0x4004AA24
```

## SC3

```text
SC3 @ 0x40063328
```

SC3 performs additional FPGA acquisition parameter application but is not required for the current digital oracle.

Potential internal phases:

```text
SC3.1  0x400635D0
SC3.2  0x40063354
SC3.3  0x4006366C
SC3.4  0x40063430
SC3.5  0x40063460
SC3.6  0x400634F0
SC3.7  0x4003FAE8
```

This work is deferred until a future measurement-quality oracle requires it.

---

# Current minimal stock-like acquisition environment

## Custom and hardware-confirmed

```text
FPGA_Start         custom
SMEC               custom minimal S4/job path
HorizontalInit     custom H2 only
SysConfig SC1.2    custom
SysConfig SC1.3    custom T6 fixed-profile state
SysConfig SC1.5.1  custom
SysConfig SC1.5.2  custom
SysConfig SC1.5.3  custom
SysConfig SC1.5.4  custom
```

## Stock helpers still active for the digital oracle

```text
0x4004C544       retained inside current SC1.3 T6
0x4003FA10       acquisition prepare
0x400405E0       stream geometry + FPGA selector 0x22 configuration
0x400406C8(0)    stock per-bank Bank0 capture
```

## Not required for the current fixed-profile digital oracle

```text
SMEC S1/S2/S3
Horizontal H1/H3/H4/H5/H6
SC1.1
SC1.4
SC1.5.5
SysConfig SC2
SysConfig SC3
```

## Active but still to be investigated

```text
AllInfoInit
```

## Deferred

```text
SC2 analog frontend / relay configuration
SC3 extra acquisition parameter apply
general SC1.3 profile derivation
complete custom CSI runtime
IRQ/callback parity
measurement/calibration correctness
```

# Custom SC1 / G1 milestone status

The original SC1 dependency-matrix plan is complete for the confirmed fixed profile:

```text
SC1.1  SKIP
SC1.2  CUSTOM PASS
SC1.3  CUSTOM T6 PASS (fixed profile; retains 0x4004C544)
SC1.4  SKIP
SC1.5  CUSTOM PASS: SC1.5.1..SC1.5.4
SC1.5.5 SKIP
SC2     SKIP
SC3     SKIP
```

Acceptance now requires all of:

```text
0x0A READY       YES
HW DONE          YES
GEOM             0x1800
FPGA 0x22 qty    0x300E
CHANGED BYTES    0x1800
FIRST CHANGE     0
LAST CHANGE      0x17FF
```

The next SC1-specific work is to remove `0x4004C544` and reverse the general producers of `APP_STATE +0x640/+0x6F2`, rather than performing another SC1 dependency bisect.

# Roadmap

```text
[DONE] custom FPGA_Start
[DONE] custom minimal SMEC S4
[DONE] custom Horizontal H2
[DONE] isolate SysConfig top-level minimum to SC1
[DONE] custom SC1.2
[DONE] full custom minimal SC1.5.1..SC1.5.4
[DONE] fixed-profile custom SC1.3 T6 restores full 0x1800 payload
[NOW ] replace retained SC1.3 helper 0x4004C544
[NOW ] reverse general producers of APP_STATE +0x640/+0x6F2
[NEXT] custom equivalent of 0x400405E0
[THEN] custom acquisition prepare + per-bank CSI capture
[THEN] re-bisect AllInfoInit in the reduced runtime
[LATER] restore/implement analog frontend configuration
[LATER] measurement and calibration validation
```

## SC1.5 completion update

### Custom SC1.5.3

Runtime capture identified the active inputs to `SC1.5.3 @ 0x40063F04`:

```text
MODE  = APP_STATE +0x1B0 = 0x00000001
KEY   = APP_STATE +0x538 = 0x4C1A6700
RATE  = APP_STATE +0x53C = 0x00000037
```

SC1.5.3 is a RAM-state builder in this path. A strict custom implementation for the HW-confirmed profile was implemented without a stock fallback. Unknown profiles fail closed.

```text
SC1.5.3 STOCK   -> PASS
SC1.5.3 SKIP    -> FAIL
SC1.5.3 CUSTOM  -> C0 -> C1, CSI PASS, 0x1800 PASS
```

**HW-CONFIRMED:** custom SC1.5.3 passes.

### Custom SC1.5.4

`SC1.5.4 @ 0x40063CD0` applies the eight-byte state block at `+0x674..+0x67B`, including parameter indices `0x19..0x20`, commits `0xB9..0xC0`, FPGA selectors `0xA6..0xAD`, and the final selector `0x11` pulse. The custom implementation preserves the corrected four-register `0x40049EC4` ABI established during SC1.5.2 reverse engineering.

```text
SC1.5.4 STOCK   -> PASS
SC1.5.4 SKIP    -> FAIL
SC1.5.4 CUSTOM  -> C0 -> C1, CSI PASS, 0x1800 PASS
```

**HW-CONFIRMED:** custom SC1.5.4 passes.

### Custom SC1.5.1 timing lookup reverse engineering

Runtime capture identified the active inputs to `SC1.5.1 @ 0x4003F2F4`:

```text
DIV    = APP_STATE +0x1B1 = 0x00000008
SRC    = APP_STATE +0x524 = 0x00000008
MODE   = APP_STATE +0x588 = 0x00000017
SCALE  = APP_STATE +0x01C = 0x0000005F

64-bit timing base (+0x538:+0x53C)
       = 0x000000374C1A6700
```

The stock arithmetic for this profile reduces to:

```text
q = SRC / DIV = 8 / 8 = 1
timing = 0x000000374C1A6700 * q
MODE != 11 -> timing = timing * 100 / 95
lookup_key = timing / 2
           = 125000000000
           = 0x0000001D1A94A200
```

`0x400657FC` implements a computed dispatch over 24 inline entries. The calculated key matches the entry with displacement `0x7C`, which dispatches to `0x4003F4DC`.

The selected branch sets:

```text
r4 = 4
r5 = 5
r6 = 1
r7 = 0
stack parameter = 0x3E
```

The common tail at `0x4003F868` maps these values to:

```text
APP_STATE +0x6E4 = 0x3C
APP_STATE +0x6E5 = 0x3B
APP_STATE +0x6E6 = 0x3E
APP_STATE +0x6E7 = 0x01
APP_STATE +0x6E8 = 0x00000000
```

Independent hardware capture matched exactly:

```text
OUT4 = 0x013E3B3C
OUT8 = 0x00000000
```

A strict custom SC1.5.1 implementation validates the full supported input profile and reproduces the selected branch/common-tail result without calling stock `0x4003F2F4` or `0x400657FC`. Unknown profiles fail closed.

```text
SC1.5.1 STOCK   -> PASS
SC1.5.1 SKIP    -> FAIL
SC1.5.1 CUSTOM  -> C0 -> C1, READY YES, CSI PASS, 0x1800 PASS
```

**STATIC-CONFIRMED + HW-CONFIRMED:** the current timing profile maps to lookup key `0x0000001D1A94A200`, branch `0x4003F4DC`, and output `OUT4=0x013E3B3C`, `OUT8=0`.

### Full custom minimal SC1.5 milestone

All hardware-required SC1.5 subphases now pass together without their stock implementations:

```text
SC1.5.1  CUSTOM  PASS
SC1.5.2  CUSTOM  PASS
SC1.5.3  CUSTOM  PASS
SC1.5.4  CUSTOM  PASS
SC1.5.5  SKIP    NOT REQUIRED
```

Cumulative acceptance result:

```text
FPGA 0x0A PRE    C0
FPGA 0x0A READY  C1
READY            YES
CSI              PASS
Bank0 payload    0x1800 bytes
```

**HW-CONFIRMED:** minimal SC1.5 is now stock-independent for the currently supported horizontal profile.

### Historical baseline after full custom SC1.5

```text
CUSTOM
  FPGA_Start
  SMEC S4/job minimum
  Horizontal H2
  SysConfig SC1.2
  SysConfig SC1.5.1
  SysConfig SC1.5.2
  SysConfig SC1.5.3
  SysConfig SC1.5.4

STOCK
  SysConfig SC1.3

SKIP
  SC1.1
  SC1.4
  SC1.5.5
  SC2
  SC3
```

Golden fingerprint remains `0x0A C0 -> C1` with the full `0x1800` Bank0 payload.

### Roadmap update

```text
[DONE] custom minimal SC1.5: SC1.5.1-4 custom, SC1.5.5 skipped
[DONE] custom SC1.3 T6 fixed-profile full-payload pass
[NOW ] remove retained SC1.3 helper and generalize stream-state derivation
[NEXT] custom 0x400405E0 and remaining acquisition runtime
```

## Standalone custom acquisition milestone (2026-10-29)

The fixed-profile digital acquisition path is now hardware-confirmed in `apps/acq_test` without executing the stock EXE2 acquisition runtime.

### Standalone path

```text
custom Timer0 + AVS initialization
custom FPGA initialization
custom minimal SMEC initialization
custom fixed-profile configuration
custom acquisition prepare / stream configuration
custom CSI capture
```

### Missing System_Init prerequisite: AVS timebase

The standalone profile initially stalled in `timer_avs_delay_units(2)`. Static analysis of stock `System_Init` @ `0x40042D28` established the missing AVS initialization chain:

```text
System_Init              0x40042D28
  AVS init wrapper       0x40042CD8
    AVS channel 0 setup  0x40042D40  divisor 24000
    AVS channel 1 setup  0x40042D4C  divisor 24
    common AVS setup     0x40042D54
```

The common setup enables bit 31 at `0x01C20144` and programs the two 16-bit divider fields at `0x01C20C8C`. For the stock configuration:

```text
AVS enable register  0x01C20144 |= 0x80000000
AVS channel 0 divisor           = 24000
AVS channel 1 divisor           = 24
AVS divider register 0x01C20C8C = 0x00175DBF
```

The stock delay primitive then clears AVS counter 0, enables AVS counter 0 through `0x01C20C80`, and waits on `0x01C20C84 << 1`. This matches the custom `timer_avs_delay_units()` behavior. The missing prerequisite was the AVS initialization, not the delay primitive itself.

**STATIC-CONFIRMED:** the AVS initialization above is part of stock `System_Init` and precedes use of the AVS delay primitive.

**HW-CONFIRMED:** after adding the AVS setup to custom timer initialization, the complete fixed profile reaches completion instead of stalling in `timer_avs_delay_units(2)`.

### Standalone capture result

```text
FPGA INIT       YES
SMEC PASS       YES
PROFILE PASS    YES
STEPS DONE      0x0000007F
FAILED STEP     0x00000000

CAPTURE CALL    YES
CAPTURE PASS    YES
HW DONE         YES
CSI BEFORE      0x00000000
CSI AFTER       0x00000083
CSI MODE        0x00000000
CSI IRQ EN      0x00000002
WAIT POLLS      0x00000001

RAM CHANGED     YES
CHANGED BYTES   0x00001800
FIRST CHANGE    0x00000000
LAST CHANGE     0x000017FF
```

**HW-CONFIRMED:** the standalone custom runtime completes the entire fixed-profile digital path through CSI and transfers the full contiguous `0x1800`-byte payload into RAM.

The stock EXE2 runtime is therefore no longer required to execute the confirmed fixed-profile digital capture path.

### Current code-level naming

For the standalone implementation, prefer the current code names over historical address-only names when both are known:

```text
stock FPGA_Start / F1-F4        -> fpga_init() and related FPGA helpers
stock minimal SMEC S4/job path  -> smec_init()
fixed SC1 profile path          -> profile_init()
0x400405E0 stream setup role    -> acq_configure_stream()
stock acquisition prepare role  -> acq_prepare()
per-bank CSI transfer role      -> acq_capture()
CSI fixed geometry setup        -> r11_csi_configure_fixed_stream_geometry()
FPGA selector 0x22 quantity     -> fpga_protocol_stream_set_quantity()
```

Historical stock addresses remain useful as reverse-engineering anchors, but the custom-runtime sections should use the code-level names above wherever the mapping is established.

### SMEC boundary

This standalone result does **not** establish that SMEC is unnecessary. The successful path still includes the custom minimal SMEC implementation. The result establishes that the custom SMEC path is sufficient for real FPGA acquisition, including CSI completion and a complete payload transfer.

The key-like / "magic" FPGA register sequences used during profile application also remain part of the working FPGA protocol. Their exact semantic role is not yet proven; they may recur while additional FPGA commands are identified.

## Current standalone fixed-profile baseline

### Custom and hardware-confirmed

```text
Timer0 + AVS initialization  custom
FPGA initialization         custom
SMEC initialization         custom minimal path
Profile configuration       custom fixed profile
Stream configuration        custom
Acquisition prepare         custom
CSI capture                 custom
```

### Fixed stream profile

```text
CSI payload / geometry  0x1800 bytes
FPGA stream quantity    0x300E
APP_STATE +0x640        0x17FA
APP_STATE +0x6F2        0x0003
```

### Acceptance oracle

```text
FPGA INIT       YES
SMEC PASS       YES
PROFILE PASS    YES
STEPS DONE      0x0000007F
FAILED STEP     0x00000000
CAPTURE PASS    YES
HW DONE         YES
RAM CHANGED     YES
CHANGED BYTES   0x00001800
FIRST CHANGE    0x00000000
LAST CHANGE     0x000017FF
```

This standalone oracle supersedes the older hybrid stock-like oracle as the primary acceptance test for the currently supported fixed profile. The older stock/hybrid results remain valuable historical evidence for how the minimum was derived.

## Updated roadmap

```text
[DONE] custom FPGA initialization
[DONE] custom minimal SMEC path
[DONE] custom Horizontal/profile prerequisite reduction
[DONE] custom fixed-profile SC1/profile state
[DONE] custom stream geometry + FPGA quantity configuration
[DONE] custom acquisition prepare + CSI capture
[DONE] standalone fixed-profile capture: HW DONE + full 0x1800 payload
[NOW] generalize profile derivation beyond the fixed APP_STATE +0x640/+0x6F2 values
[NEXT] identify and name additional FPGA commands and their key/magic apply sequences
[LATER] restore/implement analog frontend configuration
[LATER] validate measurement/calibration correctness
[LATER] IRQ/callback parity where required
```
