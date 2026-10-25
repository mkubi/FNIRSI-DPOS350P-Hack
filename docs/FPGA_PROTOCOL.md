# FNIRSI DPOS350P ARM-to-FPGA Protocol

> **Status:** Working implementation reference  
> **Target firmware build:** `8eeb56ba0086` / FNIRSI update dated 2025-06-19  
> **Last updated:** 2026-10-25  
> **Companion evidence record:** `docs/FPGA.md`

This document is the concise implementation reference for the DPOS350P ARM-to-FPGA interface. It records what a driver or standalone acquisition application can rely on today. Investigation history, disassembly notes, failed experiments, call graphs, and competing hypotheses belong in `docs/FPGA.md`.

The command numbers used by older FNIRSI instruments must not be copied into the DPOS350P driver without local confirmation. The DPOS350P uses a related eight-bit FPGA interface, but sample transfer and several command meanings differ from the FNIRSI-1013D/1014D generation.

---

## 1. Evidence labels

Every protocol claim should carry one of these labels.

- **HW-CONFIRMED**: observed repeatedly on the local DPOS350P hardware.
- **STATIC-CONFIRMED**: established from the local EXE2 disassembly or exact stock call sites.
- **EXTERNAL HW-CONFIRMED**: measured independently on another DPOS350P and consistent with the local build.
- **EXTERNAL STATIC-CONFIRMED**: reconstructed independently from firmware, but not yet reproduced locally on hardware.
- **INFERRED**: interpretation consistent with current evidence, but not yet proven.
- **OPEN**: unknown, conflicting, or insufficiently tested.

Driver code should use neutral names for **INFERRED** and **OPEN** commands. Do not encode a speculative meaning into a permanent API.

---

## 2. Firmware-build scope

Absolute addresses in this document apply to the local 2025-06-19 EXE2 build unless noted otherwise.

### Local build addresses

```text
EXE2 base               0x40000000
FPGA_READ               0x400491B8
FPGA_WRITE              0x40049214
FPGA_CMD / SELECTOR     0x4004927C
FPGA_READ return        0x40049210
SYS_INFO                 0x40544E68
SYS_INFO run/stop byte  +0x3C
SYS_INFO run counter    +0x64C
CSI capture ring        0x42DF76D4
```

**STATIC-CONFIRMED / independently cross-checked:** The primitive addresses and capture-ring base match an independent reconstruction of the same firmware build.

A separate factory/reference build uses shifted addresses, including capture-ring base `0x42DF76EC`. Never use an absolute address without identifying the firmware build.

---

## 3. Physical ARM-to-FPGA bus

### 3.1 Data and control pins

**HW-CONFIRMED / STATIC-CONFIRMED:**

```text
PE8..PE15    eight-bit bidirectional data bus
PE0..PE3     control, direction, and strobe functions
```

The exact electrical names of all four control signals remain **OPEN**. The stock primitives reliably implement three logical operations:

```text
fpga_select(command)
fpga_write_byte(value)
fpga_read_byte() -> value
```

### 3.2 Local primitive API

Recommended low-level API:

```c
void fpga_select(uint8_t command);
void fpga_write_byte(uint8_t value);
uint8_t fpga_read_byte(void);
```

Recommended framing helpers:

```c
void fpga_write_u16_be(uint8_t command, uint16_t value);
void fpga_write_u24_be(uint8_t command, uint32_t value);
void fpga_write_u32_be(uint8_t command, uint32_t value);

uint16_t fpga_read_u16_be(uint8_t command);
uint32_t fpga_read_u32_be(uint8_t command);
```

Multi-byte values observed in the `0x64..0x6B` path are assembled in bus order:

```c
value = (value << 8) | byte;
```

---

## 4. Diagnostic identity transaction

### FPGA alive/identity check

**HW-CONFIRMED:** A standalone Custom App reached the FPGA through the entirely custom boot chain and reproduced the diagnostic response:

```text
select 0x01
read four bytes

RAW5 = 00 A5 A5 5A 5A
```

The four meaningful response bytes are:

```text
A5 A5 5A 5A
```

This proves that the FPGA is already configured and responsive after:

```text
BootROM
  -> custom Stage 1
  -> custom Stage 2
  -> Custom App
```

It does **not** prove that acquisition, trigger processing, CSI streaming, or all analog-front-end settings are initialized.

