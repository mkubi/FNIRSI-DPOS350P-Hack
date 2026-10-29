# SMEC reverse engineering notes

Status: consolidated baseline, 2026-10-26

This document records the current SMEC/security-chip and related FPGA service-interface reverse engineering for the FNIRSI DPOS350P stock EXE2 (`exe2-from-backup.bin`, runtime base `0x40000000`). It intentionally separates confirmed facts from interpretation. The primary target of this revision is a complete reconstruction of `SMEC_Init()` sufficient for standalone reproduction.

## Evidence labels

- **STATIC-CONFIRMED**: directly established from stock EXE2 instructions and call flow.
- **HW-CONFIRMED**: directly observed on the real device.
- **DERIVED**: follows deterministically from confirmed instructions/data, but is a higher-level description.
- **INTERPRETATION**: semantic meaning suggested by structure or behavior, but not yet authoritative.
- **OPEN**: not yet known.

---

## 1. Confirmed symbol map relevant to SMEC

```text
main                         0x40034700
FPGA_Start                   0x40049188
SMEC_Init                    0x4004A15C
PowerInit                    0x4004A198
HorizontalInit               0x4003A62C
VerticalInit                 0x4004A50C
HardCostTimeInfoInit         0x4003EF1C
Gen_Init                     0x4004D164
SysConfigExec                0x4003E708

FPGA read byte               0x400491B8
FPGA write data              0x40049214
FPGA write command/selector  0x4004927C
FPGA ID probe                0x400492E4
```

The startup ordering in `main` confirms that SMEC/front-end initialization occurs after `FPGA_Start()` and before horizontal/vertical acquisition initialization.

Relevant startup sequence:

```text
FPGA_Start             0x40049188
...
SMEC_Init              0x4004A15C
PowerInit              0x4004A198
HorizontalInit         0x4003A62C
VerticalInit           0x4004A50C
HardCostTimeInfoInit   0x4003EF1C
...
Gen_Init               0x4004D164
...
SysConfigExec          0x4003E708
```

`SysConfigExec()` is later than `SMEC_Init()`. Therefore the seven early SMEC startup job values already exist before this later configuration pass.

---

## 2. Why SMEC became a priority

Standalone acquisition experiments reproduced important FPGA control-plane behavior:

```text
FPGA ID               valid
FPGA 0x0A READY       asserted
FPGA 0x18 VALID       asserted
FPGA 0x14 POSITION    valid, observed 0x0101
MS5351M clocks         reproduced
CSI setup              reproduced to the tested level
PE1 stream gate        exercised
```

However, a direct Stage-13 GPIO activity probe sampled the expected physical sample interface for 4,000,000 iterations and observed no transitions on:

```text
PE0       expected sample PCLK
PE2/PE3   expected synchronization signals
PE8..PE15 expected sample data bus
```

Therefore a valid FPGA command/control state does **not** imply an active physical sample stream. The stock firmware performs `SMEC_Init()`, `PowerInit()`, `HorizontalInit()` and `VerticalInit()` after `FPGA_Start()`, while the standalone acquisition experiment did not reproduce the complete sequence.

**INTERPRETATION:** SMEC/front-end initialization is a leading candidate for a prerequisite of the physical sample producer. It is not yet proven that `SMEC_Init()` alone activates the stream.

---

## 3. SMEC_Init reconstruction

### 3.1 Confirmed minimal platform prerequisite

**HW-CONFIRMED:** DPOSLAB minimal-runtime A/B testing established a
platform prerequisite that must be satisfied before the stock `FPGA_Start()`
path can complete.

The beginning of stock `main()` is:

    0x40034708  BL 0x4004282C   sys_clock_init(1152000000)
    0x4003470C  BL 0x40042A60   sys_mmu_init()
    0x40034710  BL 0x40042D28   System_Init()
    ...
    0x40034738  BL 0x40049188   FPGA_Start()
    ...
    0x40034750  BL 0x4004A15C   SMEC_Init()

DPOSLAB was used to intercept stock execution at `0x40034710`, after
`sys_clock_init()` and `sys_mmu_init()` but before `System_Init()`.
This allowed the platform prerequisites to be tested independently of the
remaining application initialization.

