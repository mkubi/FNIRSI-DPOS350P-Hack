# DPOSLAB Resident Runtime Monitor

DPOSLAB is a resident runtime reverse-engineering and instrumentation environment for the FNIRSI DPOS350P stock EXE2 image. It is loaded separately from EXE2, remains resident while the stock firmware runs, and can temporarily redirect selected stock execution points into diagnostic code before resuming the original firmware.

The first hardware-confirmed baseline intercepts the stock `FPGA_Start()` call, executes the original function, returns to the DPOSLAB monitor, and can then continue normal stock execution.


```markdown
## Automatic Runtime Trace

DPOSLAB runtime v2 supports automatic checkpoint recording into a resident
RAM trace buffer.

The trace is exported manually through the stock Screenshot flow:
```


```text
runtime checkpoints
    -> resident RAM trace
    -> stock UI
    -> Screenshot trigger
    -> stock FAT API
    -> /DPOSLAB.TRC
```

Trace ABI v1:

Plain Text
magic 0x42414C44 ("DLAB")
version 1
header size 0x20
record size 0x84
capacity 32 records

The trace contains CPU context and passive PE, CCU and CSI snapshots.

The PC-side decoder is:

Plain Text
tools/decode_dposlab_trace.py

It validates the binary trace, identifies checkpoint IDs and produces automatic differences between adjacent runtime phases.

The tracing pipeline has been HW-confirmed with multiple internal FPGA_Start() checkpoints and zero dropped records.


## Status

**Hardware-confirmed baseline:** `dposlab-runtime-monitor-v1`

**Confirmed properties:**

- EXE2 is cold-started through its image entry at `0x40000000`.
- DPOSLAB remains resident at `0x42E70000` while EXE2 runs.
- The stock `FPGA_Start()` call site can be patched in RAM.
- A near veneer can transfer control from stock EXE2 to resident DPOSLAB code.
- DPOSLAB can call the original `FPGA_Start()` implementation.
- Control returns from `FPGA_Start()` to DPOSLAB.
- DPOSLAB can capture CPU and passive MMIO state.
- DPOSLAB can resume stock execution at the original continuation address.
- The oscilloscope remains operational after the hook and resume sequence.

DPOSLAB is therefore more than an alternative loader. It is a confirmed resident runtime hook and debug-monitor foundation for stock EXE2.

## Repository Layout

The implementation is split between the Stage 2 loader and the resident DPOSLAB application.

```text
loader/
  dposlab.c              Load DPOSLAB.BIN and prepare its image descriptor
  dposlab.h              Load address, limits, and 8.3 filename
  dposlab_abi.h          Stage 2 to DPOSLAB handoff structure
  dposlab_jump.S         Stage 2 handoff trampoline
  boot_targets.c         Boot-menu integration

apps/dposlab/
  Makefile               Resident application build
  link.ld                Resident memory layout at 0x42E70000
  start.S                DPOSLAB entry and BSS initialization
  main.c                 Validation, monitor UI, and experiment control
  stock_api.c/.h         Known stock symbols and validation
  stock_jump.S           EXE2 cold-entry handoff
  runtime_patch.c/.h     Runtime call-site patch and near veneer
  runtime_hook.S         Register-preserving FPGA_Start wrapper
  runtime_snapshot.c/.h  BEFORE/AFTER CPU and MMIO snapshots
```

The exact file set can evolve in later DPOSLAB versions. The addresses and behavior in this document describe the confirmed v1 baseline and the initial v2 snapshot work.

## Memory Map

```text
0x40000000  Stock EXE2 image base and executable entry
0x40034700  Stock C main()
0x40034738  Stock BL call site for FPGA_Start()
0x4003473C  Stock continuation after FPGA_Start()
0x40049188  Stock FPGA_Start()
0x40055B4C  DPOSLAB near veneer used by the v1 PoC
0x42E70000  Resident DPOSLAB image base
0x42E7E000  Maximum resident code/data end enforced by link.ld
0x42E80000  DPOSLAB stack top and resident-region limit
```

