# DPOS350P Standalone Acquisition

**Status:** HW-confirmed fixed-profile standalone baseline  
**Baseline date:** 2026-10-29  
**Scope:** custom FPGA/SMEC/profile/CSI acquisition path without stock EXE2 runtime execution

## 1. Purpose

This document is the implementation-oriented reference for raw acquisition in the FNIRSI DPOS350P custom runtime.

The current milestone proves that the custom application can initialize the required hardware state, apply the confirmed fixed acquisition profile, start one FPGA-to-CSI transfer, observe hardware completion, and receive the complete expected payload in RAM.

Historical reverse-engineering evidence remains in `FPGA.md`, `SMEC.md`, and `STOCK_LIKE_MINIMAL_RUNTIME.md`. Protocol details and command semantics belong in `FPGA_PROTOCOL.md`. This document focuses on the working acquisition chain and its acceptance criteria.

## 2. Current end-to-end baseline

The confirmed standalone path is:

```text
Timer0 + AVS initialization
        -> fpga_init()
        -> smec_init()
        -> profile_init()
        -> acq_capture()
        -> CSI HW completion
        -> 0x1800-byte payload in RAM
```

No stock EXE2 acquisition function is executed in this path.

### HW-confirmed result

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
CSI BUF0        0x100017C0
CSI IRQ EN      0x00000002
WAIT POLLS      0x00000001

RAM CHANGED     YES
CHANGED BYTES   0x00001800
FIRST CHANGE    0x00000000
LAST CHANGE     0x000017FF
HASH BEFORE     0x50B995C5
HASH AFTER      0x8FD97C05
```

**HW-CONFIRMED:** the complete expected active range `0x0000..0x17FF` changes during the standalone capture.

## 3. Driver structure

The current implementation separates acquisition into these layers:

```text
arch/r11/timer.c       Timer0 + AVS timebase

drivers/fpga_bus.c     PE GPIO control-bus and GPIO/CSI interface transitions
drivers/fpga_protocol.c FPGA selectors, READY, position, bank, stream quantity
drivers/ms5351.c        external clock configuration
drivers/r11_csi.c       R11 CSI receiver and completion polling
drivers/fpga_init.c     FPGA/CSI/clock initialization
drivers/smec.c          minimal SMEC startup/job path
drivers/profile.c       fixed acquisition profile
drivers/acq.c           acquisition prepare and capture orchestration

apps/acq_test/main.c    standalone hardware oracle and diagnostics
```

Use these code-level names in new implementation documentation instead of historical stock addresses when the mapping is already established.

## 4. Platform timing prerequisite

The custom runtime requires both the normal Timer0 setup and the AVS timebase used by profile delays.

Stock `System_Init()` initializes the relevant AVS block through:

```text
System_Init              0x40042D28
  AVS init wrapper       0x40042CD8
    channel 0 setup      0x40042D40  divisor 24000
    channel 1 setup      0x40042D4C  divisor 24
    common setup         0x40042D54