---

## 5. Current command map

This map intentionally contains only locally established commands and clearly labelled external leads.

### `0x01`: FPGA identity/alive check

- Direction: read.
- Length: four meaningful bytes.
- Result observed: `A5 A5 5A 5A`.
- Evidence: **HW-CONFIRMED**.

### `0x02`: charging/status input

- Direction: read.
- Length: one byte.
- Reported meaning: bit 0 reflects the charging-status input routed through the FPGA.
- Evidence: **EXTERNAL STATIC-CONFIRMED**.
- Local verification: **OPEN**.

### `0x03`: power-off command

- Direction: write.
- Length: one byte, reported value `1`.
- Reported behavior: requests power-off, after which stock firmware waits for power removal.
- Evidence: **EXTERNAL STATIC-CONFIRMED**.
- Local hardware test is intentionally not required for initial acquisition work.

### `0x0A`: acquisition/status byte

- Direction: read.
- Length: one byte.
- Confirmed static use: bit 0 participates in an acquisition-completion/status check.
- Evidence: **STATIC-CONFIRMED**.
- Exact meanings of the remaining bits: **OPEN**.

### `0x10`: trigger level

- Direction: write.
- Reported source: trigger-level setting.
- Evidence: **EXTERNAL STATIC-CONFIRMED**.
- Exact byte width and encoding for the local driver: **OPEN**.

### `0x15`: trigger channel

- Direction: write.
- Reported values: `0 = CH1`, `1 = CH2`.
- Evidence: **STATIC-CONFIRMED / EXTERNAL STATIC-CONFIRMED**.

### `0x16`: trigger edge

- Direction: write.
- Reported values: `0 = rising`, `1 = falling`.
- Evidence: **EXTERNAL STATIC-CONFIRMED**.

### `0x1A`: trigger mode

- Direction: write.
- Reported values: `0 = auto`, `1 = single`, `2 = normal` in the DPOS350P settings model.
- Evidence: **EXTERNAL STATIC-CONFIRMED**.
- Local runtime confirmation: **OPEN**.

### `0x20`, `0x21`, `0x22`

- Direction and width: acquisition-path dependent.
- Current working role: acquisition quantity, bank, or capture-control parameters.
- Evidence: **INFERRED** from local static acquisition reconstruction.
- Exact public names must remain neutral until a focused trace confirms each field.

### `0x28`: CH1 vertical apply path

- Reported role: CH1 V/div and related position/derived settings.
- Evidence: **EXTERNAL STATIC-CONFIRMED**.

### `0x29`: CH1 bandwidth-derived filter

- Evidence: **EXTERNAL STATIC-CONFIRMED**.

### `0x2B`: CH2 vertical apply path

- Reported role: CH2 V/div and related position/derived settings.
- Evidence: **EXTERNAL STATIC-CONFIRMED**.

### `0x2C`: CH2 bandwidth-derived filter

- Evidence: **EXTERNAL STATIC-CONFIRMED**.

### `0x2F`: CH1 coupling

- Reported values: setting-dependent AC/DC selection.
- Evidence: **EXTERNAL STATIC-CONFIRMED**.

### `0x30`: CH2 coupling

- Reported values: setting-dependent AC/DC selection.
- Evidence: **EXTERNAL STATIC-CONFIRMED**.

### `0x64..0x6B`: asynchronous FPGA job/transform protocol

- Direction: mixed read/write.
- Evidence: transaction framing and runtime behavior are **HW-CONFIRMED**.
- Backend purpose: **OPEN**. Current evidence supports a general calculation, transform, or job engine. It must not be documented solely as a direct physical SMEC command protocol.

---

## 6. Asynchronous job protocol `0x64..0x6B`

### 6.1 Confirmed transaction framing

**HW-CONFIRMED:** DPOSDBG v4.1 / trace format v5 closes the remaining primitive-coverage gaps. Every one of the 50 complete startup jobs has the exact payload counts `2 / 1 / 4 / 4` for `0x69 / 0x6A / 0x6B / 0x66`. Record 0 is a pre-job fragment and is intentionally incomplete.

