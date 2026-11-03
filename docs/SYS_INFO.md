# DPOS350P SYS_INFO field map

**Status:** reverse-engineering reference  
**Updated:** 2026-11-03  
**Base address in the reference build:** `0x40544E68`

## 1. Purpose

`SYS_INFO` is the main stock firmware state block used by acquisition, frontend, trigger, display, settings, measurements, and background services.

This document is the authoritative map for fields that have been identified during static analysis and hardware experiments. Other documents should use the symbolic names defined here and retain the raw offset in parentheses, for example:

```text
SYS_ACQ_DONE (+0x038)
SYS_RUNTIME_READY (+0x6EE)
SYS_RUNTIME_FPGA_POSITION (+0x6F0)
```

The offset is the stable reverse-engineering identifier. A symbolic name may be refined when stronger evidence becomes available.

## 2. Evidence levels

Use these labels consistently:

- **HW-CONFIRMED**: behavior was observed on the physical DPOS350P.
- **STATIC-CONFIRMED**: reads, writes, arithmetic, or call relationships are unambiguous in the stock disassembly.
- **VERIFIED SETTING**: a setting change was applied through the stock interface and its effect was observed.
- **STRONGLY INFERRED**: the role is supported by static and runtime behavior, but the exact user-facing meaning is not yet proven.
- **WORKING NAME**: the field is known to participate in a specific path, but its precise semantics remain open.
- **UNKNOWN**: no safe semantic name is available yet.

Do not promote a working name to a confirmed semantic name merely because it is convenient in code.

## 3. Naming policy

Preferred C-style names use a `SYS_` prefix:

```c
#define SYS_ACQ_PATH_MODE_OFF              0x028U
#define SYS_ACQ_DONE_OFF                   0x038U
#define SYS_CH1_ENABLED_OFF                0x03EU
#define SYS_CH2_ENABLED_OFF                0x102U
#define SYS_TRIGGER_MODE_OFF               0x27AU
#define SYS_CAPTURE_COUNT_OFF              0x550U
#define SYS_RUNTIME_READY_OFF              0x6EEU
#define SYS_RUNTIME_FPGA_POSITION_OFF      0x6F0U
#define SYS_RUNTIME_STEP_OFF               0x6F6U
#define SYS_RUNTIME_COARSE_STEP_OFF        0x6F8U
#define SYS_RUNTIME_FINE_PROGRESS_OFF      0x6FCU
#define SYS_RUNTIME_COARSE_PROGRESS_OFF    0x700U
#define SYS_RUNTIME_PROGRESS_THRESHOLD_OFF 0x708U
```

For fields whose exact meaning is still open, keep the offset visible in the name if necessary:

```c
#define SYS_RUNTIME_FIELD_6F4_OFF 0x6F4U
#define SYS_RUNTIME_FIELD_704_OFF 0x704U
```

## 4. Acquisition control fields

### `SYS_ACQ_PATH_MODE (+0x028, u8)`

**Evidence:** STATIC-CONFIRMED, STRONGLY INFERRED.

The acquisition dispatcher at `0x400419CC` selects between the full-acquisition and runtime-sweep paths using this field:

```text
value == 0  -> 0x40041918 -> full acquisition 0x400413C8
value != 0  -> runtime sweep 0x4004191C
```

Known writers include `0x4003BE90` and `0x4003EC5C`. The exact user-visible mode name is not yet proven, so `ACQ_PATH_MODE` is preferred over a stronger label such as `RUN_MODE`.

### `SYS_ACQ_PATH_MODE_SHADOW (+0x029, u8)`

**Evidence:** STATIC-CONFIRMED.

`0x4003BEB4` copies `SYS_ACQ_PATH_MODE (+0x028)` into this field while committing the runtime state. The exact downstream use remains open.

### `SYS_ACQ_DONE (+0x038, u8)`

**Evidence:** STATIC-CONFIRMED and HW-CONFIRMED.