The following normal startup operations were excluded from the test:

    MemoryType setup
    AllInfoInit
    SPI flash initialization
    GetSysSavedInfo
    LanguageInit
    GUI initialization
    touch initialization
    FAT initialization
    remaining stock main()

A hook-only control at `0x40034710` completed successfully, proving that
stock execution reaches this point after `sys_clock_init()` and
`sys_mmu_init()` and that the resident DPOSLAB environment remains usable.

Static analysis decomposes `System_Init()` as:

    System_Init @ 0x40042D28
        |
        +-- BL 0x40042C94     Part 1
        |
        `-- B  0x40042CD8     Part 2

Minimal-runtime testing then isolated the two components.

With Part 1 skipped and Part 2 executed:

    sys_clock_init             PASS
    sys_mmu_init               PASS
    System_Init Part 1         SKIPPED
    System_Init Part 2         PASS
    stock FPGA_Start           PASS
    stock Job 1                PASS
    stock Job 2                PASS

The observed stock transaction results were:

    Job 1, index 1, value 20:
        result66 = 0x01010101

    Job 2, index 2, value 64:
        result66 = 0x03030303

A direct negative-control build was then produced from this known-good
configuration. The only functional change was removal of the call to:

    0x40042CD8

With Part 2 disabled, the subsequent stock `FPGA_Start()` path no longer
completed. The separate hook-only control had already established that
execution itself reaches the interception point correctly.

**HW-CONFIRMED conclusion:**

    sys_clock_init
        -> sys_mmu_init
        -> System_Init Part 2 @ 0x40042CD8
        -> FPGA_Start

is the current minimum confirmed platform chain for the tested stock
`FPGA_Start()` path.

`System_Init Part 1 @ 0x40042C94` is not required for this minimal path.

Static analysis of Part 2 shows:

    0x40042CD8
        -> 0x40042D40
        -> common implementation 0x40042D54

    0x40042CD8
        -> 0x40042D4C
        -> common implementation 0x40042D54

The implementation operates on MMIO registers including:

    0x01C20144
    0x01C20C8C

and uses the values:

    24000
    24

**INTERPRETATION:** the structure strongly suggests initialization of two
timer/timebase resources. At a 24 MHz source clock, the constants are
consistent with millisecond- and microsecond-scale timebases, but the exact
peripheral semantics and official function names are not yet confirmed.

This prerequisite is directly relevant to standalone reproduction.
A standalone `FPGA_Start()` clone should not be considered equivalent to the
stock startup path unless the required Part 2 platform state has already been
established.

## 3.2 Retry/timing table initialization

`0x40049F44` fills 255 32-bit entries with decimal `2000`:

```c
for (i = 0; i < 255; ++i)
    retry_or_timing[i] = 2000;
```

The exact official semantic name is unknown. Its later use in retry logic supports a retry/timing/settling interpretation.

---

## 3.3 Direct FPGA setup inside SMEC_Init

Helper `0x4004A13C` performs exactly:

```text
FPGA selector 0x67
FPGA payload  0x08
```

Equivalent confirmed operation:

```text
FPGA 0x67 <- 0x08
```

Important: selector `0x67` is later also **read** during SMEC job completion checking. It must therefore not yet be named a stream-enable register. Read and write semantics may differ, or the register may contain both configuration and status fields.

---

## 3.4 Baseline selector 0x69 value

`SMEC_Init()` calls `0x40049FB8(2000)`.

`0x40049FB8` writes a 16-bit value through FPGA selector `0x69`, big-endian. Decimal `2000` is `0x07D0`:

```text
selector 0x69
payload  07 D0
```

---

# 4. SMEC startup jobs 1..7

## 4.1 Runtime source block

`0x40049E30` reads seven startup parameters from the stock runtime state rooted at:

```text
0x40544E68
```

The unusual ordering of jobs 3..5 is present in stock code and must be preserved exactly.

```text
job 1  size 1  state +0x1B8  source 0x40545020
job 2  size 4  state +0x1C0  source 0x40545028
job 3  size 4  state +0x268  source 0x405450D0
job 4  size 4  state +0x270  source 0x405450D8
job 5  size 4  state +0x26C  source 0x405450D4
job 6  size 2  state +0x008  source 0x40544E70
job 7  size 4  state +0x62C  source 0x40545494
```

## 4.2 Hardware-confirmed startup values

A DPOSDBG ABI-v2 manifest hook was installed at the confirmed `main -> SMEC_Init` call site:

```text
call site       0x40034750
original word   0xEB005681
SMEC_Init       0x4004A15C
```

The wrapper called the original `SMEC_Init()` and captured source values, cache values and initialized flags immediately after return, before `PowerInit`, `HorizontalInit` or `VerticalInit` could alter the state.

Three independent cold boots produced bit-identical results:

```text
job 1  u8   = 0x00000014 =   20
job 2  u32  = 0x00000040 =   64
job 3  u32  = 0x00000040 =   64
job 4  u32  = 0x000003E8 = 1000
job 5  u32  = 0x00000064 =  100
job 6  u16  = 0x000003B6 =  950
job 7  u32  = 0x000000C8 =  200
```

For every job on every tested cold boot:

```text
source == cached
initialized == 1
```

That is 21/21 matching source/cache checks across the three cold boots.

The snapshot also reported:

```text
captured        = 1
job_count       = 7
any_initialized = 1
chip_status_60  = 0
```

**HW-CONFIRMED conclusion:** the stable cold-boot baseline for `SMEC_Init()` on the tested DPOS350P is:

```text
{ 20, 64, 64, 1000, 100, 950, 200 }
```

This proves stability across the three tested cold boots. It does **not** yet prove that the values are compile-time constants. They could still originate from stable defaults, persistent configuration, calibration or a deterministic calculation.

---

## 4.3 SMEC cached values and initialized flags

The cached normalized parameter values live at:

```text
cache root = 0x42E62C2C