```

Confirmed register state:

```text
0x01C20144 |= 0x80000000
AVS channel 0 divisor = 24000
AVS channel 1 divisor = 24
0x01C20C8C = 0x00175DBF
```

`timer_avs_delay_units()` uses:

```text
AVS CTRL      0x01C20C80
AVS COUNTER0  0x01C20C84
```

The delay primitive clears counter 0, enables counter 0, and waits until:

```text
(counter0 << 1) >= requested_units
```

**STATIC-CONFIRMED + HW-CONFIRMED:** the earlier standalone profile stall was caused by missing AVS initialization. After reproducing the stock AVS setup, the complete profile reaches `STEPS DONE = 0x7F`.

## 5. FPGA initialization prerequisite

`fpga_init()` provides the working standalone FPGA-side baseline required before profile application and capture.

The current path includes:

```text
FPGA control bus and diagnostic ID state
R11 CSI initialization
shared PE interface setup
MS5351M clock initialization
fixed CSI receiver baseline
```

The diagnostic FPGA ID transaction alone is not sufficient evidence of an acquisition-capable state. The HW-confirmed acquisition result applies to the complete current `fpga_init()` path.

## 6. SMEC prerequisite

The current acquisition baseline includes custom `smec_init()`.

Historical SMEC substitution reduced the required startup behavior to the custom minimal S4/job path:

```text
S1  NOT REQUIRED for current fixed-profile oracle
S2  NOT REQUIRED for current fixed-profile oracle
S3  NOT REQUIRED for current fixed-profile oracle
S4  REQUIRED in the tested minimal runtime
```

**HW-CONFIRMED:** custom `smec_init()` is sufficient as part of the standalone path that reaches real CSI completion and receives the complete payload.

This does not prove that SMEC is optional. The confirmed baseline retains SMEC as a required communication/configuration stage.

## 7. Fixed profile

`profile_init()` currently implements one HW-confirmed fixed acquisition profile.

The successful profile completes all seven tracked steps:

```text
PROFILE PASS    YES
STEPS DONE      0x0000007F
FAILED STEP     0x00000000
```

The profile includes direct FPGA configuration and the key-like apply sequences previously recovered from stock behavior.

Known key-like values include:

```text
0x5A5555A5
0xA5AA555A
0x5A5AA5AA
0xA5A5AA55
```

Their exact semantic role is still open. They should be treated as protocol/apply landmarks, not automatically as cryptographic authentication.

## 8. Stream geometry contract

The confirmed fixed profile uses two 16-bit state values:

```text
A = APP_STATE +0x640 = 0x17FA
B = APP_STATE +0x6F2 = 0x0003
```

The stock-derived equations are:

```text
CSI geometry  = A + 2*B
FPGA quantity = 20 + 2*(A+B)
```

For the working profile:

```text
CSI geometry  = 0x1800
FPGA quantity = 0x300E
```

The custom implementation applies this state through:

```text
r11_csi_configure_fixed_stream_geometry()
fpga_protocol_stream_set_quantity(0x300E)
```

**HW-CONFIRMED:** this pair produces a complete contiguous `0x1800`-byte capture.

READY and CSI DONE alone are insufficient success criteria. Earlier partial profiles could reach READY and CSI completion with only a few changed bytes. Correct receiver and FPGA stream geometry must therefore remain part of the acquisition oracle.

## 9. READY preparation

The public preparation API is:

```c
struct acq_prepare_result {
    uint8_t status_before;
    uint8_t status_after;
    uint32_t polls;
    int ready;
};

int acq_prepare(
    uint8_t mode,
    uint32_t max_polls,
    struct acq_prepare_result *result);
```

The current FPGA status register is selector `0x0A`.

The minimal fixed-profile fingerprint is:

```text
before READY  0xC0
READY state   0xC1
```

READY is defined by bit 0:

```c
ready = (status_0a & 1U) != 0U;
```

Do not compare the complete status byte against one fixed value. Higher bits are configuration fingerprints and can change when additional stock configuration groups are applied.

## 10. Capture API

The current public capture interface is:

```c
struct acq_capture_result {
    uint32_t status_before;
    uint32_t status_after;
    uint32_t mode;
    uint32_t buffer0;
    uint32_t irq_enable;
    uint32_t wait_polls;
    int done;
};

int acq_capture(
    uintptr_t buffer,
    uint16_t position,
    uint8_t bank,
    uint32_t timeout_iterations,
    struct acq_capture_result *result);
```

The API is intentionally independent of the historical stock raw-ring addresses. The caller provides the destination buffer, position, bank, and finite timeout.

## 11. Capture sequence

The working custom capture sequence in `drivers/acq.c` is:

```text
acq_configure_stream()
    r11_csi_configure_fixed_stream_geometry()
    fpga_protocol_stream_set_quantity(0x300E)

fpga_protocol_capture_prepare_position(position)
fpga_protocol_capture_select_bank(bank)
fpga_bus_enter_csi_mode()

r11_csi_set_buffer0(buffer)
r11_csi_ack(R11_CSI_DONE)
status_before = r11_csi_status()

r11_csi_timer0_gate(0)
r11_csi_capture_enable()
fpga_capture_gate(1)

r11_csi_wait_done_bounded(timeout_iterations, &status_after)

fpga_capture_gate(0)
r11_csi_timer0_gate(1)
r11_csi_ack(R11_CSI_DONE)
r11_csi_capture_disable()
fpga_bus_leave_csi_mode()
```

All waits must remain bounded.

## 12. CSI transport

The bulk sample path is:

```text
GOWIN FPGA
    -> PE sample pins in CSI function 2
    -> R11 CSI @ 0x01CB4000
    -> CSI buffer 0
    -> caller-provided RAM buffer