This field stores bit 0 read from FPGA selector `0x0A` by stock helper `0x4003F9D0`.

```text
SYS_ACQ_DONE = fpga_read_selector(0x0A) & 1
```

It is also initialized or forced by several stock runtime helpers. Use this name for the software-cached selector `0x0A` completion bit, not for runtime selector `0x0B` readiness.

### `SYS_ACQ_TRANSITION_FLAG (+0x03C, u8)`

**Evidence:** STATIC-CONFIRMED, WORKING NAME.

This field gates the dispatcher at `0x400419CC`. If nonzero, the dispatcher clears `SYS_ACQ_DONE (+0x038)` and returns without executing the normal full/runtime branch. It is also set at the end of a specific mode-1 full-acquisition path before a tail call to `0x4003CDC0`.

The exact semantic meaning remains open. Do not call it simply `RUNNING` or `STOPPED` yet.

### `SYS_CH1_ENABLED (+0x03E, u8)`

**Evidence:** STATIC-CONFIRMED and HW-CONFIRMED.

Controls CH1 capture and processing branches. The confirmed CH1-only acquisition baseline uses:

```text
SYS_CH1_ENABLED = 1
SYS_CH2_ENABLED = 0
```

### `SYS_CH1_RUNTIME_STATE (+0x101, u8)`

**Evidence:** STATIC-CONFIRMED, WORKING NAME.

Written by full-acquisition and related runtime paths. It is not yet safe to name this field as a simple enable or done flag.

### `SYS_CH2_ENABLED (+0x102, u8)`

**Evidence:** STATIC-CONFIRMED and HW-CONFIRMED.

Controls CH2 capture and processing branches.

### `SYS_TRIGGER_MODE (+0x27A, u8)`

**Evidence:** VERIFIED SETTING and STATIC-CONFIRMED.

Applied to FPGA command `0x1A` together with neighboring trigger fields. It also affects full-acquisition completion behavior and capture-count handling.

### `SYS_TRIGGER_EDGE (+0x27B, u8)`

**Evidence:** VERIFIED SETTING and STATIC-CONFIRMED.

Applied through FPGA command `0x16`.

### `SYS_TRIGGER_CHANNEL (+0x27C, u8)`

**Evidence:** VERIFIED SETTING and STATIC-CONFIRMED.

Applied through FPGA command `0x15`. The same field also affects the even/odd phase selection generated by stock helper `0x4006338C`.

### `SYS_CAPTURE_COUNT (+0x550, u16)`

**Evidence:** STATIC-CONFIRMED and HW-CONFIRMED.

Number of capture iterations used by full acquisition `0x400413C8`. A confirmed DPOSLAB full-acquisition run observed a value of `4`.

### `SYS_CAPTURE_PHASE_BASE (+0x584, u16)`

**Evidence:** STATIC-CONFIRMED, STRONGLY INFERRED.

Before each full-acquisition iteration, stock computes:

```text
phase_argument = low8(SYS_CAPTURE_PHASE_BASE + capture_index)
```

and passes it to `0x4006338C`, which programs FPGA command `0x15` with a derived `0..7`, even, or odd phase value.

### `SYS_ACQ_PROFILE_OR_MODE (+0x588, u8)`

**Evidence:** STATIC-CONFIRMED, WORKING NAME.

This heavily referenced field affects runtime-state initialization, postprocessing, and several settings/test paths. The exact semantic label remains open. Keep the offset in discussions until the field is isolated experimentally.

## 5. Runtime READY field

### `SYS_RUNTIME_READY (+0x6EE, u8)`

**Evidence:** STATIC-CONFIRMED and HW-CONFIRMED.

Stock helper `0x4003F8B8` reads FPGA selector `0x0B` and stores bit 0 here:

```text
SYS_RUNTIME_READY = fpga_read_selector(0x0B) & 1
```