cached_value[i] = 0x42E63028 + 4*i
initialized[i]  = 0x42E63424 + i
```

For startup jobs 1..7:

```text
job 1  cache 0x42E6302C  flag 0x42E63425
job 2  cache 0x42E63030  flag 0x42E63426
job 3  cache 0x42E63034  flag 0x42E63427
job 4  cache 0x42E63038  flag 0x42E63428
job 5  cache 0x42E6303C  flag 0x42E63429
job 6  cache 0x42E63040  flag 0x42E6342A
job 7  cache 0x42E63044  flag 0x42E6342B
```

A global byte indicating that at least one parameter has been initialized is:

```text
0x42E63523  = cache root + 0x8F7
```

The cached encryption/security-chip POST status is:

```text
0x42E63524  = cache root + 0x8F8
```

---

## 4.4 Cached submission behavior

`0x40049EC4(index, size, value_low_or_ptr, value)` performs parameter normalization and change detection.

For parameter sizes below 4 bytes, the normalized value is taken from the value passed in `r3`. For 32-bit parameters it uses the value passed in `r2`. This explains duplicated argument values at some stock call sites.

The effective logic is:

```c
if (cached_value[index] != normalized_value || initialized[index] == 0) {
    initialized[index] = 1;
    cached_value[index] = normalized_value;
    smec_job(index, normalized_value);
    any_parameter_initialized = 1;
}
```

Thus normal runtime code does not resend a job unless that parameter is new or changed.

At first `SMEC_Init()`, jobs `1..7` traverse this path and become initialized.

---

# 5. SMEC job transport protocol

## 5.1 Confirmed helper roles

The active SMEC job engine is implemented around `0x4004A0B0` and the helper functions in `0x40049Fxx..0x4004A0xx`.

Confirmed selectors:

```text
0x69  16-bit retry/timing/settling parameter
0x6A  one-byte job index
0x6B  32-bit job value
0x68  execute/control write; later read for returned job/index status
0x64  polled service status
0x65  completion/status polling
0x67  status bit read; also receives 0x08 during SMEC_Init
0x66  32-bit result
```

Confirmed helper encodings:

```text
0x40049F60  selector 0x6A, one-byte job index
0x40049F7C  selector 0x6B, 32-bit job value, big-endian
0x40049FB8  selector 0x69, 16-bit timing/retry value, big-endian
```

For a job `(index=i, value=v, timing=t)`:

```text
69  t[15:8]  t[7:0]
6A  i
6B  v[31:24] v[23:16] v[15:8] v[7:0]
```

Although jobs 1 and 6 originate from `u8` and `u16` source fields, the normalized job value sent through `0x6B` is still a 32-bit big-endian payload.

---

## 5.2 Exact startup payloads

With the HW-confirmed cold-boot baseline, the seven startup payloads are:

```text
job 1:
6A 01
6B 00 00 00 14