```text
SEL   0x69
WRITE byte 0
WRITE byte 1

SEL   0x6A
WRITE byte 0

SEL   0x6B
WRITE input byte 0 (MSB)
WRITE input byte 1
WRITE input byte 2
WRITE input byte 3 (LSB)

SEL   0x68
WRITE 0x01

SEL   0x64
READ  state_before

SEL   0x68
SEL   0x65
READ repeatedly until bit 0 becomes 1

SEL   0x68
SEL   0x67
READ  state_after

SEL   0x66
READ  byte 0
READ  byte 1
READ  byte 2
READ  byte 3
```

### 6.2 Confirmed field behavior

```text
0x69    exactly two written bytes in every complete observed startup job
0x6A    exactly one written byte in every complete observed startup job
0x6B    exactly four written bytes in every complete observed startup job; assembled MSB-first as a 32-bit input value
0x68    control/start; observed start value is 0x01
0x64    one state/status byte before completion polling
0x65    completion/status byte; bit 0 is the confirmed ready bit
0x67    one returned state/result byte
0x66    four reads; currently each read equals the 0x67 byte
```

The earlier v3 interpretation (`0x69 = 1 byte`, `0x6A = no data`, `0x6B = 3 bytes`) was a tracer coverage artifact caused by five unpatched tail branches. DPOSDBG v4.1 patches those tail branches while preserving ARM tail-call semantics. The corrected byte counts above supersede the v3 interpretation.

### 6.3 DPOSDBG v4.1 coverage validation

**HW-CONFIRMED:** A clean v4.1 startup capture produced:

```text
trace format       5
capacity           768
written records    51
wrapped            0
exports            1
dropped            0
record size        64 bytes
```

Of the 51 records, record 0 is a startup fragment. The remaining 50 jobs are complete and all have:

```text
bytes_69           2
bytes_6A           1
bytes_6B           4
bytes_66           4
begin_call_site    0x40049FC4
last_call_site     0x4004A0A4
```

The v4.1 Stage 2 patch map adds five previously uncovered ARM tail branches:

```text
0x40049F78
0x40049FB4
0x40049FD8
0x4004A01C
0x4004A07C
```

These account for the missing `0x6A` byte, final `0x6B` byte, second `0x69` byte, stop/control write, and one tail read. This is the reason the old three-byte payload model was incomplete.

Complete jobs currently also carry the diagnostic `UNEXPECTED` flag. This does **not** indicate missing payload: all 50 complete jobs satisfy the exact `2/1/4/4` byte-count invariant, with no overflow, dropped records, or ring wrap. The specific newly covered operation that triggers `UNEXPECTED` remains **OPEN**.

The same exact `2/1/4/4` payload-count invariant was subsequently
confirmed for runtime waveform-drag jobs, demonstrating that this
framing is not specific to the startup sequence.


### 6.4 State propagation

**HW-CONFIRMED:** In the observed sequences:

```text
state_before[n] == final_65[n]
state_after[n]  == state_before[n + 1]
```

This is consistent with a deterministic state machine:

```text
state_before + input
        |
        v
 FPGA job/transform
        |
        +--> variable completion latency
        |
        v
state_after
```

### 6.5 Meaning of `0x66`

**HW-CONFIRMED:** Four stock reads are issued after selecting `0x66`, and all four bytes equal the byte returned by `0x67`.

Examples:

```text
0x67 = 0x15    0x66 reads = 15 15 15 15
0x67 = 0x17    0x66 reads = 17 17 17 17
0x67 = 0xB9    0x66 reads = B9 B9 B9 B9
```

The conservative interpretation is that stock firmware reads the same byte-valued FPGA register four times. Treating this as an independent 32-bit result remains **INFERRED** and is not required for the first driver.

---

## 7. Confirmed startup job sequence

### 7.1 Timing

**HW-CONFIRMED across repeated boots:**

```text
first job begins      approximately 322 ms
last job begins       approximately 814 ms
complete jobs         50
sequence duration     approximately 492 ms
main-loop run_count   0 for every job
```

Therefore this is a pre-main-loop startup sequence, not continuous acquisition traffic.

No additional job was observed during:

```text
30 seconds of idle RUN
STOP and continued waiting
STOP followed by RUN
RUN -> STOP -> RUN in one boot
```

### 7.2 Determinism

**HW-CONFIRMED:** Across independent boots, the following are identical:

```text
0x69 value
0x6A presence
0x6B input sequence
0x64 state_before
0x65 final value
0x67 state_after
0x66 four-byte readback
```