DPOSLAB code and data occupy the lower part of the `0x42E70000-0x42E7FFFF` region. The linker assertion keeps resident code and data below `0x42E7E000`, reserving 8 KiB below `0x42E80000` for the DPOSLAB stack.

## Correct EXE2 Cold Entry

The stock EXE2 image must be entered through `0x40000000`, not by jumping directly to its C `main()` at `0x40034700`.

The first EXE2 vector instruction is:

```asm
0x40000000:  ldr pc, [pc, #0x18]
```

Its literal at image offset `+0x20` is:

```text
0x40000020: 0x4008CB28
```

The cold-start chain is therefore:

```text
0x40000000  EXE2 image entry / reset-vector stub
    |
    +--> 0x4008CB28  reset and runtime startup handler
             |
             +--> IRQ, SYS, and SVC stack initialization
             +--> low-vector selection and vector-table copy
             +--> VFP/CP10/CP11 initialization
             +--> Thumb runtime/scatter initialization
             |
             +--> 0x4008CBDC: BL 0x40034700
                                      |
                                      +--> C main()
```

Direct entry at `0x40034700` bypasses the stock runtime contract and is not a valid cold-start replacement.

## Stage 2 Handoff

Stage 2 loads EXE2 at `0x40000000`, loads DPOSLAB at `0x42E70000`, and passes a handoff structure to DPOSLAB.

The handoff records include:

- ABI magic and version
- structure size
- flags
- captured Stage 2 stack pointer
- EXE2 base, size, and entry
- DPOSLAB base, size, and entry

The confirmed EXE2 handoff uses:

```c
exe2_base  = 0x40000000U;
exe2_entry = 0x40000000U;
```

Before transferring control, DPOSLAB validates the handoff ABI and known stock opcodes. The EXE2 entry validation checks the reset-vector instruction `0xE59FF018`, not the `main()` prologue.

## Runtime Hook Architecture

### Original stock flow

The relevant part of stock `main()` is:

```text
0x40034738: BL 0x40049188   ; FPGA_Start()
0x4003473C: ...             ; continuation
```

### Patched flow

DPOSLAB replaces the `BL` at `0x40034738` with a `BL` to a nearby veneer:

```text
stock main()
    |
    +--> patched BL at 0x40034738
             |
             +--> near veneer at 0x40055B4C
                      |
                      +--> absolute resident-hook address at 0x42E7xxxx
                               |
                               +--> DPOSLAB wrapper
                                        |
                                        +--> stock FPGA_Start at 0x40049188
                                        +--> DPOSLAB monitor
                                        +--> restore stock context
                                        +--> return to 0x4003473C
```

### Why a near veneer is required

A classic ARM `B` or `BL` immediate has a range of approximately plus or minus 32 MiB. The resident DPOSLAB image at `0x42E70000` is outside direct branch range from the stock call site at `0x40034738`.

The v1 veneer is an 8-byte absolute jump:

```asm
0x40055B4C:  ldr pc, [pc, #-4]
0x40055B50:  .word dposlab_fpga_start_hook
```

The stock call site can reach the veneer with a normal ARM `BL`; the veneer then loads the absolute resident-hook address into `PC`.

The v1 PoC uses the final aligned slot of the existing near-veneer area:

```text
0x40055AD4-0x40055B54
```

This location is a confirmed project-specific resource, not a generally free EXE2 region. Future hook allocation must coordinate veneer ownership and must not overlap another active resident plugin.

## Cache Coherency

Runtime code patching requires explicit cache synchronization. Merely writing a new instruction to RAM is insufficient because the processor may continue executing an old instruction from I-cache or branch-predictor state.

After writing the veneer and call-site patch, DPOSLAB performs:

```text
1. Clean D-cache for the veneer.
2. Clean D-cache for the patched call site.
3. Invalidate the instruction cache.
4. Invalidate branch-predictor state.
5. Execute DSB.
6. Execute ISB.
```

Any future runtime patch mechanism must preserve this sequence or provide an equivalent architecture-correct synchronization path.

## FPGA_Start Wrapper

The confirmed wrapper saves the stock call-site context before invoking any diagnostic C code.

Conceptually:

```asm
push    {r0-r12, lr}       ; original stock context

; capture BEFORE state

ldmia   sp, {r0-r12}       ; restore original input registers
ldr     r12, =0x40049188
blx     r12                ; original FPGA_Start()

push    {r0-r12, lr}       ; exact post-return GPR state

; capture AFTER state
; execute monitor

add     sp, sp, #56        ; discard post-call frame
pop     {r0-r12, pc}       ; restore original stock context and continue
```

The original `LR` saved at hook entry is `0x4003473C`, so restoring it into `PC` naturally resumes the stock firmware after the original `FPGA_Start()` call.

## Snapshot Model

DPOSLAB v2 captures two classes of information around `FPGA_Start()`.

### CPU state

The CPU snapshot includes:

- `r0-r12`
- `LR`
- `CPSR`
- original stock `SP`
- wrapper execution `SP`

The initial hardware-confirmed snapshot produced:

```text
CPSR before:       0x60000153
CPSR after:        0x20000153
GPR changed mask:  0x0000100F
Stock SP:          0x42FE8D20
FPGA execution SP: 0x42FE8CE8
```

The changed-register mask corresponds to:

```text
r0, r1, r2, r3, r12
```

These are ARM caller-saved registers. Registers `r4-r11` were preserved across `FPGA_Start()`, as expected by the ABI.

The stack delta is:

```text
0x42FE8D20 - 0x42FE8CE8 = 0x38 = 56 bytes
```

This exactly matches the wrapper frame created by `push {r0-r12, lr}` and is an important consistency check for the snapshot mechanism.

### Passive MMIO state

The initial snapshot reads MMIO registers without issuing FPGA selector transactions or changing PE pin direction. This minimizes observer effect.

Currently sampled groups include:

- PE pin configuration, data, and pull state
- selected CCU gate/reset registers
- selected CSI configuration and status registers

Active FPGA reads are intentionally excluded from the passive boundary snapshot because they modify the FPGA parallel bus state.

## Photographic Breakpoints

The DPOS350P has no suitable physical debug button. The power control is not usable as a continue input, and invoking touch initialization inside an early runtime breakpoint could modify the state being investigated.

The current practical solution is a timed photographic breakpoint:

```text
capture state
    |
    +--> display diagnostic page
    +--> busy-wait approximately 3 seconds
    +--> display next page or resume stock execution
```

The delay is empirical and deliberately independent of stock timers. It is intended for photographing the screen, not for accurate time measurement.

A compile-time loop count controls the delay. The exact duration depends on CPU clock and generated code, so the displayed duration is approximate.

## Hardware-Confirmed Execution Flow

The following complete path has been observed on hardware:

```text
Stage 2
  |
  +--> load EXE2 at 0x40000000
  +--> load DPOSLAB at 0x42E70000
  +--> enter DPOSLAB
          |
          +--> validate handoff and stock opcodes
          +--> install veneer and call-site patch
          +--> enter EXE2 at 0x40000000
                  |
                  +--> stock reset/runtime startup
                  +--> stock main()
                  +--> patched FPGA_Start call site
                          |
                          +--> DPOSLAB wrapper
                          +--> original FPGA_Start()
                          +--> DPOSLAB monitor
                          +--> stock continuation at 0x4003473C
                                  |
                                  +--> remaining stock initialization
                                  +--> operational oscilloscope UI
```

A diagnostic halt version displayed:

```text
FPGA_Start RETURNED        PASS
HOOK COUNT                 0x00000001
FPGA HOOK HIT              CONFIRMED
```

An auto-continue version subsequently resumed the stock firmware and reached a working oscilloscope UI.

## Reliability and Safety Rules

### Preserve the real EXE2 startup

Always enter stock EXE2 through `0x40000000`. Do not use `0x40034700` as a cold entry.

### Validate original instructions

Before applying a patch, verify that the target site still contains the expected original opcode. Refuse to patch if it does not match.

For the v1 FPGA hook:

```text
site:            0x40034738
expected opcode: 0xEB005292
meaning:         BL 0x40049188
```

This protects against binary-version mismatch and accidental double patching.

### Coordinate veneer allocation