job 2:
6A 02
6B 00 00 00 40

job 3:
6A 03
6B 00 00 00 40

job 4:
6A 04
6B 00 00 03 E8

job 5:
6A 05
6B 00 00 00 64

job 6:
6A 06
6B 00 00 03 B6

job 7:
6A 07
6B 00 00 00 C8
```

The baseline selector `0x69` parameter is:

```text
69 07 D0
```

before job execution/retries.

---

## 5.3 Job lifecycle

The job transaction shape reconstructed from stock code is:

```text
write selector 0x69 timing/retry parameter
write selector 0x6A job index
write selector 0x6B 32-bit job value
write selector 0x68 = 1
poll/read service status through 0x64
write selector 0x68 = 0
poll completion through 0x65
read selector 0x68
read selector 0x67 bit 0
read 32-bit result through selector 0x66 on success
```

The important success gate is:

```text
read 0x68 == requested job index
AND
read 0x67 bit 0 != 0
```

Only then does the code proceed to read the 32-bit result via selector `0x66`.

---

## 5.4 Retry behavior

Each per-index timing/retry entry starts at:

```text
2000 decimal = 0x07D0
```

On an unsuccessful attempt:

```text
value += 100
```

The inner retry window continues through:

```text
3000 decimal = 0x0BB8
```

After exhausting that window, the stored per-index value is restored/set to `3000`, an outer retry counter increments, and the operation retries. The outer retry path is bounded at four attempts.

This is consistent with a robust service/handshake protocol rather than a simple FPGA register write.

---

# 6. Encryption/security-chip POST check

The stock UTF-8 POST label:

```text
加密芯片检测 ...
```

means an encryption/security-chip check and is stored at:

```text
0x4008E304
```

A pointer to the label is present at:

```text
0x40051D98
```

The code at `0x400510D8` loads and displays this label, updates the display, then calls the actual check:

```text
0x400510D8  load label pointer
0x400510E4  display label helper
0x400510E8  display/present helper
0x400510EC  call 0x4004F5C0
```

`0x4004F5C0` calls:

```text
0x40049E00
```

`0x40049E00` performs:

```text
SMEC/pending-state helper 0x40049F1C
FPGA selector 0x60
FPGA read byte
mask bit 0
store result at 0x42E63524
```

Confirmed chain:

```text
"加密芯片检测 ..."
    -> 0x4004F5C0
    -> 0x40049E00
    -> read FPGA selector 0x60 bit 0
    -> POST result
```

Static xref analysis found only one direct caller of `0x40049E00` in stock EXE2:

```text
0x4004F5C8 -> 0x40049E00
```

Therefore `0x40049E00` is not currently evidenced as a periodic runtime health-check helper. Its confirmed use is the explicit encryption/security-chip POST test.

Stock POST also distinguishes FPGA and encryption-chip checks:

```text
FPGA ID/check result       separate POST result field
security-chip 0x60 check   separate POST result field
```

**DERIVED:** stock firmware treats the FPGA and the encryption/security-chip status as distinct entities even though the security-chip status is exposed through an FPGA selector.

`chip_status_60 = 0` in the immediate post-`SMEC_Init()` snapshots is not evidence of failure. The snapshot is taken before the separate POST helper `0x40049E00` updates the cached `0x42E63524` byte.

---

# 7. Additional SMEC job namespace

The same cached submitter `0x40049EC4` is used well beyond `SMEC_Init()`.

Confirmed groups include:

```text
0x01..0x07  SMEC_Init startup baseline

0x08,0x0A,0x0C  one 32-bit grouped path
0x09,0x0B,0x0D  paired 32-bit grouped path

0x14..0x18       Gen_Init group

0x1E..0x20       later byte-sized runtime group