The D0.4A baseline confirmed:

```text
before runtime rearm: FPGA 0x0B = B8, bit0 = 0
 after runtime rearm: FPGA 0x0B = B9, bit0 = 1
SYS_RUNTIME_READY: 0 -> 1 during runtime capture
```

Do not confuse this field with `SYS_ACQ_DONE (+0x038)`, which caches bit 0 of selector `0x0A`.

## 6. Runtime sweep structure

The runtime sweep structure is anchored at `SYS_INFO+0x6F0`. Stock helper `0x4003BF98` treats this region as a group of related counters and increments.

### `SYS_RUNTIME_FPGA_POSITION (+0x6F0, u16)`

**Evidence:** STATIC-CONFIRMED and HW-CONFIRMED.

`0x4003F910` writes this value to FPGA selector `0x09`, most significant byte first.

Observed progression:

```text
FULL           0x0002
INITIAL REARM  0x0008
C1 after adv   0x000E
C2 after adv   0x0014
C3 after adv   0x001A
C4 after adv   0x0021
```

`RUNTIME_FPGA_POSITION` is a safe implementation-oriented name. Its exact physical unit remains open.

### `SYS_RUNTIME_POSITION_BASE (+0x6F2, u16)`

**Evidence:** STATIC-CONFIRMED, WORKING NAME.

Used by stock position/runtime math and historically paired with acquisition geometry fields. The exact relationship to `SYS_RUNTIME_FPGA_POSITION (+0x6F0)` requires further analysis.

### `SYS_RUNTIME_FIELD_6F4 (+0x6F4, u16)`

**Evidence:** UNKNOWN.

Reserved in the field map to preserve the structure layout. Do not assign a semantic name yet.

### `SYS_RUNTIME_STEP (+0x6F6, u16)`

**Evidence:** STATIC-CONFIRMED, STRONGLY INFERRED.

`0x4003BF98` adds this value to `SYS_RUNTIME_FINE_PROGRESS (+0x6FC)` during every runtime advance:

```text
SYS_RUNTIME_FINE_PROGRESS += SYS_RUNTIME_STEP
```

D0.4C observed `+0x6FC` increments of `0x32` per cycle in the tested profile.

### `SYS_RUNTIME_COARSE_STEP (+0x6F8, u16)`

**Evidence:** STATIC-CONFIRMED, STRONGLY INFERRED.

When the fine/current progress reaches the runtime threshold, `0x4003BF98` adds this value to `SYS_RUNTIME_COARSE_PROGRESS (+0x700)` and to a related accumulated position field.

### `SYS_RUNTIME_FIELD_6FA (+0x6FA, u16)`

**Evidence:** UNKNOWN.

Reserved to preserve structure layout.

### `SYS_RUNTIME_FINE_PROGRESS (+0x6FC, u32)`

**Evidence:** STATIC-CONFIRMED and HW-CONFIRMED.

Current fine-grained sweep progress used by `0x4003BF98`.

Observed D0.4C progression:

```text
0x00000000
0x00000032
0x00000064
0x00000096
0x000000C8
```

This field is compared against `SYS_RUNTIME_PROGRESS_THRESHOLD (+0x708)` before the coarse progress is incremented.

### `SYS_RUNTIME_COARSE_PROGRESS (+0x700, u32)`

**Evidence:** STATIC-CONFIRMED and HW-CONFIRMED.

Coarse runtime sweep progress. It is not incremented on every capture. The update is conditional on the fine progress reaching the threshold.

Observed D0.4C progression:

```text
FULL / REARM / C1 / C2: 0
C3:                       1
C4:                       2
```

The stock runtime loop compares this field against `SYS_INFO+0x008`.

### `SYS_RUNTIME_FIELD_704 (+0x704, u32)`

**Evidence:** UNKNOWN.

Reserved until its readers and writers are characterized.

### `SYS_RUNTIME_PROGRESS_THRESHOLD (+0x708, u32)`