Do not assume that a veneer slot is globally free. DPOSDBG and DPOSLAB can use the same general pool, but two active resident systems must not overwrite the same slot.

### Minimize observer effect

Prefer passive reads for boundary snapshots. Clearly separate:

```text
PASSIVE OBSERVATION
```

from:

```text
ACTIVE FPGA PROBE
```

An active FPGA selector/read transaction can alter PE configuration, bus direction, selector state, and FPGA protocol state.

### Preserve processor state

A call-site wrapper must preserve enough state to make the intercepted call indistinguishable from the original call when stock execution resumes.

Future internal breakpoints have stricter requirements than the current whole-function wrapper and may need to preserve:

- all GPRs
- complete CPSR state
- exact ARM/Thumb state
- stack alignment
- VFP registers and status
- interrupt-mask state
- displaced original instructions

### Avoid unsafe driver initialization

Do not initialize touch, timers, interrupts, or unrelated peripherals merely to implement debugger control. Doing so can change the hardware state under investigation.

## Build and Deployment

The project build order matters. The main Stage 2 build must run before application builds because the main `make clean` also removes application build artifacts.

Typical sequence:

```fish
make clean
and make
and make -C apps/dposlab DPOSLAB_MODE=0
```

The produced resident application is:

```text
build/apps/dposlab/DPOSLAB.BIN
```

The DPOSLAB boot target expects the corresponding 8.3 filename configured by `loader/dposlab.h`.

Before deployment, useful verification commands include:

```fish
git diff --check

arm-none-eabi-nm -n build/apps/dposlab/DPOSLAB.elf \
    | rg 'dposlab_(capture|fpga|gpr|runtime)'

arm-none-eabi-objdump -d build/apps/dposlab/DPOSLAB.elf \
    | sed -n '/<dposlab_fpga_start_hook>/,+60p'
```

## Source-of-Truth Baseline

The first hardware-confirmed resident runtime monitor baseline is preserved by:

```text
Tag:    dposlab-runtime-monitor-v1
Commit: 423ba78414f076889546917ed310f91496034908
```

The v1 baseline confirms the resident hook, original `FPGA_Start()` call, return to DPOSLAB, and successful stock resume.

Subsequent BEFORE/AFTER snapshot development belongs to the follow-up branch:

```text
experiment/dposlab-runtime-v2
```

## Current Limitations

DPOSLAB is not yet a conventional interactive debugger.

Current limitations include:

- no JTAG integration
- no hardware single-step
- no interactive physical continue button
- no safe touch-based control path
- no generic breakpoint allocator
- no automatic instruction relocation
- no VFP-context snapshot
- no generic ARM/Thumb mixed-state breakpoint engine
- no persistent trace export from the runtime monitor
- photographic delays are approximate busy loops

Despite these limitations, DPOSLAB already provides the critical primitive required for targeted runtime reverse engineering:

```text
stock execution
    -> resident diagnostic code
    -> original stock function
    -> resident observation
    -> restored stock execution
```

## Planned Evolution

Recommended next steps:

1. Stabilize and document the whole-function BEFORE/AFTER snapshot.
2. Add compact multi-page register views.
3. Add explicit passive-versus-active probe labeling.
4. Generalize hook descriptors while retaining opcode validation.
5. Add multiple timed breakpoints in one boot.
6. Introduce internal `FPGA_Start()` checkpoints around confirmed phases.
7. Capture transient CCU and CSI state inside the function.
8. Preserve and display complete CPU context where internal hooks require it.
9. Add an optional persistent trace buffer or FAT export.
10. Keep every hardware-confirmed tool milestone tagged before deeper experiments.

## Evidence Classification

DPOSLAB findings should use explicit evidence labels in project documentation:

- **HW-confirmed:** observed directly on the physical device and reproducible.
- **Runtime observation:** captured by DPOSLAB at a defined execution boundary.
- **Statically confirmed:** established from disassembly or binary structure.
- **Hypothesis:** plausible interpretation not yet confirmed on hardware.

This distinction is especially important when interpreting temporary clock state, CSI register meaning, FPGA protocol transactions, and internal `FPGA_Start()` phases.