0x21..0x23       runtime group paired with FPGA selectors 0x04..0x06
```

The `0x08..0x0D` paths are interleaved with direct FPGA writes and service commands `0xB5` / `0xB6`.

`Gen_Init = 0x4004D164` submits jobs:

```text
0x14
0x15
0x16
0x17
0x18
```

Some `Gen_Init` job values are produced from floating-point-to-integer conversions.

A later byte-sized group shows explicit SMEC/FPGA pairing:

```text
SMEC job 0x1E -> FPGA selector 0xAB
SMEC job 0x1F -> FPGA selector 0xAC
SMEC job 0x20 -> FPGA selector 0xAD
```

with service commands:

```text
0xBE
0xBF
0xC0
```

Another runtime group pairs:

```text
SMEC job 0x21 -> FPGA selector 0x04
SMEC job 0x22 -> FPGA selector 0x05
SMEC job 0x23 -> FPGA selector 0x06
```

**INTERPRETATION:** the SMEC job engine coordinates physical/front-end state with direct FPGA digital/control state. Exact physical meanings of individual jobs remain open.

---

# 8. Runtime trace evidence

The established DPOSDBG v5 trace records complete SMEC transactions using the FPGA primitive wrappers.

A later real-device test in RUN mode, after moving the displayed waveform vertically before taking a screenshot, preserved:

```text
110 complete SMEC job records
658 CSI destination records
```

Complete SMEC records carried flags:

```text
0x09FF
```

representing the full observed lifecycle (`0x69`, `0x6A`, `0x6B`, START, `0x64`, `0x65`, `0x67`, `0x66`, COMPLETE).

The preserved runtime jobs included changing values in later job groups, supporting the conclusion that the SMEC engine is active after startup and participates in runtime configuration.

The same trace preserved eight cyclic stock CSI destination addresses:

```text
0x42DF76D4
0x42DF9808
0x42DFB93C
0x42DFDA70
0x42DFFBA4
0x42E01CD8
0x42E03E0C
0x42E05F40
```

with a spacing of:

```text
0x2134 = 8500 bytes
```

This is not part of `SMEC_Init()` itself, but shows that the hooked stock acquisition path continues to run normally while SMEC runtime jobs are observed.

---

# 9. PowerInit side analysis

**CONFIRMED symbol:**

```text
PowerInit = 0x4004A198
```

Confirmed call shape:

```text
PowerInit
  -> 0x40068E94
  -> 0x4004A1D0
  -> tail 0x4004A2A0
