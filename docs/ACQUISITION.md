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