```

Relevant CSI registers currently used by the custom driver include:

```text
CSI base         0x01CB4000
MODE             base + 0x08
HSIZE            base + 0x44
BUFFER0          base + 0x50
IRQ ENABLE       base + 0x70
STATUS           base + 0x74
DONE mask        bit 1 / 0x02
```

The destination register stores the physical destination in four-byte units:

```text
CSI BUFFER0 register = physical_address >> 2
```

The standalone diagnostic therefore displays the encoded register value rather than the original byte address.

## 13. Shared PE interface lifecycle

The FPGA control plane and CSI sample plane share PE pins.

Conceptually:

```text
GPIO control mode
    ARM <-> FPGA register/selector traffic
        |
        v
fpga_bus_enter_csi_mode()
        |
        v
CSI sample mode
    FPGA -> R11 CSI bulk stream
        |
        v
fpga_bus_leave_csi_mode()
        |
        v
GPIO control mode restored
```

The capture transaction must preserve this ordering. FPGA control-bus transactions must not be inserted arbitrarily while CSI owns the shared sample pins.

## 14. Timer0 gate during capture

The reconstructed per-bank stock sequence changes bit 0 of Timer0 control around the active capture window. The custom driver preserves that behavior through:

```text
r11_csi_timer0_gate(0)
...
r11_csi_timer0_gate(1)
```

The capture wait itself therefore uses bounded iteration polling rather than the normal Timer0-based millisecond deadline while Timer0 is temporarily stopped.

Do not replace the bounded CSI wait with `timer_get_ms()` inside this gated interval.

## 15. Current capture-buffer model

The standalone application uses a dedicated aligned `0x1800`-byte capture buffer rather than the stock EXE2 raw-ring allocation.

Before capture, the test buffer is filled with a sentinel byte and hashed. After capture the application checks:

```text
CSI completion
changed-byte count
first changed offset
last changed offset
hash before
hash after
```

The confirmed successful result is:

```text
CHANGED BYTES   0x00001800
FIRST CHANGE    0x00000000
LAST CHANGE     0x000017FF
HASH BEFORE     0x50B995C5
HASH AFTER      0x8FD97C05
```

This proves full-range memory modification. It does not by itself establish calibrated voltage/time interpretation of the received sample bytes.

## 16. Relationship to the stock raw ring

Stock firmware uses eight logical raw slots:

```text
RAW_ACQ_BASE     0x42DF76D4
RAW_BANK_STRIDE  0x2134 = 8500 bytes
RAW_BANK_COUNT   8
active payload   0x1800 = 6144 bytes
```

Known stock destinations are:

```text
bank 0  0x42DF76D4
bank 1  0x42DF9808
bank 2  0x42DFB93C
bank 3  0x42DFDA70
bank 4  0x42DFFBA4
bank 5  0x42E01CD8
bank 6  0x42E03E0C
bank 7  0x42E05F40
```

The standalone `acq_capture()` API does not require this allocation scheme. The stock layout remains useful when reproducing multi-bank behavior and downstream stock sample-processing semantics.

The meaning of the unused `0x2134 - 0x1800 = 0x934` bytes per stock logical slot remains open.

## 17. Acceptance levels

Use the following hierarchy when evaluating acquisition changes.

### Level 1: initialization

```text
FPGA INIT  YES
SMEC PASS  YES
```

This proves only that the prerequisite stages completed.

### Level 2: profile

```text
PROFILE PASS  YES
STEPS DONE    0x7F
```

This proves the supported fixed profile was applied.

### Level 3: FPGA READY

```text
FPGA 0x0A bit 0 = 1
```

This proves the current acquisition state machine reports READY.

### Level 4: CSI completion

```text
HW DONE  YES
```

This proves the CSI capture transaction completed, but does not alone establish the expected payload length.

### Level 5: full fixed-profile payload

```text
RAM CHANGED    YES
CHANGED BYTES  0x1800
FIRST CHANGE   0x0000
LAST CHANGE    0x17FF
```

This is the current golden fixed-profile acquisition pass condition.

## 18. Safety and invariants

Keep these rules for every acquisition experiment:

- use finite polling bounds;
- preserve a boot-manager recovery path;
- begin new experiments from the HW-confirmed fixed-profile baseline;
- change one acquisition variable at a time;
- do not infer success only from READY;
- do not infer full transfer only from CSI DONE;
- preserve the shared GPIO/CSI PE transition order;
- do not issue unrelated FPGA diagnostic transactions inside dependency tests;
- keep `-mno-unaligned-access` in ARM builds;
- verify the complete `0x1800` changed range after changes to profile or stream setup.

## 19. What is solved

The following questions are closed for the current fixed profile:

```text
Can the custom runtime initialize the FPGA?          YES
Can custom SMEC participate in a real capture?       YES
Can the custom profile complete?                     YES
Can the FPGA reach READY?                            YES
Can the custom driver start CSI capture?             YES
Does CSI signal completion?                          YES
Can the FPGA stream reach custom RAM?                YES
Is the full expected payload transferred?            YES, 0x1800 bytes
Is stock EXE2 required for this capture path?        NO
```

## 20. What remains open

The acquisition transport itself is no longer the main blocker. Remaining work is primarily about making the fixed baseline general and measurement-correct:

- derive `APP_STATE +0x640/+0x6F2` for additional timebases and modes;
- determine how profile values vary for CH1-only, CH2-only, and dual-channel capture;
- generalize position and bank sequencing;
- characterize multi-bank acquisition in the fully custom runtime;
- identify lane/interleave ordering;
- restore analog frontend/channel configuration;
- determine sample-code calibration and voltage conversion;
- map trigger configuration beyond the fixed profile;
- validate timebase/sample-period correctness;
- implement waveform processing/display only after raw capture remains stable across profiles.

## 21. Next acquisition milestone

The next acquisition milestone should not repeat the already-complete single-buffer smoke test.

The recommended progression is:

```text
1. freeze the current fixed-profile acquisition as the regression oracle
2. run one controlled profile/timebase variation
3. derive the corresponding stream-state values instead of hardcoding them
4. preserve HW DONE + full expected payload at the new profile
5. extend to controlled multi-bank capture
6. correlate banks with channel/interleave semantics
7. only then add calibration and waveform rendering
```

The project has moved from "can the FPGA stream samples?" to "how do we generalize and interpret a proven sample stream?"

### 22. Stock EXE2 full-acquisition and runtime-sweep reconstruction

**Status:** HW-confirmed stock live acquisition and repeated runtime-sweep baseline  
**Baseline date:** 2026-11-02  
**Scope:** stock EXE2 acquisition execution under DPOSLAB after the confirmed stock initialization path. This is a separate reverse-engineering baseline from the standalone custom acquisition path documented above.

The standalone custom path remains valid and unchanged: it proves that stock EXE2 is not required for the fixed-profile single-buffer capture described in sections 2-21. The results below answer a different question: how stock firmware performs its own full acquisition and then advances repeated runtime captures.

#### 22.1 Confirmed stock acquisition entry points

The following stock addresses are now both statically characterized and exercised on hardware:

```text
0x400413C8  full acquisition entry
0x400419CC  acquisition dispatcher
0x4004191C  runtime sweep path
0x4003C520  runtime rearm/setup
0x400417E4  one runtime capture step
0x4003BEB4  runtime state commit/copy
0x4003C9F4  per-channel processing
0x4003BF98  runtime counter/progress advance
0x4003F910  write SYS_INFO+0x6F0 to FPGA selector 0x09
0x4003F8E0  pulse FPGA selector 0x08: 0 -> 1
0x40049B08  runtime/event service
```

The dispatcher at `0x400419CC` selects between the full-acquisition path and the runtime-sweep path using `SYS_INFO+0x28`, subject to the surrounding runtime/event gates.

```text
SYS_INFO+0x28 == 0  -> 0x40041918 -> 0x400413C8
SYS_INFO+0x28 != 0  -> 0x4004191C
```

**STATIC-CONFIRMED:** `0x400413C8` has a real function prologue, takes its working state from `SYS_INFO`, performs the full capture-count loop, and returns normally in the tested stock-initialized DPOSLAB state.

#### 22.2 Native stock CSI ring

Stock acquisition uses the native eight-slot CSI ring:

```text
RAW_ACQ_BASE     0x42DF76D4
RAW_BANK_STRIDE  0x2134
RAW_BANK_COUNT   8
ACTIVE_PAYLOAD   0x1800 bytes per slot
```

The CSI destination register stores byte addresses divided by four. Hardware tests therefore observe the DMA position through the encoded CSI buffer address.

**HW-CONFIRMED:** a successful full stock acquisition moves the native DMA position from slot 0 to slot 7 and returns from `0x400413C8`.

#### 22.3 Cache-coherency requirement for stock-ring inspection

Early DPOSLAB stock-ring experiments read the native ring before DMA and then read it again after DMA. That sequence can leave stale sample lines in the Cortex-A7 D-cache and make post-DMA RAM appear unchanged.

The confirmed diagnostic procedure is therefore:

```text
do not read S0-S7 before acquisition
run stock acquisition
invalidate the native payload ranges with DCIMVAC
DSB + ISB
read/hash S0-S7 for the first time
```

No clean operation is performed before the post-DMA read, because cleaning stale CPU cache lines could overwrite newer DMA data.

**HW-CONFIRMED:** with cache-safe inspection and an applied input signal, all eight native slots contain non-zero, changing payloads. The previous all-zero observation must not be treated as proof that the FPGA streamed zero samples.

#### 22.4 Full acquisition is not the repeated-runtime API

Calling `0x400413C8` repeatedly is not equivalent to stock repeated acquisition.

A four-call experiment using repeated direct full-acquisition calls produced non-stock payload patterns such as live/mixed data followed by all-nonzero/all-zero/all-nonzero buffers. Waiting one second between calls did not make the sequence converge or stabilize.

This ruled out a simple passive FPGA warm-up explanation. The missing mechanism was active stock runtime progression between captures.

**HW-CONFIRMED:** `0x400413C8` is valid as the full-acquisition entry, but repeated stock captures must use the runtime state machine rather than repeated isolated calls to the full-acquisition entry.

#### 22.5 FPGA debug/config phase values B8-C0

The observed FPGA selector `0x0A` fingerprint moves from `0xB8` to `0xB9` during the first full acquisition:

```text
full acquisition:
FPGA 0x0A  B8 -> B9
```

Static analysis also found the stock configuration-sync sequence associated with diagnostic phases:

```text
B9 -> FPGA selector A6
BA -> FPGA selector A7
BB -> FPGA selector A8
BC -> FPGA selector A9
BD -> FPGA selector AA
BE -> FPGA selector AB
BF -> FPGA selector AC
C0 -> FPGA selector AD
```

These phase values are conditional configuration-sync landmarks, not a simple acquisition counter. The stock sync path only emits a phase when the current configuration value differs from its cached value.

**HW-CONFIRMED:** calling the stock FPGA runtime-sync wrapper with unchanged configuration leaves the observed phase at `B9`. Therefore `B9` is not by itself evidence that the acquisition state machine is stuck.

#### 22.6 Confirmed runtime rearm and READY behavior

The runtime rearm function is:

```text
0x4003C520
    -> 0x4003BEA8
    -> 0x4003B7B8
    -> 0x40024E14
    -> 0x4003C008
    -> 0x4003442C
    -> 0x4003F910
    -> 0x4003FA10
    -> 0x4003F8E0