**Evidence:** STATIC-CONFIRMED, STRONGLY INFERRED.

`0x4003BF98` compares `SYS_RUNTIME_FINE_PROGRESS (+0x6FC)` against this field. If fine progress is below this threshold, the function skips the coarse-progress update.

The exact physical meaning and unit remain open.

## 7. Runtime arithmetic confirmed in `0x4003BF98`

The current implementation-oriented pseudocode is:

```c
if (state->fine_progress >= state->progress_threshold) {
    state->coarse_progress += state->coarse_step;
    related_accumulated_position += state->coarse_step;
}

state->fine_progress += state->step;

quotient = state->fine_progress / SYS_INFO[0x1B1];
remainder = state->fine_progress % SYS_INFO[0x1B1];

SYS_INFO[0x63C] = quotient;
state->remainder = remainder;
state->fpga_position += quotient;
```

The names `step`, `fine_progress`, `coarse_step`, `coarse_progress`, and `progress_threshold` are implementation-oriented names supported by the code and D0.4C progression. They are not yet claims about screen pixels, samples, nanoseconds, or trigger units.

## 8. Additional acquisition-related fields

### `SYS_RUNTIME_QUOTIENT (+0x63C, u32)`

**Evidence:** STATIC-CONFIRMED.

Stores the quotient computed during runtime advance and is consumed by positioning/runtime math. It is reset by `0x4003C008`.

### `SYS_RUNTIME_COPY_LENGTH (+0x6F6, u16)`

Do not create this alias. Earlier working notes associated `+0x6F6` with a copy length, but the direct arithmetic in `0x4003BF98` confirms that the field participates as the per-step increment. If another function interprets the same value as a copy length, document that as an additional use rather than replacing the canonical name.

### `SYS_ACQ_TARGET (+0x008, u16)`

**Evidence:** STATIC-CONFIRMED, WORKING NAME.

The runtime sweep path compares `SYS_RUNTIME_COARSE_PROGRESS (+0x700)` against this field:

```text
while (SYS_RUNTIME_COARSE_PROGRESS < SYS_ACQ_TARGET) {
    runtime_iteration();
}
```

A DPOSLAB diagnostic observed `0x03B6`. The exact unit remains open.

## 9. Fields recovered from FPGA/settings documentation

This section consolidates additional `SYS_INFO` fields already identified through the stock FPGA command and register-file interface. These fields belong in the central map even when they are not directly part of the runtime sweep state machine.

### Channel and frontend settings

#### `SYS_CH1_COUPLING (+0x03F, u8)`

**Evidence:** VERIFIED SETTING and STATIC-CONFIRMED.

CH1 coupling setting, applied through FPGA command `0x2F`.

#### `SYS_CH1_BANDWIDTH_FILTER (+0x047, u8)`

**Evidence:** VERIFIED SETTING and STATIC-CONFIRMED.

CH1 bandwidth-derived filter value, applied through FPGA command `0x29`.

#### `SYS_CH2_COUPLING (+0x103, u8)`

**Evidence:** VERIFIED SETTING and STATIC-CONFIRMED.

CH2 coupling setting, applied through FPGA command `0x30`.

#### `SYS_CH2_BANDWIDTH_FILTER (+0x10B, u8)`

**Evidence:** VERIFIED SETTING and STATIC-CONFIRMED.

CH2 bandwidth-derived filter value, applied through FPGA command `0x2C`.

#### `SYS_CH1_FPGA_VERTICAL_SOURCE (+0x000, width open)`

**Evidence:** STATIC-CONFIRMED, WORKING NAME.

CH1-side source used by the stock vertical apply path associated with FPGA command `0x2E`. Exact width and physical meaning remain open.

#### `SYS_CH2_FPGA_VERTICAL_SOURCE (+0x100, width open)`

**Evidence:** STATIC-CONFIRMED, WORKING NAME.