Only the polling count and inter-job timing vary.

This weakens a random per-boot challenge/response interpretation. A deterministic startup transform, self-test, parameter initialization, or FPGA-mediated secure-device operation remains possible.

### 7.3 Corrected startup inputs

**HW-CONFIRMED:** DPOSDBG v4.1 confirmed that `0x6B` carries a full four-byte big-endian value. Earlier three-byte values were truncated by missing tail-branch coverage.

Confirmed examples from the clean startup trace include:

```text
0x00000014 = 20
0x00000040 = 64
0x000003E8 = 1000
0x00000064 = 100
0x000003B6 = 950
0x000000C8 = 200
0x0000003C = 60
0x0000003B = 59
0x454B047C
0x3ED1D1ED
0x000004AF
0xBF809962
```

These supersede the previously documented truncated three-byte values. The integer-looking values strengthen the interpretation that `0x6B` is a numerical 32-bit job input rather than an opaque three-byte payload, but the semantic meaning of each value remains **OPEN**.

### 7.4 Startup phases by timing gaps

Observed grouping:

```text
322..516 ms    primary fast phase
516..583 ms    approximately 67 ms gap
583..598 ms    short second phase
598..715 ms    approximately 117 ms gap
715..747 ms    third phase
747..788 ms    approximately 41 ms gap
788..814 ms    final phase
```

The gaps likely represent other startup work, delays, or dependencies between job groups. Exact callers remain **OPEN**.

---

## 8. Confirmed waveform-drag job sequence

### 8.1 UI causality

**HW-CONFIRMED:** Moving the displayed waveform vertically up and down caused the trace to grow from the clean-boot baseline of 51 records to 191 records.

```text
191 - 51 = 140 additional jobs
140 / 5 = 28 repeated UI-update blocks
```

The earlier 111-record trace is explained by the same structure:

```text
111 - 51 = 60 additional jobs
60 / 5 = 12 UI-update blocks
```

A later DPOSDBG v4.1 re-trace with complete four-byte payload coverage
produced:

```text
261 total records
51 startup records
210 waveform-drag jobs
42 complete five-job UI-update blocks
```

All 210 runtime jobs retained the exact 2/1/4/4 payload-count invariant and the trace completed with wrapped=0 and dropped=0.

The earlier trace was created while touch behavior was being tested.

### 8.2 Five-job state cycle

After the first block connects the startup end state to the runtime cycle, waveform movement repeatedly produces:

```text
B9 -> 0D
0D -> B5
B5 -> 0F
0F -> 11
11 -> B9
```

This is a closed five-job state cycle:

```text
B9 -> 0D -> B5 -> 0F -> 11 -> B9
```

### 8.3 Timing and main-loop correlation

**HW-CONFIRMED:** During continuous drag:

```text
one five-job block starts approximately every 60 ms
block starts normally advance the stock main-loop run_count by one
```

This corresponds to approximately 16.7 update blocks per second and is consistent with touch/UI update cadence.

### 8.4 Waveform-drag input values

**HW-CONFIRMED:** DPOSDBG v4.1 confirmed the full four-byte `0x6B`
values during vertical waveform dragging.

Each UI update produces the previously identified five-job state cycle:

```text
B9 -> 0D    X
0D -> B5    0
B5 -> 0F    X
0F -> 11    X
11 -> B9    0
```

where `X` is the same 32-bit big-endian `0x6B` value in jobs 1, 3, and 4 of the block.

During one upward movement, observed values increased monotonically:

```text
0x00FC = 252
0x011B = 283
0x0141 = 321
0x0147 = 327
0x0150 = 336
0x015D = 349
0x016A = 362
0x0172 = 370
0x0177 = 375
0x017A = 378
0x017C = 380
0x017E = 382
```

During the reverse movement, the values decreased:

```text
0x017A = 378
0x0171 = 369
0x0160 = 352
0x0155 = 341
...
0x0105 = 261
0x0104 = 260
0x0103 = 259
0x0101 = 257
0x0100 = 256
0x00FF = 255
0x00FE = 254
0x00FD = 253
```

The variable X therefore tracks vertical waveform movement monotonically.

The exact coordinate system, scale, offset, and whether X is the direct waveform position or a derived FPGA parameter remain **OPEN**.