```

This path initializes runtime counters/state, writes the current `SYS_INFO+0x6F0` value through FPGA selector `0x09`, performs the stock trigger/rearm sequence, and pulses selector `0x08` from 0 to 1.

Runtime readiness is read from FPGA selector `0x0B`:

```text
0x4003F8B8:
    read FPGA selector 0x0B
    SYS_INFO+0x6EE = value & 1
```

In the confirmed DPOSLAB runtime test:

```text
before runtime rearm: FPGA 0x0B = B8   bit0 = 0
 after runtime rearm: FPGA 0x0B = B9   bit0 = 1
READY polls: 1
READY elapsed time: 0 ms
```

**HW-CONFIRMED:** runtime READY is immediate after the stock rearm in the tested state. The acquisition problem is therefore not a requirement for passive one-second or multi-second FPGA settling.

#### 22.7 One confirmed stock runtime iteration

After one full acquisition and one runtime rearm, the following sequence runs successfully on hardware:

```text
0x400417E4  runtime capture
0x4003BEB4  commit/copy state
0x4003C9F4  per-channel processing
0x4003BF98  advance runtime counters
0x4003F910  write selector 0x09
0x4003F8E0  pulse selector 0x08
0x40049B08  runtime/event service
```

The runtime capture `0x400417E4` first polls `FPGA 0x0B` through `0x4003F8B8`, then performs the stock CH1/CH2 capture and copy path. In the tested CH1-only state, the function returns normally and updates `SYS_INFO+0x6EE` from 0 to 1.

A complete D0.4B iteration reached its final diagnostic stage without hanging.

**HW-CONFIRMED:** the sequence above is a valid one-step stock runtime acquisition path in the current stock-initialized DPOSLAB environment.

#### 22.8 Runtime counter structure at SYS_INFO+0x6F0

Static analysis of `0x4003BF98` shows that the function uses `SYS_INFO+0x6F0` as the base of a runtime sweep structure. Important fields include:

```text
+0x6F0  16-bit value subsequently written to FPGA selector 0x09
+0x6F6  16-bit per-step increment used by the runtime advance
+0x6F8  16-bit coarse-progress increment
+0x6FC  32-bit fine/current progress value
+0x700  32-bit coarse runtime progress
+0x708  32-bit threshold compared against +0x6FC
```

The important control flow is conceptually:

```text
if (+0x6FC >= +0x708) {
    +0x700 += +0x6F8;
    related accumulated position += +0x6F8;
}