CH2-side counterpart used by the stock vertical apply path associated with FPGA command `0x2E`. Exact semantics remain open.

### FPGA command-backed fields with open semantics

#### `SYS_FPGA_CMD62_VALUE (+0x03A, u8)`

**Evidence:** STATIC-CONFIRMED, UNKNOWN SEMANTICS.

Value sent through FPGA command `0x62`.

#### `SYS_FPGA_CMD61_VALUE (+0x03B, u8)`

**Evidence:** STATIC-CONFIRMED, UNKNOWN SEMANTICS.

Value sent through FPGA command `0x61`.

#### `SYS_FRONTEND_MIRROR (+0x6E4..., mixed)`

**Evidence:** STATIC-CONFIRMED, WORKING REGION NAME.

Front-end values mirrored to FPGA commands `0x04`-`0x07` and `0x0D`. Stock transmits individual values only when their cached copies change. Split this region into dedicated field entries as the individual semantics are recovered.

### Acquisition geometry and FPGA register-file inputs

The stock FPGA register-file initialization uses the following `SYS_INFO` fields for acquisition geometry. Their protocol linkage is confirmed; several physical meanings remain open.

#### `SYS_ACQ_TARGET (+0x008, u16)`

**Evidence:** STATIC-CONFIRMED and HW-CONFIRMED value observation.

This field has two confirmed relationships:

```text
runtime termination: SYS_RUNTIME_COARSE_PROGRESS (+0x700) < SYS_ACQ_TARGET
FPGA register file:  loaded into acquisition register 0x06
```

A live DPOSLAB value of `0x03B6` was observed. The exact physical unit remains open.

#### `SYS_ACQ_GEOMETRY_1B8 (+0x1B8, width open)`

**Evidence:** STATIC-CONFIRMED, WORKING NAME.

Input to the FPGA acquisition register-file setup (`0x01`-`0x07`). Exact geometry meaning remains open.

#### `SYS_ACQ_GEOMETRY_1C0 (+0x1C0, u32)`

**Evidence:** STATIC-CONFIRMED, STRONGLY INFERRED.

Acquisition geometry value documented in the stock FPGA path as using 64 codes per display pixel. Preserve the offset until its physical unit is fully verified.

#### `SYS_ACQ_GEOMETRY_268 (+0x268, u32)`

**Evidence:** STATIC-CONFIRMED, WORKING NAME.

Input to the stock FPGA acquisition register-file configuration.

#### `SYS_ACQ_GEOMETRY_26C (+0x26C, u32)`

**Evidence:** STATIC-CONFIRMED, WORKING NAME.

Input to the stock FPGA acquisition register-file configuration.

#### `SYS_ACQ_GEOMETRY_270 (+0x270, u32)`

**Evidence:** STATIC-CONFIRMED, WORKING NAME.

Input to the stock FPGA acquisition register-file configuration.

#### `SYS_ACQ_GEOMETRY_62C (+0x62C, u32)`

**Evidence:** STATIC-CONFIRMED, WORKING NAME.

Input to FPGA register `0x07` during the stock acquisition register-file setup.

### Per-core FPGA clock/phase masks

#### `SYS_CORE_PHASE_MASK_0..7 (+0x674..+0x67B, u8[8])`

**Evidence:** STATIC-CONFIRMED, STRONGLY INFERRED.

Eight masks generated from sample-rate/channel-mode state and sent through FPGA commands `0xA6..0xAD`. The values form rotating 4-of-8 masks such as `0x0F`, `0x1E`, and related rotations. The current interpretation is per-core clock/phase selection rather than analog gain/offset correction.

This region is associated with the FPGA register-file commit/debug phases `B9..C0`.

### Power and health fields

#### `SYS_AUTO_POWER_OFF_MINUTES (+0x658, width open)`

**Evidence:** STATIC-CONFIRMED.

Auto-power-off interval in minutes used by the stock power-management path.