```

PowerInit contains no direct calls to the confirmed FPGA read/write/selector primitives and no direct submission through `0x40049EC4`.

`0x40068E94` performs read-modify-write operations on:

```text
0x01C22800
```

establishing:

```text
bits [13:12] = 2
bits [5:4]   = 0
```

`0x4004A1D0` computes a future state value in the form:

```c
state[0x65C] = *(uint32_t *)0x42C2FE74 + 60000 * state[0x658];
```

The tail path writes raw constants:

```text
0x42E68C9C <- 0x432A0000
0x42E68C94 <- 0x4203CA1B
0x42E68C98 <- 0x45205BCA
```

As IEEE-754 single-precision values they correspond approximately to:

```text
170.0
32.947...
2565.737...
```

Only the raw values and destinations are confirmed. The higher-level power/battery interpretation remains unconfirmed.

**Current relevance:** PowerInit is a weaker direct candidate for FPGA stream activation than SMEC/Horizontal/Vertical initialization because it performs no direct known FPGA or SMEC job transaction.

---

# 10. Reconstructed SMEC_Init pseudocode

The following is intended as a behaviorally useful reconstruction. Helper names are descriptive, not original symbols.

```c
void SMEC_Init(void)
{
    platform_or_device_init(/* ... */, 0x08FC);

    for (unsigned i = 0; i < 255; ++i)
        smec_retry_or_timing[i] = 2000;

    fpga_select(0x67);
    fpga_write(0x08);

    fpga_write_u16_be(0x69, 2000);

    smec_submit_cached(1, 1, 20);
    smec_submit_cached(2, 4, 64);
    smec_submit_cached(3, 4, 64);
    smec_submit_cached(4, 4, 1000);
    smec_submit_cached(5, 4, 100);
    smec_submit_cached(6, 2, 950);
    smec_submit_cached(7, 4, 200);
}
```

The submitter behaves approximately as:

```c
void smec_submit_cached(unsigned index, unsigned size, uint32_t value)
{
    uint32_t normalized = value;

    if (!initialized[index] || cached[index] != normalized) {
        initialized[index] = 1;
        cached[index] = normalized;
        smec_job(index, normalized);
        any_parameter_initialized = 1;
    }
}
```

And the transaction engine is approximately:

```c
uint32_t smec_job(uint8_t index, uint32_t value)
{
    for (unsigned outer = 0; outer < 4; ++outer) {
        do {
            fpga_write_u16_be(0x69, retry[index]);
            fpga_write_u8(0x6A, index);
            fpga_write_u32_be(0x6B, value);

            fpga_write_u8(0x68, 1);
            /* service/status interaction through 0x64 */
            fpga_write_u8(0x68, 0);
            /* completion polling through 0x65 */

            uint8_t returned_index = fpga_read_u8(0x68);
            uint8_t status67 = fpga_read_u8(0x67) & 1;

            if (returned_index == index && status67)
                return fpga_read_u32_be(0x66);

            retry[index] += 100;
        } while (retry[index] <= 3000);

        retry[index] = 3000;
    }

    return 0;
}
```

The pseudocode deliberately abstracts some detailed `0x64/0x65` polling internals. The confirmed selectors, payloads, comparison gates and retry limits are retained.

---

# 11. Current status of SMEC_Init analysis

`SMEC_Init()` is now reconstructed to a level sufficient for a standalone behavioral reproduction on the tested unit.

**Confirmed or hardware-confirmed:**

```text
entry address and startup position
major call chain
direct FPGA 0x67 <- 0x08
255-entry baseline table = 2000
selector 0x69 baseline = 2000
job transport selectors 0x64..0x6B
big-endian payload encoding
job cache layout
initialized flag layout
change-detection behavior
retry progression 2000 -> 3000
bounded outer retry path
success gate using returned index and 0x67 bit 0
startup jobs 1..7
source addresses
source sizes
cache addresses
initialized flags
startup values {20,64,64,1000,100,950,200}
three cold boots with identical values
21/21 source/cache matches across those cold boots
System_Init Part 2 @ 0x40042CD8 is a prerequisite for the tested minimal
stock FPGA_Start path
System_Init Part 1 @ 0x40042C94 is not required for that path
minimal stock FPGA_Start + Job 1 + Job 2 succeeds without normal
application initialization
Job 1 result66 = 0x01010101 in the isolated stock test
Job 2 result66 = 0x03030303 in the isolated stock test
```

**Still open:**

1. authoritative physical meaning/name of each job `1..7`;
2. whether the seven stable values are compile-time defaults, persistent configuration, calibration values, or deterministic derived values;
3. whether changing persistent oscilloscope configuration before a cold boot changes any of the seven startup values;
4. whether `SMEC_Init()` itself activates PE sample output, or whether later `HorizontalInit()` / `VerticalInit()` is also required;
5. exact identity and electrical topology of the external encryption/security/front-end device behind the FPGA-mediated interface.
6. exact peripheral/timer identity configured by System_Init Part 2;
7. whether both Part 2 subpaths 0x40042D40 and 0x40042D4C are individually required;
8. which exact FPGA_Start subroutine depends on the Part 2 initialization;
9. whether reproducing Part 2 in the Custom App resolves the current Job 2 failure.

---

# 12. Planned validation experiments

## 12.1 Configuration-dependence test for jobs 1..7

The three identical cold boots establish a stable baseline. A final useful control is:

1. significantly change stock configuration, e.g. timebase, V/div and vertical position;
2. ensure the configuration is persisted if required by stock firmware;
3. power-cycle the unit;
4. capture the immediate post-`SMEC_Init()` snapshot again;
5. compare with:

```text
20, 64, 64, 1000, 100, 950, 200
```

If the values remain identical, the evidence for a fixed/device-stable startup baseline becomes substantially stronger. A changed value would identify a parameter that must not be blindly hardcoded in a general implementation.

## 12.2 A/B/C/D/E physical-stream checkpoint test

Once hardware testing resumes, measure physical PE activity at confirmed startup boundaries:

```text
A  immediately after FPGA_Start
B  immediately after SMEC_Init
C  immediately after PowerInit
D  immediately after HorizontalInit
E  immediately after VerticalInit
```

At each checkpoint, perform a bounded **read-only** activity window over:

```text
PE0       expected PCLK
PE2/PE3   expected sync
PE8..PE15 expected sample data
```

Record at least:

```text
first value
last value
changed mask
ever-high mask
ever-low mask
sample count
```

A no-activity -> activity transition between adjacent checkpoints will identify the startup stage that first enables the physical FPGA sample-output datapath.

This test does not require CSI or RAM DMA and therefore cleanly separates sample-producer activation from CSI receiver configuration.

---

# 13. DPOSDBG infrastructure status relevant to future SMEC work

The debug infrastructure now uses a hardware-confirmed ABI-v2 manifest model:

```text
DPOSDBG.BIN
  -> carries hook descriptors
  -> patch site
  -> expected stock word
  -> veneer address
  -> wrapper target
  -> patch type B / BL