+0x6FC += +0x6F6;
quotient  = +0x6FC / SYS_INFO+0x1B1
remainder = +0x6FC % SYS_INFO+0x1B1
SYS_INFO+0x63C = quotient
SYS_INFO+0x6F0 += quotient
```

The exact semantic names of these fields remain open, but their arithmetic and runtime role are confirmed by static analysis and hardware progression.

#### 22.9 Five-cycle HW-confirmed runtime progression

D0.4C executes one full acquisition, one initial runtime rearm, then five bounded runtime cycles using the complete sequence from section 22.7.

The experiment completed all five cycles:

```text
STAGE = 0xFF
DONE  = 5
```

Observed progression through the first four visible runtime cycles:

```text
state       SYS+6F0   SYS+6FC   SYS+700
----------------------------------------
FULL           02        00         00
INITIAL REARM  08        00         00
C1 after adv   0E        32         00
C2 after adv   14        64         00
C3 after adv   1A        96         01
C4 after adv   21        C8         02
```

Every visible runtime cycle reported:

```text
READY = 1
polls = 1
elapsed = 0 ms
FPGA 0x0A = B9
FPGA 0x0B = B9
```

Runtime ring fingerprints were non-zero and changed between captures. The old pathological `FF / 00 / FF` behavior arose when the full-acquisition entry was incorrectly repeated without the runtime advance/rearm sequence.

**HW-CONFIRMED:** repeated stock acquisition is stable when driven through the reconstructed runtime state machine. Time alone is not the missing prerequisite; active runtime counter advance and rearm are required.

#### 22.10 Updated stock runtime model

The current best-supported stock model is:

```text
FULL START
0x400413C8
    |
    +-> native 8-slot CSI acquisition
    +-> FPGA 0x0A B8 -> B9
    |
    v