#### `SYS_CHARGING (+0x779, u8)`

**Evidence:** STATIC-CONFIRMED.

Bit 0 read from FPGA command `0x02` is stored here and used by the stock charging/battery indicator path.

#### `SYS_FPGA_ALIVE (+0x7FC, width open)`

**Evidence:** STATIC-CONFIRMED.

Stores the result of the FPGA alive/ID check associated with command `0x01`. The documented FPGA response is `A5 A5 5A 5A`.


## 10. Calibration persistence and serializer state

This section records the stock calibration state identified by static analysis and confirmed by the DPOSLAB D0.4W12/D0.4W13 hardware experiments. The persistence and serializer globals below are not all located inside the main `SYS_INFO` allocation, but they belong in this central state map because they are required to interpret the stock acquisition and calibrated sample-conversion path.

### `SYS_CALIBRATION_STORAGE_ADDRESS (+0x7E0, u32)`

**Evidence:** STATIC-CONFIRMED and HW-CONFIRMED.

The stock calibration-load wrapper at `0x400446B4` reads this field and passes it as the persistence source address to `0x40043620`. The wrapper transfers `0x5000` bytes into the serializer staging buffer, selects decode mode, and invokes the calibration field traversal at `0x40044450`.

DPOSLAB D0.4W13 observed:

```text
SYS_CALIBRATION_STORAGE_ADDRESS = 0x001FB000
```

Use `STORAGE_ADDRESS` rather than `FLASH_ADDRESS` until the physical backend and address-space semantics are documented conclusively.

### External stock calibration object

#### `STOCK_CALIBRATION_OBJECT (0x42E53A0C)`

**Evidence:** STATIC-CONFIRMED and HW-CONFIRMED.

This external stock object contains the active per-lane coefficients and the larger persisted calibration tables used by the stock raw-sample conversion path. It is separate from the main `SYS_INFO` allocation.

Recommended stable definitions:

```c
#define STOCK_CALIBRATION_OBJECT_ADDR       0x42E53A0CU
#define STOCK_CAL_ACTIVE_COEFF_A_ADDR       (STOCK_CALIBRATION_OBJECT_ADDR + 0x10U)
#define STOCK_CAL_ACTIVE_COEFF_B_ADDR       (STOCK_CALIBRATION_OBJECT_ADDR + 0x30U)
#define STOCK_CAL_AUX_ARRAYS_ADDR           (STOCK_CALIBRATION_OBJECT_ADDR + 0x50U)
#define STOCK_CAL_MATRIX_ADDR               (STOCK_CALIBRATION_OBJECT_ADDR + 0x190U)
```

The `A/B` coefficient labels are intentionally neutral working names. Do not promote them to `gain`, `offset`, `scale`, or `zero` until the complete arithmetic and physical units are proven.

#### Active coefficient set A (`+0x10..+0x2C`, `u32[8]`)

**Evidence:** STATIC-CONFIRMED, STRONGLY INFERRED.

The stock selector path populates this eight-element set from the larger calibration matrix. The conversion core around `0x4004073C` selects one value according to the raw bank/lane index and uses it directly in the calibrated `uint8_t` to `uint16_t` sample conversion.

#### Active coefficient set B (`+0x30..+0x4C`, `u32[8]`)

**Evidence:** STATIC-CONFIRMED and HW-CONFIRMED.

Selector helpers including `0x4004C5F0` and `0x4004C674` participate in populating the active coefficient region from the larger calibration matrix. The conversion core around `0x4004073C` reads the selected value for the current raw bank/lane.

DPOSLAB D0.4W13 confirmed the dependency:

```text
active coefficient pairs before selector  0/8 non-zero
active coefficient pairs after selector   8/8 non-zero
calibrated CORE output before selector     0/8 non-zero
calibrated CORE output after selector      8/8 non-zero
```