Stage 2
  -> generically validates manifest
  -> validates stock words
  -> writes veneers and patches
  -> performs cache maintenance
  -> validates patch readback
```

The manifest baseline was confirmed on hardware with stock EXE2 boot, screenshot export and `DPOSDBG.TRC` generation. Stage 2 diagnostics expose loader phase, ABI, hook count, hook index, B/BL type, site, expected/actual word, veneer, target and generated branch information.

This allows future SMEC/Horizontal/Vertical checkpoint hooks to be added by replacing `DPOSDBG.BIN` without repeatedly changing the Stage 2 loader, as long as the existing ABI-v2 B/BL descriptor types are sufficient.

---

# 14. Bottom line

The original working question was whether the missing standalone sample stream could be explained by an initialization layer after `FPGA_Start()`.

The current evidence establishes that:

```text
FPGA_Start alone is not the whole stock acquisition bring-up.

SMEC_Init performs a substantial FPGA-mediated service protocol:
  FPGA 0x67 <- 0x08
  timing/retry baseline 2000
  seven cached startup jobs
  robust execute/status/result/retry lifecycle

The seven real-device cold-boot values are:
  20, 64, 64, 1000, 100, 950, 200
```

A further prerequisite was isolated before `FPGA_Start()` itself:
`System_Init` Part 2 at `0x40042CD8` is required for the tested minimal
stock `FPGA_Start()` path to complete, while Part 1 at `0x40042C94` can
be omitted.

The current minimum confirmed stock chain is therefore:

    sys_clock_init
      -> sys_mmu_init
      -> System_Init Part 2
      -> FPGA_Start
      -> SMEC startup transactions

Three independent cold boots reproduced these values exactly, with every source field matching the corresponding SMEC cache and every startup initialized flag asserted.

Therefore `SMEC_Init()` should no longer be treated as an unknown black box. Its behavior is sufficiently reconstructed for a careful standalone reproduction on the tested device.

What is **not** proved yet is causality between `SMEC_Init()` and physical sample-stream activation. The next decisive hardware experiment is the A/B/C/D/E PE activity checkpoint across `FPGA_Start`, `SMEC_Init`, `PowerInit`, `HorizontalInit` and `VerticalInit`.

## 15. 2026-10-29 standalone custom SMEC milestone

The earlier open question about whether the reconstructed SMEC path is sufficient for a real acquisition can now be closed for the currently supported fixed profile.

### 15.1 Current code-level implementation

The standalone implementation now uses the project-level names below:

```text
stock SMEC_Init / required S4 path  -> smec_init()
SMEC cached/job submission          -> custom SMEC job implementation
FPGA profile application            -> profile_init()
acquisition readiness               -> acq_prepare()
stream + CSI transfer               -> acq_capture()
```

Historical stock addresses remain useful RE anchors, but current custom-runtime descriptions should prefer these code names where the mapping is established.

### 15.2 Minimal SMEC requirement confirmed by substitution

Hardware substitution testing reduced the stock SMEC startup path to the S4/job path:

```text
S1  NOT REQUIRED for current fixed-profile digital oracle
S2  NOT REQUIRED for current fixed-profile digital oracle
S3  NOT REQUIRED for current fixed-profile digital oracle
S4  REQUIRED in the tested minimal runtime
```

The custom S4 implementation completes all seven startup jobs and is sufficient for the current acquisition baseline.

**HW-CONFIRMED:** custom minimal SMEC passes in the standalone runtime and no stock SMEC implementation is executed by the confirmed fixed-profile capture path.

This result does not establish that SMEC itself is optional. The successful standalone path includes `smec_init()`, and the current evidence supports retaining SMEC as part of the FPGA/front-end communication/configuration protocol.

### 15.3 System_Init Part 2 is now identified as AVS timebase initialization

The older description of `System_Init` Part 2 as an unidentified timer/timebase prerequisite can now be made more precise.

Static analysis of stock `System_Init` @ `0x40042D28` established:

```text
System_Init              0x40042D28
  Part 1                 0x40042C94
  Part 2 / AVS init      0x40042CD8
    channel 0 setup      0x40042D40  divisor 24000
    channel 1 setup      0x40042D4C  divisor 24
    common setup         0x40042D54