The earlier v3 observation 1, 0, 1, 1, 0 is superseded by this result. It represented the incomplete three-byte view of the same pattern:

Plain Text
X, 0, X, X, 0

**HW-CONFIRMED:** During continuous dragging, one five-job block is normally generated approximately every 60 ms and correlates with one increment of the stock main-loop run_count.

## 9. What the job protocol is and is not

### Confirmed

- It is an ARM-visible asynchronous FPGA job/state protocol.
- It is used during a deterministic pre-main-loop startup sequence.
- It is also invoked by waveform UI movement.
- It has a ready bit, variable completion latency, deterministic state transitions, and a confirmed four-byte big-endian `0x6B` input field in complete startup jobs.

### Not confirmed

- That commands `0x64..0x6B` are direct physical commands of the external eight-pin device.
- That every job communicates with the SMEC88SP-class candidate.
- That the path is exclusively an authentication mechanism.
- That the path is exclusively FFT, FRA, or frequency measurement.
- That the observed five-job waveform cycle is required for raw ADC acquisition.

### Current neutral model

```text
ARM UI / startup logic
        |
        | 0x64..0x6B jobs
        v
FPGA job / transform engine
        |
        +--> internal calculation or measurement logic
        |
        +--> optional external secure-device service
        |
        v
state/result returned to ARM
```

---

## 10. CSI sample transport

The DPOS350P does not transfer its bulk waveform samples through repeated reads of the eight-bit control bus. Samples are streamed from the FPGA into DRAM through the R11 CSI1 peripheral.

### 10.1 Confirmed addresses and layout

```text
CSI1 base              0x01CB4000
CSI DMA target         0x01CB4050
local ring base        0x42DF76D4
slot stride            0x2134 = 8500 bytes
slot count             8
active payload         0x1800 = 6144 bytes per slot
```

- CSI and logical ring placement: **STATIC-CONFIRMED locally**.
- Active 6144-byte hardware payload and channel grouping: **EXTERNAL HW-CONFIRMED**.

The DMA target register stores the buffer address in four-byte units.

### 10.2 Channel grouping

**EXTERNAL HW-CONFIRMED:**

```text
CH1-only acquisition:
    slots 0..7 contain CH1 data

CH1 + CH2 acquisition:
    slots 0..3 contain CH1 data
    slots 4..7 contain CH2 data
```

Each active slot contains 6144 raw eight-bit sample codes.

### 10.3 Per-slot offsets

Different slots show different zero-code offsets. One external observation included approximately:

```text
166, 162, 151, 151
```

This is consistent with several ADC/interleave lanes requiring per-slot calibration. Exact lane order is **OPEN**.

### 10.4 Unused logical-slot area

```text
8500 - 6144 = 2356 bytes
```

The meaning of the remaining 2356 bytes is **OPEN**. Do not label the area as simple padding until verified. Possible uses include allocation headroom, ring-origin handling, guards, alignment, or mode-dependent data.

---

## 11. Acquisition status and control: current boundary

The following are established:

- The FPGA is configured before the Custom App begins.
- The diagnostic identity transaction works under the custom boot chain.
- CSI1 is the bulk sample path.
- The ring contains eight logical slots with 6144 active bytes each.
- Stock acquisition is gated by a run/stop field and checks FPGA status.

The following remain necessary before the first reliable standalone capture:

1. Exact minimum FPGA acquisition command sequence.
2. Exact CSI start, completion, and interrupt/status handling.
3. Relationship between FPGA command `0x22`, CSI active length, and acquisition mode.
4. Initial channel, timebase, and trigger values required for a safe fixed profile.
5. Whether any part of the 50-job startup transform is required before standalone acquisition in the custom boot path.

Current evidence does **not** support blindly copying all 50 startup jobs into the Custom App. They may represent POST, parameter initialization, calculation-engine setup, or secure-device interaction rather than the minimum capture path.

---

## 12. Recommended driver architecture

### 12.1 Physical bus layer

```text
drivers/fpga_bus.c
drivers/fpga_bus.h
```

Responsibilities:

- GPIO direction and pin mux.
- Selector, read, and write strobes.
- Byte transactions.
- No knowledge of channels, timebase, CSI, or UI.

### 12.2 DPOS350P FPGA protocol layer

```text
drivers/dpos_fpga.c
drivers/dpos_fpga.h
```

Responsibilities:

- FPGA alive check.
- Named, confirmed setting commands.
- Neutral helpers for unknown acquisition commands.
- Timeout-bounded asynchronous job submission.

Suggested job API:

```c
struct dpos_fpga_job {
    uint8_t mode_69;
    uint8_t input[4];
    uint8_t input_size;
};

struct dpos_fpga_job_result {
    uint8_t state_before;
    uint8_t final_65;
    uint8_t state_after;
    uint8_t result[4];
    uint8_t result_size;
    uint32_t poll_count;
};

int dpos_fpga_submit_job(const struct dpos_fpga_job *job,
                         struct dpos_fpga_job_result *result,
                         uint32_t timeout);
```

### 12.3 CSI layer

```text
drivers/r11_csi.c
drivers/r11_csi.h
```

Responsibilities:

- CSI register programming.
- DMA destination programming.
- Start/stop.
- Completion polling with timeout.
- Status and error reporting.

### 12.4 Acquisition orchestration layer

```text
drivers/dpos_acq.c
drivers/dpos_acq.h
```

Responsibilities:

- Apply a fixed initial acquisition profile.
- Coordinate FPGA commands and CSI setup.
- Start one capture.
- Wait with a finite timeout.
- Return slot pointers and active lengths.
- Never depend on stock EXE2 RAM objects.

---

## 13. First standalone acquisition milestone

The first Custom App milestone is deliberately smaller than a complete oscilloscope driver.

### Fixed initial profile

```text
CH1 only
DC coupling
fixed safe V/div
fixed safe timebase
AUTO or fixed trigger
8 logical slots
6144 active bytes per slot
timeouts on every wait
```

### Required proof

```text
FPGA alive check          PASS
CSI configuration         PASS
capture start             PASS
capture completion        PASS or bounded TIMEOUT
slot data changes         PASS
min/max/checksum reported PASS
```

A recognizable calibrated waveform is not required for the first proof. Initial success is:

- non-static data,
- repeatable response to a known input signal,
- changed periodicity or checksum when the source changes,
- no infinite waits or loss of recovery capability.

---

## 14. Safe-test rules

```text
Always use finite timeouts.
Begin with one fixed profile.
Change one variable per experiment.
Do not enable every FPGA subsystem at once.
Display command/status/error values on failure.
Keep DEBUG EXE2 FROM FAT and USB recovery available.
Do not write the FPGA configuration flash.
Do not assume old FNIRSI command meanings apply to DPOS350P.
Do not promote an inferred command name to the driver API without evidence.
```

---

## 15. Next protocol experiments

Priority order:

1. Trace the exact stock FPGA command sequence around one capture start.
2. Trace one controlled timebase change.
3. Trace one V/div change without dragging.
4. Trace STOP, RUN, and SINGLE acquisition paths at selected call sites.
5. Correlate FPGA status command `0x0A` with CSI completion.
6. Reproduce the minimum sequence in a timeout-bounded Custom App smoke test.
7. Determine lane/interleave order only after raw capture succeeds.

The `0x64..0x6B` job protocol is now sufficiently documented for a generic helper, but it should not block the first raw-acquisition attempt unless focused tracing proves a direct dependency.

---

## 16. Relationship to older FNIRSI protocols

The FNIRSI-1013D/1014D family documents a similar eight-bit FPGA control bus and a `0x64..0x6E` parameter-chip command family. This is useful as architectural precedent only.

Important differences include:

- The older design reads bulk channel samples through FPGA commands such as `0x20` and `0x22`.
- The DPOS350P streams bulk samples through CSI1 DMA.
- The observed DPOS350P `0x64..0x6B` framing and state behavior do not map one-to-one to the older command descriptions.
- Shared command numbers are not proof of shared semantics.

Use the older protocol to generate hypotheses, never as direct DPOS350P implementation truth.

---

## 17. Maintenance rule

When a new finding is made:

1. Put experiment details, traces, disassembly, and reasoning in `docs/FPGA.md`.
2. Promote only the stable implementation consequence into this document.
3. Keep the evidence label beside the claim.
4. Update driver constants only after the protocol reference is updated.
5. If evidence changes, preserve the history in `docs/FPGA.md` and correct this file without retaining obsolete implementation guidance.

This separation keeps the complete reverse-engineering record while giving the Custom App a concise, reliable specification.