RUNTIME REARM
0x4003C520
    |
    +-> runtime counter initialization
    +-> selector 0x09 update
    +-> stock trigger/rearm sequence
    +-> selector 0x08 pulse
    +-> FPGA 0x0B bit0 becomes READY
    |
    v
RUNTIME ITERATION
0x400417E4  capture
0x4003BEB4  commit
0x4003C9F4  process
0x4003BF98  advance counters/progress
0x4003F910  write selector 0x09
0x4003F8E0  pulse selector 0x08
0x40049B08  service
    |
    +---------- repeat ----------+
```

This model is based on both static call-graph reconstruction and hardware execution. It replaces the earlier working assumption that repeated acquisition could be reproduced by repeatedly invoking `0x400413C8`.

#### 22.11 Current acceptance criteria for stock-runtime experiments

For stock-runtime reconstruction, use a stricter acceptance hierarchy than READY or CSI DONE alone:

```text
1. full acquisition returns
2. native CSI DMA reaches slot 7
3. cache-safe native-ring inspection produces a valid payload fingerprint
4. runtime rearm returns
5. FPGA selector 0x0B bit0 reports READY
6. runtime capture returns
7. commit/process/advance all return
8. SYS_INFO+0x6F0/+0x6FC/+0x700 advance consistently
9. selector 0x09 + selector 0x08 rearm the next runtime cycle
10. multiple consecutive runtime cycles complete with non-zero changing payloads
```

Do not use repeated direct calls to `0x400413C8` as an acceptance test for repeated stock acquisition.

#### 22.12 Next stock-runtime RE target

The next static-analysis target is the `SYS_INFO+0x6F0..+0x708` runtime structure. In particular:

```text
+0x6F6  per-step increment
+0x6F8  coarse-progress increment
+0x708  threshold for incrementing +0x700
```

The goal is to explain mathematically the observed hardware progression:

```text
+0x6F0: 08 -> 0E -> 14 -> 1A -> 21 -> ...
+0x6FC: 00 -> 32 -> 64 -> 96 -> C8 -> ...
+0x700: 00 -> 00 -> 00 -> 01 -> 02 -> ...
```

Only after this state machine is understood should the stock runtime path be reduced further or translated into custom driver-level abstractions.