#### Auxiliary calibration arrays (`+0x50..+0x18F`)

**Evidence:** STATIC-CONFIRMED, UNKNOWN SEMANTICS.

The stock serializer traverses four arrays of 20 32-bit elements. Their serialization order is the RAM block order `+0x50`, `+0xF0`, `+0xA0`, `+0x140`, with one element from each array processed per iteration. The exact semantic role of the four arrays remains open.

#### Calibration matrix (`+0x190...`)

**Evidence:** STATIC-CONFIRMED and HW-CONFIRMED.

The main persisted matrix consists of 25 columns, each containing 50 32-bit elements. RAM column stride is `0xC8` bytes (`50 * 4`). The field traversal serializes the matrix index-first across all 25 columns:

```text
for index = 0..49:
    for column = 0..24:
        process_u32(object + 0x190 + column * 0xC8 + index * 4)
```

The serialized size of this first block is:

```text
50 * 25 * 4 = 5000 bytes = 0x1388
```

Together with the four auxiliary arrays (`0x140` bytes), the active serialized calibration data totals:

```text
0x1388 + 0x0140 = 0x14C8 bytes
```

The persistence wrapper transfers `0x5000` bytes. Bytes beyond serialized offset `0x14C8` are not traversed by `0x40044450`; their ownership and required contents remain open.

Several observed 32-bit values have bit patterns compatible with IEEE-754 single-precision values. For example, D0.4W13 observed `0x40200000`, which represents `2.5f`. This does not prove that every matrix column is a `float`; the serializer itself treats each entry only as a 32-bit scalar.

### Stock serializer globals

#### `STOCK_SERIAL_BUFFER (0x408A526C)`

**Evidence:** STATIC-CONFIRMED.

This address is `SYS_INFO_BASE + 0x360404` and is the staging buffer used by the stock calibration serializer. The calibration persistence wrappers transfer `0x5000` bytes between this buffer and the persistence backend.

#### `STOCK_SERIAL_CURSOR (0x4091F38C, u32)`

**Evidence:** STATIC-CONFIRMED and HW-CONFIRMED.

This address is `SYS_INFO_BASE + 0x3DA524`. Function `0x40067128` writes the cursor, and both field-processing directions advance it by the requested scalar size.

`0x40044450` resets the cursor to zero before traversing the calibration schema. DPOSLAB D0.4W13 observed:

```text
before stock calibration load  0x00000000
after stock calibration load   0x000014C8
```

#### `STOCK_SERIAL_MODE (0x4091F390, u8)`

**Evidence:** STATIC-CONFIRMED and HW-CONFIRMED.

```text
0 = encode/save: live fields -> staging buffer
1 = decode/load: staging buffer -> live fields
```

Function `0x40067134` selects encode mode. Function `0x40067144` selects decode mode. DPOSLAB D0.4W13 observed mode `0 -> 1` during the stock calibration-load path.

### Stock serializer function map

```text
0x40067128  serializer_set_cursor
0x40067134  serializer_set_encode_mode
0x40067144  serializer_set_decode_mode
0x40067160  serializer_encode_scalar_be
0x40067278  serializer_encode_field
0x400672A8  serializer_decode_scalar_be
0x40067420  serializer_decode_field
0x40067450  serializer_process_field
0x40044450  calibration_process_serialized_fields
0x40044680  calibration_save_wrapper
0x400446B4  calibration_load_wrapper
```

The scalar serializer supports widths `1`, `2`, `4`, and `8` bytes. Multi-byte values are represented in big-endian byte order in the staging buffer. Unsupported widths perform no copy in the scalar helper, although the field wrapper still advances the cursor by the requested size.

### Persistence backend functions

```text
0x40043620  persistence_read
0x40043874  persistence_write_read_modify_write
```

`0x40043874` performs erase-aware 4 KiB read-modify-write handling through a scratch block at `0x42E5E010`. The normal D0.4W13 experiment uses only the read/decode wrapper `0x400446B4`; it does not invoke calibration save or factory calibration generation.