```

The common setup sets bit 31 at `0x01C20144` and programs the two 16-bit fields of `0x01C20C8C`:

```text
0x01C20144 |= 0x80000000
AVS channel 0 divisor = 24000
AVS channel 1 divisor = 24
0x01C20C8C = 0x00175DBF
```

The stock AVS delay path for counter 0 then clears `0x01C20C84`, sets bit 0 in `0x01C20C80`, and waits until `(counter0 << 1)` reaches the requested delay units.

This behavior matches the custom `timer_avs_delay_units()` implementation. The standalone failure was caused by the missing AVS initialization, not by the delay primitive itself.

**STATIC-CONFIRMED:** Part 2 is the AVS timebase initialization used by the relevant stock delay path.

**HW-CONFIRMED:** after reproducing this AVS setup in the custom timer initialization, `profile_init()` completes all seven fixed-profile steps instead of stalling in `timer_avs_delay_units(2)`.

### 15.4 End-to-end standalone acquisition proof

The complete standalone chain now succeeds:

```text
custom Timer0 + AVS initialization
        -> custom FPGA initialization
        -> smec_init()
        -> profile_init()
        -> acq_capture()
        -> CSI HW DONE
        -> full 0x1800-byte RAM payload
```

Hardware result:

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

**HW-CONFIRMED:** the reconstructed custom SMEC path is sufficient as part of a fully custom fixed-profile acquisition runtime. The proof extends beyond FPGA READY: the CSI transfer completes and exactly the complete expected `0x1800`-byte payload range is updated in RAM.

Consequently, the earlier concern that an unreconstructed portion of stock `SMEC_Init()` might prevent real FPGA sample transfer is no longer supported for this fixed profile.

### 15.5 SMEC and the wider FPGA apply protocol

SMEC should not be considered only an early-startup block. The existing runtime traces show later SMEC job groups interleaved or paired with direct FPGA selector writes. The custom fixed-profile profile path also contains key-like / "magic" FPGA register sequences.

The current evidence therefore supports the following working model:

```text
SMEC job/service protocol
        +
direct FPGA selector/register protocol
        +
key-like / apply / commit sequences
        -> coordinated FPGA/front-end configuration
```

The exact semantic role of the key-like values remains open. They should not yet be described as cryptographic authentication. They are, however, useful landmarks when reverse engineering additional FPGA commands because similar apply/commit sequences may recur.

### 15.6 Updated open questions

The following questions remain relevant:

- authoritative physical meaning/name of SMEC jobs 1..7;
- whether the startup values `{20, 64, 64, 1000, 100, 950, 200}` vary with persistent configuration or calibration;
- exact identity and electrical topology of the external security/front-end device behind the FPGA-mediated interface;
- semantic meaning of later SMEC job groups and their paired FPGA selectors;
- semantic meaning of the key-like / magic FPGA apply sequences;
- whether additional FPGA commands require new SMEC jobs or only direct FPGA protocol operations.

The following older questions are now resolved for the supported fixed profile:

```text
Can custom SMEC complete the required startup jobs?       YES
Can the custom runtime complete the profile after SMEC?   YES
Can the FPGA reach real CSI completion?                   YES
Can a full sample payload reach RAM?                      YES, 0x1800 bytes
Is stock SMEC_Init required at runtime?                   NO, custom smec_init() is sufficient
Was the standalone AVS delay failure a SMEC failure?      NO
```

### 15.7 Updated bottom line

`SMEC_Init()` is no longer a blocking unknown for the fixed-profile standalone acquisition path. Its required minimal behavior has been reproduced by the custom `smec_init()` path, and that path has now been validated end-to-end through actual FPGA acquisition, CSI completion, and a full `0x1800`-byte RAM transfer.

SMEC should still be treated as a required part of the currently confirmed communication/configuration chain. The evidence does not justify removing it. Future SMEC work should therefore focus on naming and understanding additional job groups and their relationship to direct FPGA commands, rather than on proving the already working startup transport again.