### D0.4W13 hardware-confirmed state transition

```text
persistence source address     0x001FB000
serializer cursor              0x00000000 -> 0x000014C8
serializer mode                0x00000000 -> 0x00000001
matrix sample 0                0x00000000 -> 0x40200000
matrix sample 1                0x00000000 -> 0x40200000
source non-zero pairs                         8/8
active pairs before selector                  0/8
active pairs after selector                   8/8
changed active pairs                          8/8
CORE non-zero before selector                 0/8
CORE non-zero after selector                  8/8
verdict                        0x50415353 (`PASS`)
```

**HW-CONFIRMED conclusion:** the earlier zero output from the unchanged stock calibrated conversion core was caused by missing calibration persistence state, not by failure of the raw FPGA/CSI acquisition transport. After the stock load restored the calibration matrix, the stock selector populated all eight active coefficient pairs and the unchanged stock conversion core produced non-zero output for all eight lanes.

## 11. Known aliases and terminology corrections

The central map supersedes these older raw-offset names in new documentation:

```text
SYS_INFO+0x038  -> SYS_ACQ_DONE
SYS_INFO+0x6EE  -> SYS_RUNTIME_READY
SYS_INFO+0x6F0  -> SYS_RUNTIME_FPGA_POSITION
SYS_INFO+0x6F6  -> SYS_RUNTIME_STEP
SYS_INFO+0x6FC  -> SYS_RUNTIME_FINE_PROGRESS
SYS_INFO+0x700  -> SYS_RUNTIME_COARSE_PROGRESS
SYS_INFO+0x708  -> SYS_RUNTIME_PROGRESS_THRESHOLD
```

The older hypothesis that FPGA commands `0x08/0x09` were simply a CH1/CH2 offset pair is superseded by the runtime reconstruction: selector `0x08` is used as the runtime rearm pulse and selector `0x09` receives `SYS_RUNTIME_FPGA_POSITION (+0x6F0)`.

## 12. Code access recommendations

Prefer typed helpers over raw pointer arithmetic spread through the codebase:

```c
static inline uint8_t sys_read_u8(uint32_t offset);
static inline uint16_t sys_read_u16(uint32_t offset);
static inline uint32_t sys_read_u32(uint32_t offset);

static inline void sys_write_u8(uint32_t offset, uint8_t value);
static inline void sys_write_u16(uint32_t offset, uint16_t value);
static inline void sys_write_u32(uint32_t offset, uint32_t value);
```

For the custom runtime, prefer an independent structure rather than retaining dependency on stock RAM:

```c
struct acq_runtime_state {
    uint16_t fpga_position;
    uint16_t position_base;
    uint16_t field_6f4;
    uint16_t step;
    uint16_t coarse_step;
    uint16_t field_6fa;
    uint32_t fine_progress;
    uint32_t coarse_progress;
    uint32_t field_704;
    uint32_t progress_threshold;
};
```

Before treating this structure as final, confirm packing and all accesses to `+0x6F4`, `+0x6FA`, and `+0x704`.

## 13. Maintenance rules

When adding or renaming a field:

1. Preserve the raw offset and width.
2. Record readers and writers when known.
3. Mark the evidence level.
4. Distinguish direct observation from interpretation.
5. Update references in `ACQUISITION.md`, `FPGA.md`, and source comments.
6. Keep deprecated working names in a short alias note to avoid losing searchability.
7. Do not reuse one symbolic name for selector `0x0A` and selector `0x0B` readiness.

## 14. Current priority

The next SYS_INFO mapping target is the complete runtime structure at `+0x6F0..+0x708`, especially:

```text
+0x6F4
+0x6FA
+0x704
+0x708
```

The goal is to replace remaining working names with structurally and physically meaningful names before implementing the fully custom runtime acquisition state machine.
