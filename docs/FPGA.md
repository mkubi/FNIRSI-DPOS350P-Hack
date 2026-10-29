# FPGA Documentation Policy (2026-10-24)

> **Canonical fact/reference document.** The complete previous FPGA reverse-engineering record is preserved below verbatim so no technical detail, address, mapping, pseudocode, historical correction, or milestone is lost.
>
> New hardware experiments are recorded in `FPGA-EXPERIMENTS.md`. Current hypotheses and active analysis targets are recorded in `FPGA-WORKLOG.md`. When an older statement below conflicts with a later milestone, the later `CONFIRMED` statement wins.

## Evidence classes used going forward

- **HW-CONFIRMED**: reproduced on physical DPOS350P hardware.
- **STATIC-CONFIRMED**: unambiguous local EXE2 disassembly/data flow.
- **DERIVED**: direct consequence of confirmed facts.
- **OPEN**: unresolved question with no asserted semantic meaning.

## Post-snapshot corrections and confirmed hardware results

The following results were obtained after the 2026-10-23 snapshot and therefore supersede the relevant implementation assumptions later in the preserved document.

### HW-CONFIRMED: diagnostic FPGA ID cycle

Known-good Custom App diagnostic cycle repeatedly returns:

```text
RAW5       00 A5 A5 5A 5A
ID         0xA5A55A5A
PRESENT    YES
```

The known-good diagnostic selector uses the previously validated PE control sequence. An A/B hardware test showed that the isolated stock-decoded command cycle cannot simply replace this diagnostic selector in the current Custom App startup state. This does **not** disprove the stock primitive in the correctly initialized original application.

### HW-CONFIRMED: local CSI bring-up

Custom App stages confirmed:

```text
CSI bus clock/reset preparation     PASS
CSI module clock / PLL readiness    PASS
CSI PE pinmux                       PASS
CSI BUF0 write/readback              PASS
CSI geometry write/readback          PASS
timeout-protected capture disable    PASS
memory guards                        PASS
```

No Custom App test has yet observed CSI completion or changed capture bytes.

## HW-confirmed FPGA_Start runtime boundary

DPOSLAB resident runtime instrumentation can now intercept the stock
`FPGA_Start()` call at `0x40034738`, execute the original function at
`0x40049188`, capture state immediately after return, and resume stock
execution at `0x4003473C`.

The experiment was repeated across three cold boots and produced
identical hardware-register observations.

### CPU / ABI

Before:
- CPSR: `0x60000153`
- stock SP: `0x42FE8D20`

After:
- CPSR: `0x20000153`
- changed GPR mask: `0x0000100F`

The changed GPR mask corresponds to `r0-r3` and `r12`.
`r4-r11` were preserved across `FPGA_Start()`.

The DPOSLAB FPGA execution stack was `0x42FE8CE8`, exactly 0x38
(56) bytes below the stock SP, matching the resident wrapper's
`push {r0-r12, lr}` frame.

### PE state

Before -> After:

- `PE_CFG0`: `0x77777777` -> `0x77771111`
- `PE_CFG1`: `0x77777777` -> `0x00000000`
- `PE_DATA`: `0x01400000` -> `0x01700001`
- `PE_PULL0`: `0x00000000` -> `0x00000000`

This confirms that `FPGA_Start()` establishes the stock PE
parallel-control idle state before returning:
PE0..PE3 are GPIO outputs, PE8..PE15 are GPIO inputs, and PE0 is
high while PE1 is low.

### CSI state

Before -> After:

- `CSI_IF_CFG`: `0x00000000` -> `0x01050400`
- `CSI_MODE`: `0x00000000` -> `0x00000000`
- `CSI_HSIZE`: `0x00000000` -> `0x00000A00`
- `CSI_DEST(+0x50)`: `0x00000000` -> `0x10E7DDB5`
- `CSI_IRQ_ENABLE`: `0x00000000` -> `0x00000002`
- `CSI_STATUS`: `0x00000000` -> `0x00000000`

Therefore `FPGA_Start()` configures a reproducible subset of the CSI
block before returning, but the observed MODE and STATUS registers
remain zero at the function boundary.

The `0x10E7DDB5` value observed at CSI+0x50 is reproducible across
all three runs but is not yet interpreted as a DMA destination address.

### CCU boundary observation

- `CCU_GATE0`: `0x00104100` -> `0x00104100`
- `CCU_RESET0`: `0x00104100` -> `0x00104100`

No net change was observed in these two CCU registers across the whole
`FPGA_Start()` call. This does not exclude transient clock/reset changes
inside the function.


### STATIC-CONFIRMED: current acquisition-start ordering

Runtime bank loop around `0x40041524` performs:

```text
bank_select(position + block)
acquisition_prepare()
0x4003FCC0
readiness/state check
0x4003FAE4
0x4003FB90
```

`0x40040638(R0,R1)` preserves `R0/R1` and performs:

```text
0x4003FAB8
FPGA register 0x1F <- R0
0x40049094(1)
0x40066030(R1)
0x40042C40(0)
0x400660AC()
0x4003F9F8()
```

Thus the next implementation/debug step must not treat isolated `0x15` and `0x08` writes as the complete stock acquisition start sequence.

---

### HW-CONFIRMED minimal runtime prerequisite for FPGA_Start

DPOSLAB minimal-runtime A/B testing isolated a previously missing prerequisite
for the stock `FPGA_Start()` path.

The beginning of stock `main()` is:

    0x40034708  BL 0x4004282C   sys_clock_init(1152000000)
    0x4003470C  BL 0x40042A60   sys_mmu_init()
    0x40034710  BL 0x40042D28   System_Init()
    ...
    0x40034738  BL 0x40049188   FPGA_Start()

`System_Init()` decomposes into:

    System_Init @ 0x40042D28
        |
        +-- BL 0x40042C94     Part 1
        |
        `-- B  0x40042CD8     Part 2

DPOSLAB intercepted execution at `0x40034710`, after `sys_clock_init()` and
`sys_mmu_init()` but before `System_Init()`. A hook-only control completed
successfully, confirming that this interception point is reached and remains
usable without executing either part of `System_Init()`.

A minimal stock path was then tested with only `System_Init` Part 2 enabled:

    sys_clock_init
      -> sys_mmu_init
      -> System_Init Part 2 @ 0x40042CD8
      -> stock FPGA_Start
      -> stock SMEC Job 1
      -> stock SMEC Job 2

`System_Init` Part 1 at `0x40042C94` and all normal application initialization
were skipped.

This path completed successfully:

    System_Init Part 1      SKIPPED
    System_Init Part 2      PASS
    stock FPGA_Start        PASS
    Job 1 result66          0x01010101
    Job 2 result66          0x03030303

A direct negative-control build was then made from the known-good image.
The only functional change was removal of the call to:

    0x40042CD8

With Part 2 disabled, the subsequent stock `FPGA_Start()` path no longer
completed.

**HW-CONFIRMED conclusion:**

`System_Init` Part 2 at `0x40042CD8` is a prerequisite for the tested minimal
stock `FPGA_Start()` path.

`System_Init` Part 1 at `0x40042C94` is not required for this path.

The current minimum confirmed stock dependency chain is therefore:

    sys_clock_init
        -> sys_mmu_init
        -> System_Init Part 2
        -> FPGA_Start

Static analysis of Part 2 shows:

    0x40042CD8
        +-- 0x40042D40
        `-- 0x40042D4C

Both paths use the common implementation at `0x40042D54`, with observed MMIO
accesses including:

    0x01C20144
    0x01C20C8C

and constants:

    24000
    24

**INTERPRETATION:** Part 2 strongly resembles initialization of timer/timebase
resources. The exact peripheral semantics and the specific dependency inside
`FPGA_Start()` are not yet proven.

#### Implementation consequence

A standalone `FPGA_Start()` reproduction must not begin from the assumption
that `sys_clock_init()` and `sys_mmu_init()` alone establish the required stock
platform state.

The equivalent state produced by `System_Init` Part 2 must also be reproduced
before judging a custom `FPGA_Start()` implementation against stock behavior.

This is now a leading explanation for the earlier Custom App result where the
FPGA control path and Job 1 worked but Job 2 did not reproduce stock behavior.


## FPGA_Start runtime phase decomposition

DPOSLAB runtime tracing now provides automatic passive snapshots at selected
stock call boundaries. Records are accumulated in resident RAM and exported
manually as `DPOSLAB.TRC` through the stock Screenshot path.

The following observations are HW-confirmed.

### Confirmed execution timeline

Current checkpoints:

```text
FPGA_START_ENTRY
  0x40049188
      |
      | 0x40049194: BL 0x400660C8
      |
      |    0x400660C8(8000)
      |        |
      |        | 0x400660D4: BL 0x40065F48
      |        |
      |        +-- CSI_AFTER_CLOCK_RESET
      |            continuation 0x400660D8
      |
      +-- FPGA_EARLY_INIT
          continuation 0x40049198
              |
              | 0x400491A8: BL 0x40049094
              |
              +-- FPGA_PHASE_A
                  continuation 0x400491AC
                      |
                      +-- remaining FPGA_Start chain
                      |
                      +-- FPGA_START_RETURN
                          continuation 0x4003473C
```
Phase 1: clock/reset and initial CSI state

Immediately after stock 0x40065F48:

Plain Text
CSI_IF_CFG 0x00000000 -> 0x01050000
CSI_HSIZE 0x00000000 -> 0x00300200

The returned register state also contains:

Plain Text
r0 = 0x01C20134
r1 = 0x81008000

Static analysis of 0x40065F48 confirms that this routine performs read/modify/write operations around the CCU region near 0x01C20100, waits for a status/lock bit, and finishes with a write using the 0x81008000 value.

Therefore 0x40065F48 is part of the CSI clock/reset bring-up and, by its return boundary, the CSI block already exposes an initial configuration state.

The precise mechanism by which the initial CSI register values arise still requires decomposition of the helper calls inside 0x40065F48.

Phase 2: full early CSI initialization

Between CSI_AFTER_CLOCK_RESET and the return from 0x400660C8:

Plain Text
PE_CFG0 0x77777777 -> 0x77772272
PE_CFG1 0x77777777 -> 0x22222222
 
CSI_IF_CFG 0x01050000 -> 0x01060400
CSI_HSIZE 0x00300200 -> 0x00000A00
CSI_DESTINATION 0x00000000 -> 0x10B7DDB5
CSI_IRQ_ENABLE 0x00000000 -> 0x00000002

Thus 0x400660C8 is confirmed as an early CSI initialization orchestrator.

Its statically confirmed call chain is:

Plain Text
0x400660C8(8000)
|
+-- 0x40065F48(0)
+-- 0x400658BC(0, 0x01CB4000)
+-- 0x40065C68(0)
+-- 0x40065C64(0)
+-- 0x40065FA8(8000)
|
+-- tail 0x40066054

0x400658BC(0, 0x01CB4000) stores the CSI base into the instance table at 0x42E68C50.

0x40065FA8(8000) initializes stock CSI software/configuration structures before invoking lower-level configuration routines.

The exact ownership of the final IF_CFG, HSIZE, destination and IRQ values will be determined by additional runtime checkpoints between these calls.

Phase 3: transition from CSI pinmux to FPGA GPIO control bus

Immediately after 0x400660C8:

Plain Text
PE_CFG0 = 0x77772272
PE_CFG1 = 0x22222222

This represents the CSI-side pinmux state.

After the subsequent call:

Plain Text
0x400491A8: BL 0x40049094

the state becomes:

Plain Text
PE_CFG0 0x77772272 -> 0x77771111
PE_CFG1 0x22222222 -> 0x00000000
PE_DATA 0x01400000 -> 0x01400001

The CSI configuration registers remain unchanged across this transition.

This HW-confirms that 0x40049094 does not establish the CSI configuration. Instead, it transitions the shared PE interface from CSI pinmux mode into the parallel FPGA GPIO control-bus idle state.

The resulting state is:

Plain Text
PE0..PE3 GPIO output
PE8..PE15 GPIO input
PE0 high
PE1 low
Remaining FPGA_Start phase

From FPGA_PHASE_A to FPGA_START_RETURN, the currently monitored CSI configuration registers remain stable.

The only monitored PE change is:

Plain Text
PE_DATA 0x01400001 -> 0x01700001

Because PE8..PE15 are inputs in this phase, their bits in PE_DATA must not be interpreted simply as output latch values.

The remaining phase contains the previously identified clock/configuration chain including:

Plain Text
0x400490B8
+-- 0x4005F1BC
+-- 0x4005F1F0
+-- tail 0x4005F38C

Internal state changes in this later chain have not yet been fully traced.

Implication for the custom FPGA_Start reproduction

The stock initialization order is now known to be more complex than the original custom PE/FPGA handshake reproduction.

HW-confirmed ordering:

Plain Text
CSI clock/reset bring-up
|
initial CSI register state
|
full CSI configuration
|
CSI PE pinmux
|
transition to FPGA GPIO control-bus mode
|
remaining FPGA clock/configuration chain

Therefore a custom reproduction consisting only of PE GPIO setup and the FPGA ID handshake is incomplete.

The final custom implementation should not yet hard-code the observed boundary values. The remaining internal CSI checkpoints should first identify the minimal stock sequence and required ordering.

### Runtime-trace checkpoint (2026-10-25)

**HW-CONFIRMED:** Resident DPOSDBG now runs at `0x42E70000`, outside the stock EXE2 zero-init range `0x400954A4..0x42E68D23`. Earlier payloads at `0x41000000` and `0x41700000` were confirmed cleared by stock startup; a 64-byte sentinel at `0x42E70000` survives unchanged.

The stock application remains operational through the resident hook path: acquisition, UI and repeated screenshots survive, while DPOSDBG records runtime events and exports `/DPOSDBG.TRC` through the stock filesystem API.

**HW-CONFIRMED primitive trace:**

```text
0x4004927C  FPGA selector/command write
0x40049214  FPGA data-byte write
0x400491B8  FPGA data-byte read
```

The observed repeating CPU↔FPGA transaction is:

```text
SEL   0x69
WRITE 0x07
SEL   0x6A
SEL   0x6B
WRITE <input byte 0>
WRITE <input byte 1>
WRITE <input byte 2>
SEL   0x68
WRITE 0x01
SEL   0x64
READ  <status>
SEL   0x68
SEL   0x65
READ repeatedly until bit 0 becomes set
SEL   0x68
SEL   0x67
READ  <one byte>
SEL   0x66
READ  <four bytes>
```

The transaction repeats with changing input and return values. Therefore the ARM software continuously participates in an FPGA-side asynchronous job/service protocol; reproducing only the FPGA ID transaction is insufficient evidence of the complete operational state.

**HW-CONFIRMED polling:** the path at `0x4004A04C/0x4004A050` repeatedly selects `0x65` and reads until bit 0 becomes set. DPOSDBG event `0x1010` (`POLL65_DONE`) compresses this loop into the final value and the count of preceding unsuccessful reads. Many observed transactions require about 3000 reads; one captured example completed after 3070 bit-clear reads with final value `0x01`. Other final values include `0x03`, `0x05`, `0x07` and `0x17`, so the returned byte carries state beyond the confirmed ready bit.

**Terminology correction:** `SMEC_Init @ 0x4004A15C` remains the historical working name, but current static cross-checks and runtime evidence identify it as part of a broader FPGA full-state/register-file initialization path. Commands `0x64..0x6B` are confirmed at the ARM↔FPGA boundary; they must not be treated as proven physical commands of the external 8-pin device.

DPOSDBG observes:

```text
ARM/R11 CPU ↔ GOWIN FPGA
```

not directly:

```text
GOWIN FPGA ↔ external 8-pin device
```

The runtime tracer can therefore reconstruct CPU selectors, written bytes, returned bytes, call sites, ordering, polling behavior and higher-level transaction framing without claiming the FPGA-to-device encoding.

The locally confirmed acquisition architecture remains:

```text
CONTROL PLANE
ARM/R11
    │ PE GPIO mode
    ▼
GOWIN FPGA
    ├── command/register protocol
    ├── acquisition setup
    └── repeating 0x64..0x6B service transactions

SAMPLE PLANE
GOWIN FPGA
    │ PE CSI function 2
    ▼
R11 CSI
    ▼
0x42DF76D4 + bank * 0x2134
```

The externally cross-checked MS5351M path on PE20/PE21 remains separate from the probable FPGA-side two-wire traffic near the unmarked 8-pin device.

#### Consequence for standalone Custom App acquisition

The previous isolated Custom App proved FPGA bus access, FPGA ID and local CSI setup, but did not complete acquisition. Runtime tracing now identifies a concrete missing subsystem: broader FPGA state initialization plus repeating `0x64..0x6B` asynchronous job/service transactions. The exact relationship of that protocol to acquisition remains under active test.

This is a leading explanation for the failed acquisition experiment, but a causal dependency on CSI waveform emission still requires an active hardware test.

#### Revised next step

Do not immediately repeat the isolated CSI test. First aggregate complete stock service transactions into compact records containing:

```text
input value written through 0x6B
status returned through 0x64
final 0x65 value
0x65 polling count
byte returned through 0x67
32-bit value returned through 0x66
```

Then identify the producer/state relationship of the changing `0x6B` input and compare transactions across normal, stopped, FFT and FRA modes. Only after the protocol's role and any acquisition dependency are established should a minimum required job/service sequence be reproduced in the standalone Custom App and the single-bank CSI smoke test repeated.

Priority static region:

```text
0x40049EC4
0x40049F60
0x40049F7C
0x40049FB8
0x40049FDC
0x4004A020
0x4004A040
0x4004A064
0x4004A080
0x4004A0B0
```

A stock-assisted fallback is now also viable: let stock EXE2 perform the complete FPGA/service initialization while resident DPOSDBG supplies custom tracing and later custom functionality.

## Independent hardware-variant cross-check (2026-10-25)

The following observations were reported from a different physical DPOS350P unit and are **EXTERNAL CROSS-CHECK** unless independently reproduced locally.

### Reported hardware variant

```text
Local SPI baseline:
    Winbond-compatible NOR, JEDEC EF 40 15

Externally reported unit:
    FPGA    GW1N-LV9EQ176C6/I5
            marking 2543C / PE7A98.00
    NOR     Zbit ZB25VQ16A
            JEDEC manufacturer 0x5E
```

DPOS350P units must not be assumed to use one FPGA package or one SPI-NOR manufacturer. Package-specific JTAG/configuration pin numbers from the reported EQ176 unit must not be transferred to the local unit until its exact FPGA package marking is re-checked.

### FPGA control-bus and ID cross-check

The external analysis independently reports the same ARM-to-FPGA control plane: PE8..PE15 data, PE0 strobe, PE2 read/write, PE3 command/data, and FPGA ID `0xA5A55A5A`. It additionally reports FPGA command `0x02` as a status byte whose bit 0 drives the charging indication and command `0x03` as power off. The two command meanings remain **EXTERNAL CROSS-CHECK** until locally verified.

### FPGA configuration-source cross-check

The external analysis reports no GOWIN bitstream/configuration image in its NOR dumps, the 2025-06-19 updater image or the V3s BootROM, and no SoC-side FPGA-configuration path in the inspected SPI code. This is consistent with the local result that FPGA ID is already available through the custom `BootROM -> Stage 1 -> Stage 2 -> Custom App` chain.

**DERIVED:** No FPGA-configuration programming path has been identified on the examined SoC/NOR firmware side, and the FPGA is already configured sufficiently to answer its diagnostic ID before stock EXE2 runs.

**OPEN:** This does not prove the physical source of FPGA configuration.

### Zbit NOR flashing warning

On the external Zbit `ZB25VQ16A` unit, `xfel` reportedly completed writes without actually programming the intended contents, while reads still worked. `sunxi-fel spiflash-write` was reported working on that unit. This does not change the local Winbond `EF 40 15` baseline.

**Project safety rule:** A flash-tool success message is not proof of programming. After any NOR write, perform an independent readback and verify the programmed range or full-image hash. Treat `xfel` writes on Zbit/manufacturer `0x5E` units as externally reported unsafe until independently verified.

### FEL fallback without a physical jumper

The external unit reportedly enters BootROM FEL after sending NOR deep-power-down command `0xB9` and triggering a watchdog reset. BootROM reportedly does not wake the flash with `0xAB`; host-side flash access therefore needs an explicit wake before subsequent `spiflash-*` operations. A full power cycle restores normal flash response. This is a future recovery option, not part of the current acquisition path.

### Battery and clock cross-checks

The external unit reports a single-cell 3.7 V / 8000 mAh / 29.6 Wh battery, V3s 6-bit LRADC measurement and firmware thresholds beginning around 4150 mV. Charging itself is reported as not firmware-controlled.

The same analysis independently confirms the MS5351M path on PE20/PE21 at 7-bit address `0x60`, further separating the R11-controlled clock-generator bus from the distinct I2C-like traffic observed near the unmarked 8-pin device.

### Current cross-project convergence

```text
ARM/R11
    +-- PE GPIO parallel control plane -> GOWIN FPGA -> ID 0xA5A55A5A
    +-- PE CSI function 2 <- FPGA waveform/sample stream
    +-- PE20/PE21 bit-banged I2C -> MS5351M @ 0x60

GOWIN FPGA
    +-- probable separate two-wire link -> unmarked 8-pin device
```

The reported hardware differences currently look like revision/package/vendor differences rather than contradictions of the shared FPGA control, clock and acquisition architecture.

### Documentation consequences

1. Keep local `HW-CONFIRMED` facts distinct from external-revision reports.
2. Do not present one SPI-NOR vendor or FPGA package as universal across all DPOS350P units.
3. Keep Winbond `EF 40 15` as the local bootloader SPI/JEDEC baseline.
4. Require readback/hash verification after NOR writes across revisions.
5. Treat FPGA command `0x02` charging status and command `0x03` power-off as local verification targets.
6. Treat the absence of a discovered SoC-side bitstream as evidence, not proof of the FPGA configuration source.

### Independent secure-device cross-check

Independent hardware work by Marco Menti (`mmame`) strongly identifies the
unmarked SOP-8 device near the FPGA as a Sinormous SMEC88SP-class
programmable secure IC.

The supporting physical evidence is documented in [hardware.md](hardware.md)
and includes:

- an SOP-8 pinout closely matching the reported SMEC88SP pinout;
- SDA/SCL with 3.3 kOhm pull-ups on the nearby four-pin header;
- I2C-style traffic at approximately 2 MHz;
- an approximately 2.35 s fast startup phase;
- steady write/read traffic at approximately 2.79 ms cadence;
- the stock POST label `加密芯片检测 ...` ("encryption chip check").

The literal string `SMEC` has not been identified in the stock firmware or
SPI flash. The historical project name `SMEC_Init` and related terminology
must therefore not be treated as a stock firmware symbol for the physical
device.

Local DPOSDBG runtime tracing independently establishes the ARM-visible side
of this subsystem:

```text
ARM/R11
    |
    | selectors 0x64..0x6B
    | command/input/status/result protocol
    v
GOWIN FPGA
    |
    | probable two-wire service link
    v
SMEC88SP-class secure-device candidate
```

### Independent runtime/acquisition cross-check (2026-10-25)

A separate DPOS350P reverse-engineering effort by Adam Ivie (`Line-of-Sight-Services/FNIRSI-DPOS350P-RemoteControl`) independently reconstructs the same FPGA control primitives and has verified raw acquisition-ring readout on physical hardware. The source repository is private because it also contains per-unit firmware/calibration material; only the technical findings summarized here are incorporated.

#### EXTERNAL HW-CONFIRMED: CSI active payload and channel grouping

On the 2025-06-19 build, the external project uses the same locally known CSI/ring addresses:

```text
CSI base       0x01CB4000
CSI BUF0       0x01CB4050
ring base      0x42DF76D4
slot stride    0x2134 = 8500 bytes
slot count     8
active payload 0x1800 = 6144 bytes per captured slot
```

Hardware tests with a 1 kHz generator report:

```text
CH1 only:
    slots 0..7 contain CH1 data

CH1 + CH2:
    slots 0..3 contain CH1 data
    slots 4..7 contain CH2 data
```

Each active slot contains 6144 raw 8-bit samples. Different slots exhibit different zero-code offsets; an observed example included values around `166/162/151/151`. This is consistent with multiple ADC/interleave cores requiring per-slot calibration, but the exact ADC-core ordering and fast-timebase interleave order remain open.

This resolves two previous practical unknowns for at least the externally tested acquisition modes:

- the active sample payload is 6144 bytes inside each 8500-byte logical slot;
- bank groups `0..3` and `4..7` correspond to CH1 and CH2 respectively when both channels are enabled, while single-channel CH1 can use all eight slots.

The remaining `8500 - 6144 = 2356` bytes per slot must not yet be assigned a fixed meaning; allocation, ring-origin headroom, guard space, alignment and other runtime-layout explanations remain possible.

#### EXTERNAL STATIC-CONFIRMED: independent primitive-address match

For local build `8eeb56ba0086`, the external symbol map independently identifies:

```text
FPGA_CMD       0x4004927C
FPGA_WRITE     0x40049214
FPGA_READ      0x400491B8
FPGA_READ_RET  0x40049210
RING           0x42DF76D4
```

These match the locally reconstructed primitive addresses and raw-ring base. Its separate factory/reference build uses the same structures at shifted addresses, including ring base `0x42DF76EC`; firmware-build context must therefore accompany absolute RAM/code addresses.

#### Interpretation correction: `0x64..0x6B` is an asynchronous FPGA job/service protocol

Local DPOSDBG hardware tracing proves the transaction shape and repeated runtime use of commands `0x64..0x6B`, but it does not by itself prove the backend purpose of the transaction.

The independent static analysis reaches the same command family and describes its shape as:

```text
write job/parameter inputs
start operation
poll 0x64 / 0x65 ready state
read status / 32-bit result
```

However, that project currently interprets the family as a possible measurement engine used by frequency/FFT/FRA-related paths. That semantic label is explicitly inferred rather than hardware-confirmed.

Therefore the current neutral terminology is:

```text
0x64..0x6B = ARM-visible asynchronous FPGA job/service protocol
```

Confirmed:

- ARM submits command/input state through FPGA registers;
- completion is polled through status/ready bits;
- a result/status value is returned;
- the protocol is active repeatedly at runtime.

Not yet established:

- whether all transactions target the external secure-device subsystem;
- whether some or all transactions implement measurement/FFT/FRA work;
- whether this is a general FPGA job engine shared by multiple subsystems;
- the byte-level relation between this ARM-visible protocol and the physical two-wire SMEC88SP-candidate traffic.

The previous wording that treated the repeating `0x64..0x6B` activity as specifically a secure-device service loop is therefore too narrow and is superseded by this section.

#### Independent runtime tracer cross-check

The external project also implements an independent FPGA primitive tracer which hooks command, write and read-return primitives and records `(kind, byte, main-loop counter)` into a large DRAM ring. Its implementation includes repeated-read compression, quiet-command counters, hook validation, cache maintenance and emulator tests against both firmware builds.

At the time of the supplied snapshot, that tracer is documented as emulator-tested but not yet run on the physical scope. Therefore local DPOSDBG remains the hardware-confirmed primitive tracer, while the external implementation provides useful independent design validation and a larger-ring/session methodology.

A companion session tool is designed to collect idle traffic and then change one UI variable at a time, including V/div, coupling, timebase, trigger settings, generator settings, FFT and FRA screens. This provides a direct experiment for distinguishing background service traffic from measurement-mode-specific `0x64..0x6B` activity.

#### Additional externally mapped FPGA controls

The external static/RAM-diff work reports these useful mappings:

```text
0x02  status read; bit 0 = charging indication
0x03  power off

0x28  CH1 V/div / position apply path
0x29  CH1 bandwidth-derived filter
0x2B  CH2 V/div / position apply path
0x2C  CH2 bandwidth-derived filter
0x2F  CH1 coupling
0x30  CH2 coupling

0x10  trigger level
0x15  trigger channel
0x16  trigger edge
0x1A  trigger mode
```

These mappings are valuable implementation leads but remain **EXTERNAL CROSS-CHECK** in this document until locally reproduced or independently verified against the local EXE2/hardware.

#### Revised immediate experiment

The next DPOSDBG transaction-level capture should answer a more neutral question than the earlier secure-service hypothesis:

```text
Is 0x64..0x6B primarily:
    A. secure-device service,
    B. measurement/FFT/FRA work,
    C. or a general FPGA asynchronous job engine used by both?
```

Capture and compare at minimum:

```text
normal scope running
normal scope stopped
FFT screen active
FRA screen active
frequency/measurement-related mode where available
startup/encryption-check interval where practical
```

For each completed transaction preserve:

```text
0x69 input/timing value
0x6A parameter/job ID or selector
full 0x6B input value and byte count
0x64 status
0x65 final value and polling count
0x67 returned byte/status
0x66 returned 32-bit value
```

The aggregator must not assume that `0x6B` always contains exactly three bytes; independent static analysis describes a path capable of writing up to four bytes.

# Preserved canonical reverse-engineering record

## FPGA state before original EXE2

### HW-CONFIRMED

The FPGA is already responsive without executing the stock BT0/SPL
or the original EXE2 initialization path.

Confirmed boot path:

BootROM
  -> custom Stage 1
  -> custom Stage 2
  -> Custom App

A standalone Custom App running through this entirely custom boot
chain successfully reproduces the FPGA diagnostic ID transaction:

RAW5 = 00 A5 A5 5A 5A
ID   = 0xA5A55A5A

Separately, the following boot path successfully runs the complete
original oscilloscope application:

BootROM
  -> custom Stage 1
  -> custom Stage 2
  -> original EXE2

Therefore the original BT0/SPL is not required for the FPGA to be
present and responsive to the ARM-side diagnostic interface.

This does not yet identify where the FPGA configuration originates
or prove how/when the FPGA bitstream is loaded.

### Interpretation boundary

This distinguishes:

1. FPGA configuration / bitstream availability
2. FPGA runtime initialization
3. FPGA acquisition/CSI streaming initialization

The diagnostic FPGA ID being available before EXE2 proves (1) is
already satisfied, but does not prove that (2) or (3) have been
performed.

# FPGA Subsystem Reverse-Engineering Notes

**Project:** FNIRSI DPOS350P
**Snapshot date:** 2026-10-23
**Source image:** `work/updater-re/exe2-from-backup.bin`
**Expected load address:** `0x40000000`
**Source SHA-256:** `8eeb56ba00865d7bacde2d56c006c57cc8284d22bd26b24fcf0e3cf4defbca19`

This document is the canonical milestone snapshot of the current reverse-engineering state. It deliberately separates facts established by static analysis or hardware experiments from interpretations and open hypotheses.

## Evidence labels

- **CONFIRMED:** Directly established by disassembly, data flow, MMIO access, known working custom-driver definitions, or a reproducible hardware experiment.
- **STRONG INTERPRETATION:** Supported by multiple independent observations, but not yet proven end-to-end.
- **HYPOTHESIS:** Plausible working model that still needs static or hardware confirmation.
- **UNKNOWN:** Insufficient evidence for a reliable interpretation.

---

## 1. Hardware overview

### Confirmed components

- Allwinner R11/V3s-family SoC.
- GOWIN GW1N-9 FPGA.
- External MS5351M clock generator with a 25 MHz reference.
- GT911 touch controller.
- Unmarked device near the FPGA, associated with a four-pin header carrying I2C-like traffic.

### FPGA project modules visible in the developer screenshot

```text
adc.v
analog.v
clk_div.v
clk_inside.v
clk_sel.v
clock.v
gen.v
pll_dynamic.v
ram_adc.v
ram_dac.v
sample_trig.v
smec.v
top.v
trans.v
```

### Unknowns

- Exact identity and function of the unmarked device.
- Whether `smec.v` directly implements the physical interface to that device.
- Exact routing of the MS5351M clock outputs.
- Whether any FPGA bitstream protection is enabled.

---

## 2. Known board routing relevant to RE

### FPGA parallel interface

```text
PE0..PE3     control signals
PE8..PE15    DATA[7:0]
```

PIO registers used by the original firmware:

```text
PE_CFG0 = 0x01C20890
PE_CFG1 = 0x01C20894
PE_DATA = 0x01C208A0
```

The interface is an 8-bit parallel data bus with four control signals. It is not SPI.

### GT911 routing

**CONFIRMED from the working custom driver and static EXE2 analysis**

```text
GT911 INT    PE22
GT911 RESET  PE24
GT911 SDA    PG6
GT911 SCL    PG7
I2C address  0x5D
write byte   0xBA
read byte    0xBB
```

Known registers used by the custom driver:

```text
PRODUCT_ID   0x8140
STATUS       0x814E
POINT1       0x8150
```

### GT911 reset/address-selection timing

The working custom implementation uses:

```text
RESET low       5 ms
INT setup      100 ms
post-reset       5 ms
```

**CONFIRMED by hardware testing:**

- Product ID reads can work with shorter INT setup intervals.
- Touch `DATA_READY` coordinate reporting did not work below approximately 50 ms on this DPOS350P.
- Touch coordinate reporting started working from approximately 50 ms.
- `100 ms` was deliberately selected as a safe reliability margin.

Important: the 100 ms value is a conservative choice derived from custom-driver testing. It is not evidence that the original FNIRSI firmware itself uses exactly 100 ms.

### MS5351M routing known so far

```text
software I2C pins  PE20 / PE21
SCL / SDA          PE20 / PE21 (external cross-check; local re-verification pending)
I2C address        0x60
reference clock    25 MHz
configured clocks  95 MHz and 10 MHz
```

Physical output routing remains open. PE20=SCL and PE21=SDA are reported by an external independent analysis and are consistent with the reconstructed bit-banged I2C path; local instruction-level confirmation is still pending.

---

## 3. Original application startup flow

The developer screenshot shows this startup order:

```c
FPGA_Start();
GUI_Init();
GT911_Init();
Win_Init();
UI_ColorInit();
FatFsInit();
SMEC_Init();
PowerInit();
HorizontalInit();
VerticalInit();
HardCostTimeInfoInit();
Gen_Init();
CheckAndUpdateNewSystem();
FactoryCheck();
SysConfigExec();
```

Confirmed addresses in the original EXE2 image:

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

---

## 4. MCU-to-FPGA parallel interface

### CONFIRMED

The Allwinner communicates with the FPGA through the 8-bit bidirectional PE bus described above.

### Known firmware primitives

```text
0x400491B8  read one FPGA byte
0x40049214  write one FPGA data byte
0x4004927C  write FPGA command/selector
0x400492E4  perform FPGA identification probe
```

The low-level code accesses the data register as GPIO-port-base `+ 0x10`; therefore a standalone literal `0x01C208A0` need not occur in the binary.

The exact semantic name of each PE0..PE3 control line is still unknown.

---

## 5. FPGA identification transaction

### CONFIRMED by Custom App

A standalone Custom App loaded as `DPOSAPP.BIN` at `0x40000000` successfully reproduced the FPGA identification transaction without running the original oscilloscope application.

Sequence:

```text
write selector 0x01
read dummy/stale byte
read four identification bytes
```

Observed data:

```text
RAW5:      00 A5 A5 5A 5A
ID [0..3]: 0x00A5A55A
ID [1..4]: 0xA5A55A5A
FPGA PRESENT: YES
```

The first byte is treated as a dummy or stale read. The original firmware checks the following `A5 A5 5A 5A` pattern.

### Important boundary

The diagnostic Custom App reproduced only:

```text
FPGA bus initialization
+ FPGA ID handshake
```

It did **not** reproduce the complete original `FPGA_Start()`.

The minimalist bus initialization and ID handshake did not produce a relay click.

---

## 6. Complete `FPGA_Start()` structure

### CONFIRMED call graph

```text
FPGA_Start @ 0x40049188
 ├─ helper 0x400660C8(0x1F40)  [CONFIRMED R11 CSI acquisition initialization]
 ├─ set byte at 0x42E68D19 = 1
 ├─ helper 0x40049094(0)
 │   └─ configure FPGA parallel-bus direction/state
 └─ tail-call 0x400490B8
     ├─ 0x4005F1BC
     ├─ 0x4005F1F0(95,000,000)
     └─ tail-call 0x4005F38C(10,000,000)
```

Confirmed literal values:

```text
0x05A995C0 = 95,000,000
0x00989680 = 10,000,000
```

### Functional split

```text
FPGA_Start
 ├─ R11 CSI acquisition initialization at 0x400660C8(8000)
 ├─ FPGA PE-port GPIO/register-bus setup on PE0..PE15
 └─ external clock-generator setup on PE20/PE21
```

---

## 7. MS5351M clock-generator initialization

### CONFIRMED

Function `0x4005F1BC` configures PE20 and PE21 as GPIO outputs. The following low-level code bit-bangs a two-wire serial protocol on those pins.

The register-write primitive transmits:

```text
START
0xC0
register address
register value
STOP
```

`0xC0` is the 8-bit write form of seven-bit I2C address `0x60`.

The combination of:

```text
software I2C on PE20/PE21
I2C address 0x60
25 MHz reference
95 MHz requested output
10 MHz requested output
physical MS5351M on the PCB
```

confirms that this branch initializes the external MS5351M clock generator.

### Clock-setting functions

```text
0x4005F1F0(95,000,000)
    computes synthesizer parameters
    writes an eight-register block beginning at register 0x1A

0x4005F38C(10,000,000)
    computes synthesizer parameters
    writes an eight-register block beginning at register 0x22
```

### STRONG INTERPRETATION

The two register blocks correspond to two MS5351M multisynth/output paths, likely clock outputs 0 and 1.

### UNKNOWN

- Which physical MS5351M output carries 95 MHz.
- Which physical output carries 10 MHz.
- Which downstream device or FPGA block receives each output.
- Locally re-verify the externally reported PE20=SCL / PE21=SDA assignment at instruction level.

---

## 8. Original GT911 software-I2C implementation

### CONFIRMED

The code region around `0x40049350..0x400498E4` is the original GT911 software-I2C and register-access layer. It is **not** an FPGA waveform reader.

Evidence:

```text
PG6/PG7 bit-banging
MSB-first byte transfer
START / STOP / ACK handling
write byte 0xBA
read byte 0xBB
7-bit address 0x5D
one- or two-byte register addressing
buffer read/write helpers
```

Working function map:

```text
0x40049350  GT911 I2C start / bus prepare
0x40049428  GT911 I2C stop
0x40049490  GT911 wait ACK, timeout 50000
0x4004951C  GT911 send ACK/NACK
0x40049554  GT911 write byte, MSB-first
0x400495CC  GT911 read byte, MSB-first
0x40049688  GT911 register/buffer write
0x40049700  GT911 register/buffer read
0x400498A4  GT911 bus initialization wrapper
0x400498C4  GT911 16-bit-address write wrapper
0x400498D4  GT911 16-bit-address read wrapper
```

### Corrected transient interpretation

During exploratory analysis, the pin numbers `6` and `7` were temporarily described as PE6/PE7. That interpretation was wrong: the GPIO helper receives the pin number separately from the GPIO-port base pointer.

Correct routing is:

```text
PG6 = GT911 SDA
PG7 = GT911 SCL
```

This incorrect PE6/PE7 statement was never part of the previous canonical snapshot supplied by the user.

---

## 9. SMEC architecture

> **2026-10-25 terminology correction:** The historical `SMEC` naming in sections 9-17 describes the ARM-visible FPGA command/service subsystem. Runtime DPOSDBG tracing and the update-image cross-reference show that `SMEC_Init @ 0x4004A15C` is part of a broader FPGA full-state/register-file initialization path. Consequently, command IDs `0x64..0x6B` must not be documented as direct commands of the external secure IC unless the FPGA-to-device physical protocol is independently correlated.

### CONFIRMED

`SMEC_Init()` communicates entirely through the FPGA parallel read/write primitives. It uses FPGA command values in the range `0x64..0x6B` and implements:

- parameter IDs,
- 8-bit, 16-bit and 32-bit values,
- polling and ready flags,
- adaptive timeout values,
- retry logic,
- local caching,
- a final flush/commit-style transaction.

This is a runtime parameter subsystem, not merely a single startup check.

### STRONG INTERPRETATION

The architecture closely resembles the FNIRSI 1013D design:

```text
Allwinner MCU
    │
    │ parallel FPGA command bus
    ▼
GOWIN FPGA / smec.v
    │
    │ I2C-like physical interface
    ▼
unmarked external device
```

The final FPGA-to-device leg is not yet proven directly on DPOS350P.

---

## 10. `SMEC_Init()`

### CONFIRMED call flow

```text
SMEC_Init @ 0x4004A15C
 ├─ clear SMEC state at 0x42E62C2C, size 0x8FC
 ├─ initialize 255 timeout entries to 2000
 ├─ FPGA command 0x67, data 0x08
 ├─ FPGA command 0x69, data 0x07 0xD0
 └─ initialize SMEC parameters 0x01..0x07
```

Approximate pseudocode:

```c
void SMEC_Init(void)
{
    memset(smec_state, 0, 0x8FC);

    for (unsigned id = 0; id < 255; id++)
        smec_timeout[id] = 2000;

    fpga_write_command(0x67);
    fpga_write_data(8);

    fpga_write_command(0x69);
    fpga_write_data(0x07);
    fpga_write_data(0xD0);

    smec_load_initial_parameters();
}
```

---

## 11. SMEC FPGA commands

### CONFIRMED MCU-side behavior

```text
0x64  poll transaction-ready bit through FPGA read
0x65  poll another ready/write-ready bit through FPGA read
0x66  read a 32-bit big-endian result
0x67  read status bit 0; also written with initialization value 0x08
0x68  transaction control and one-byte read path
0x69  write a 16-bit big-endian timing/transaction value
0x6A  write an 8-bit parameter ID
0x6B  write a 32-bit big-endian parameter value
```

### Confirmed helper behavior

```c
uint32_t smec_read_u32(void)
{
    fpga_write_command(0x66);

    uint32_t value = fpga_read_byte();
    value = (value << 8) | fpga_read_byte();
    value = (value << 8) | fpga_read_byte();
    value = (value << 8) | fpga_read_byte();
    return value;
}
```

```c
void smec_write_parameter_id(uint8_t id)
{
    fpga_write_command(0x6A);
    fpga_write_data(id);
}
```

```c
void smec_write_value(uint32_t value)
{
    fpga_write_command(0x6B);
    fpga_write_data(value >> 24);
    fpga_write_data(value >> 16);
    fpga_write_data(value >> 8);
    fpga_write_data(value);
}
```

### Ready polling

One confirmed path performs:

```text
command 0x68, data 1
poll command 0x64 until read bit 0 becomes 1
command 0x68, data 0
```

Another helper polls command `0x65` until read bit 0 becomes 1.

---

## 12. SMEC transaction, retry and cache logic

### CONFIRMED

The transaction function is centered around `0x4004A0B0`.

Observed behavior:

- Per-parameter timing starts at `2000`.
- On failure it increases by `100`.
- It is limited to `3000`.
- The operation is retried up to four times.
- A 32-bit response is read.
- A status bit is checked.
- The response is compared against the requested parameter ID.

Approximate pseudocode:

```c
bool smec_transaction(uint8_t id, uint32_t value)
{
    for (unsigned attempt = 0; attempt < 4; attempt++) {
        do {
            smec_write_69(timeout[id]);
            smec_write_6a(id);
            smec_write_6b(value);

            smec_prepare_transaction();
            smec_wait_ready();

            uint32_t response = smec_read_u32();
            bool valid = smec_read_status();

            if (response == id && valid)
                return true;

            timeout[id] += 100;
        } while (timeout[id] <= 3000);

        timeout[id] = 3000;
    }

    return false;
}
```

### Local cache

The updater at `0x40049EC4` sends a parameter only when its value changed or the parameter has not yet been initialized.

SMEC state contains at least:

```text
state + 0x3FC + id*4   cached 32-bit value
state + 0x7F8 + id     initialized/dirty flag
state + 0x8F7          global transaction-active flag
```

### Flush

Function `0x40049F1C` performs a transaction with ID/value zero if any parameter changed, then clears the global active flag.

```c
void smec_flush(void)
{
    if (transaction_active) {
        smec_transaction(0, 0);
        transaction_active = 0;
    }
}
```

The exact semantic name of ID `0` is unknown. `commit`, `apply`, or `end-of-update` are interpretations only.

---

## 13. SMEC parameter-space map

### Confirmed subsystem ownership

```text
0x01..0x07  SMEC_Init / startup and global state
0x08..0x0D  VerticalInit / paired channel parameters
0x0E..0x13  SysConfigExec / common or trigger/acquisition configuration
0x14..0x18  Gen_Init / signal-generator parameters
0x19..0x23  HorizontalInit / horizontal and acquisition parameters
```

### Detailed current map

```text
ID     Size   Subsystem       Source / relation
-----  -----  --------------  ------------------------------------------
0x01   1      startup/global  app_state + 0x1B8
0x02   4      startup/global  app_state + 0x1C0
0x03   4      startup/global  app_state + 0x268
0x04   4      startup/global  app_state + 0x270
0x05   4      startup/global  app_state + 0x26C
0x06   2      startup/global  app_state + 0x008
0x07   4      startup/global  app_state + 0x62C

0x08   4      vertical        vertical table +0x00, channel-pair A
0x09   4      vertical        vertical table +0x04, channel-pair B
0x0A   4      vertical        vertical table +0x08, channel-pair A
0x0B   4      vertical        vertical table +0x0C, channel-pair B
0x0C   4      vertical        channel-A structure field +0x60
0x0D   4      vertical        channel-B structure field +0x120

0x0E   4      SysConfig       common/indexed configuration
0x0F   4      SysConfig       common/indexed configuration
0x10   2      SysConfig       signed/16-bit source field
0x11   4      SysConfig       common configuration
0x12   4      SysConfig       common configuration
0x13   4      SysConfig       common configuration

0x14   4      generator       generator-state +0x08
0x15   4      generator       generator-state +0x70
0x16   4      generator       generator-state +0x68
0x17   4      generator       generator-state +0x3C
0x18   4      generator       generator-state +0x6C

0x19   1      horizontal      app_state +0x674; FPGA command 0xA6
0x1A   1      horizontal      app_state +0x675; FPGA command 0xA7
0x1B   1      horizontal      app_state +0x676; FPGA command 0xA8
0x1C   1      horizontal      app_state +0x677; FPGA command 0xA9
0x1D   1      horizontal      app_state +0x678; FPGA command 0xAA
0x1E   1      horizontal      app_state +0x679; FPGA command 0xAB
0x1F   1      horizontal      app_state +0x67A; FPGA command 0xAC
0x20   1      horizontal      app_state +0x67B; FPGA command 0xAD

0x21   1      horizontal      app_state +0x6E4; FPGA command 0x04
0x22   1      horizontal      app_state +0x6E5; FPGA command 0x05
0x23   1      horizontal      app_state +0x6E6; FPGA command 0x06
```

---

## 14. Vertical subsystem

### CONFIRMED

`VerticalInit()` at `0x4004A50C` owns SMEC IDs `0x08..0x0D`.

The IDs form two interleaved channel groups:

```text
Channel group A: 0x08, 0x0A, 0x0C
Channel group B: 0x09, 0x0B, 0x0D
```

The first four values are stored at:

```text
0x42E53A0C + 0x00  SMEC 0x08
0x42E53A0C + 0x04  SMEC 0x09
0x42E53A0C + 0x08  SMEC 0x0A
0x42E53A0C + 0x0C  SMEC 0x0B
```

The third value for each channel comes from separate channel-specific structure fields.

### STRONG INTERPRETATION

The paired groups are the two physical oscilloscope input channels.

### UNKNOWN

- Which group is CH1 and which is CH2.
- Exact meaning of the three values per channel.
- Whether each value represents gain, offset, calibration, range, or a derived acquisition value.

---

## 15. Horizontal subsystem

### CONFIRMED call path

`HorizontalInit()` at `0x4003A62C` reaches hardware-apply function `0x4003F894`, which owns the SMEC `0x19..0x23` path.

IDs `0x19..0x20` form a contiguous eight-byte source array:

```text
app_state + 0x674 .. app_state + 0x67B
```

They map one-to-one to FPGA commands:

```text
SMEC 0x19 → FPGA 0xA6
SMEC 0x1A → FPGA 0xA7
SMEC 0x1B → FPGA 0xA8
SMEC 0x1C → FPGA 0xA9
SMEC 0x1D → FPGA 0xAA
SMEC 0x1E → FPGA 0xAB
SMEC 0x1F → FPGA 0xAC
SMEC 0x20 → FPGA 0xAD
```

The eight source bytes are initialized together in a loop. That initialization uses FPGA command pairs in the `0x96..0xA5` range before the runtime `0xA6..0xAD` setters are used.

### IDs `0x21..0x23`

```text
SMEC 0x21 ← app_state +0x6E4 → FPGA command 0x04
SMEC 0x22 ← app_state +0x6E5 → FPGA command 0x05
SMEC 0x23 ← app_state +0x6E6 → FPGA command 0x06
```

These are not independent UI values. They are derived together as part of one horizontal/timebase profile.

---

## 16. Horizontal timebase profile

### CONFIRMED

The producer writes:

```text
field_6E4 = 64 - r4
field_6E5 = 64 - r5
field_6E6 = selected profile value
field_6E7 = r6
field_6E8 = r7
```

The apply path is:

```text
app_state +0x6E4 → SMEC 0x21 → FPGA 0x04, 1 byte
app_state +0x6E5 → SMEC 0x22 → FPGA 0x05, 1 byte
app_state +0x6E6 → SMEC 0x23 → FPGA 0x06, 1 byte
app_state +0x6E7 →             FPGA 0x02, 1 byte
app_state +0x6E8 →             FPGA 0x0D, 4 bytes big-endian
apply/latch                     FPGA 0x07: write 1, then 0
```

### 64-bit timing dispatcher

The firmware computes a 64-bit timing value and dispatches through a 24-entry `(key, relative target)` table.

Confirmed timing keys:

```text
50,000
125,000
250,000
500,000
1,250,000
2,500,000
5,000,000
12,500,000
25,000,000
50,000,000
125,000,000
250,000,000
500,000,000
1,250,000,000
2,500,000,000
5,000,000,000
12,500,000,000
25,000,000,000
40,000,000,000
50,000,000,000
125,000,000,000
250,000,000,000
400,000,000,000
500,000,000,000
```

### Decoded profile region

For a large range the profile is stable:

```text
FPGA 0x04 = 54
FPGA 0x05 = 63
FPGA 0x06 = 32
FPGA 0x02 = 4
```

while command `0x0D` takes the sequence:

```text
49999
19999
9999
4999
1999
999
499
199
99
49
19
9
4
1
```

Adding one gives the classic 1-2-5 sequence:

```text
50000, 20000, 10000, 5000, 2000, 1000,
500, 200, 100, 50, 20, 10, 5, 2
```

**CONFIRMED:** FPGA command `0x0D` carries a timebase-related count/divider in `N - 1` form.

At slower profiles:

```text
FPGA 0x0D = 0
FPGA 0x02 = 1
```

and the values sent through SMEC `0x21..0x23` / FPGA `0x04..0x06` change instead.

### STRONG INTERPRETATION

The horizontal unit has at least two timing modes:

```text
counter-driven mode
    stable 0x04..0x06
    changing 0x0D counter

profile/phase-driven mode
    0x0D = 0
    changing 0x04..0x06
```

The exact internal FPGA implementation remains unknown.

### Sparse fallback profile switch

A separate fallback/internal timing switch explicitly handles:

```text
2, 4, 8, 16, 32, 48, 64, 80, 96, 112, 128
```

Its exact semantic input is still unknown.

---

## 17. Signal-generator subsystem

### CONFIRMED

`Gen_Init()` at `0x4004D164` owns SMEC IDs `0x14..0x18`.

Current source-field map:

```text
0x14  generator_state +0x08
0x15  generator_state +0x70
0x16  generator_state +0x68
0x17  generator_state +0x3C
0x18  generator_state +0x6C
```

The surrounding code performs floating-point division and integer conversion before updating some generator-state fields.

### STRONG INTERPRETATION

At least some values are calculated timing or synthesis parameters rather than direct UI settings.

### UNKNOWN

Exact mapping to generator frequency, waveform, amplitude, offset, duty cycle, or phase.

---

## 18. SysConfig/common subsystem

### CONFIRMED

`SysConfigExec()` at `0x4003E708` reapplies:

1. horizontal hardware configuration,
2. vertical hardware configuration,
3. a common configuration block at `0x40063328`.

The common block owns SMEC IDs `0x0E..0x13` and uses direct FPGA commands including:

```text
0x0C
0x15
0x16
0x17
0x61
```

It uses a mode/index byte near:

```text
app_state +0x105
```

and an indexed table beginning near:

```text
app_state +0x1C4
```

It also reads common fields near:

```text
app_state +0x1A0
app_state +0x120
```

### STRONG INTERPRETATION

This is likely a common trigger/acquisition configuration block.

### UNKNOWN

Exact meaning of IDs `0x0E..0x13` and direct FPGA commands `0x15..0x17` on DPOS350P.

---

## 19. Acquisition status/control findings

### CONFIRMED register behavior

```text
FPGA 0x0A  read one byte
FPGA 0x0B  read bit 0 and store it in app state
FPGA 0x14  read two bytes, combine big-endian to uint16
FPGA 0x21  observed in a 0→1 control pulse sequence
```

### STRONG INTERPRETATION

`0x0A`, `0x0B`, and `0x21` are close to the horizontal/acquisition state machine and are likely acquisition status/control functions.

Exact semantic names are not confirmed yet.

### Waveform reader status

**SUPERSEDED by sections 32-36:** waveform transport is now confirmed as the R11 CSI capture path. The text below is retained only as historical search context.

The waveform bulk reader had **not yet been located at this milestone**.

A previous exploratory candidate region `0x40049350..0x400498E4` has now been eliminated because it is conclusively the GT911 software-I2C driver.

The waveform path must therefore be found elsewhere, likely via:

```text
inlined PE parallel-bus access
another block-transfer helper
FPGA RAM/FIFO addressing commands
or another acquisition-specific transfer primitive
```

---

## 20. Relationship to FNIRSI 1013D

### CONFIRMED on DPOS350P

DPOS350P uses:

- an 8-bit MCU-to-FPGA parallel bus,
- command-based FPGA access,
- SMEC commands `0x64..0x6B`,
- parameter IDs,
- status polling,
- retries,
- 32-bit results,
- frequent runtime parameter updates.

### STRONG INTERPRETATION

FNIRSI reused or evolved the same broad architecture previously reverse-engineered in the 1013D family.

### Important caution

The numeric meanings documented for 1013D must not be copied blindly to DPOS350P. The architecture is closely related, but exact command payloads and parameter semantics may differ.

---

## 21. Unmarked device and captured I2C-like traffic

Another researcher observed on the four-pin header near the unmarked device:

- address zero in decoded runtime traffic,
- repeated approximately eight-byte transactions,
- runtime traffic that can partly be decoded as I2C,
- unusual power-on traffic that is less standard.

### STRONG INTERPRETATION

The observed traffic could be the FPGA-side implementation of the SMEC transactions identified in firmware.

### HYPOTHESIS

The FPGA may emulate open-drain behavior using tristate outputs, producing unusual startup waveforms while pin directions and bus ownership are changing.

### UNKNOWN

- Exact electrical protocol.
- Exact packet format on the four-pin header.
- Whether the device performs simple numerical translation, obfuscation, calibration, authentication, or another function.
- Whether the device is an EEPROM, MCU, ASIC, or another component.

---

## 22. Reconstructed high-level architecture

```text
Original or custom ARM software
          │
          ├── PE0..PE3 + PE8..PE15
          │       └── GOWIN FPGA parallel bus
          │            ├── acquisition commands
          │            ├── horizontal configuration
          │            ├── vertical configuration
          │            ├── generator configuration
          │            └── SMEC commands 0x64..0x6B
          │
          ├── PG6 / PG7
          │       └── GT911 software I2C @ 0x5D
          │
          ├── PE22 / PE24
          │       └── GT911 INT / RESET/address-selection
          │
          └── PE20 / PE21
                  └── MS5351M software I2C @ 0x60
                        ├── 95 MHz configuration
                        └── 10 MHz configuration

GOWIN FPGA
    │
    └── probable SMEC I2C-like link
            └── unmarked external device
```

---

## 23. Open questions
- Re-check the exact local GOWIN FPGA package marking before applying package-specific JTAG pin numbers.
- Locally verify externally reported FPGA command `0x02` charging-status bit and command `0x03` power-off behavior.
- Track known DPOS350P hardware revisions by FPGA package and SPI-NOR JEDEC/vendor.

1. Assign names and electrical direction to all PE0..PE3 FPGA bus-control signals.
2. Confirm PE20/PE21 as SDA/SCL.
3. Map 95 MHz and 10 MHz to physical MS5351M outputs and PCB destinations.
4. Determine exact SMEC packet format on the FPGA-side four-pin interface.
5. Identify the unmarked device.
6. Determine the meaning of SMEC.
7. Assign CH1 versus CH2 to the vertical even/odd ID groups.
8. Name individual vertical parameters `0x08..0x0D`.
9. Name common/SysConfig parameters `0x0E..0x13`.
10. Name generator parameters `0x14..0x18`.
11. Name horizontal parameters `0x19..0x20` more precisely.
12. Determine exact internal meanings of horizontal SMEC `0x21..0x23`.
13. Determine where/how the 32-bit SMEC response is consumed beyond transaction validation.
14. Compare reconstructed SMEC transactions against the logic-analyzer capture.
15. Confirm whether SMEC is merely translation/obfuscation or has a validation/security role.
16. Determine exact semantics of FPGA commands `0x0A`, `0x0B`, `0x14`, and `0x21`.
17. Locate waveform bulk readout and identify its buffer format.
18. Identify CH1/CH2 sample-read commands or bank-selection mechanism.

---

## 24. Current next steps

The CPU-GPIO bulk-reader search is obsolete. Sections 32-36 confirm CSI transport. The next milestone is a polling-based, timeout-protected, single-bank Custom App. Static analysis should resume only when a hardware result exposes a specific ambiguity.

## 25. Planned staged hardware tests

No hardware test is required merely to continue the current static analysis. Once the relevant code is reconstructed, use controlled staged tests.

### Stage A: complete `FPGA_Start()` clone

Instrument separately:

```text
FPGA parallel-bus init
FPGA ID before clock setup
MS5351M GPIO/I2C setup
95 MHz programming
10 MHz programming
FPGA ID after clock setup
```

Useful observations:

- FPGA ID before and after clock configuration.
- MS5351M traffic on PE20/PE21.
- Physical 95 MHz and 10 MHz output clocks if accessible.
- Relay click timing, if any.

### Stage B: SMEC transport

Add only low-level SMEC transport and status probes before attempting the full runtime parameter set.

### Stage C: horizontal profile

Apply one statically reconstructed timebase profile and verify status behavior.

### Stage D: acquisition

Once acquisition start/status/readout are identified:

```text
configure known timebase
configure one channel
arm acquisition
wait sample-ready
read waveform buffer
render samples on the existing framebuffer/display stack
```

The first independently rendered waveform will be the main end-to-end milestone proving that the original FPGA bitstream can be used as a black box by the custom ARM software.

---

## 26. Suggested later single-variable experiments

When static analysis reaches genuine ambiguity, change exactly one hardware/UI parameter and compare the predicted protocol effect.

Candidates:

- change CH1 V/div only,
- change CH2 V/div only,
- change one channel offset only,
- change timebase only,
- change trigger source only,
- change trigger edge only,
- change generator frequency only,
- change generator amplitude only.

---

## 27. Peripheral-information policy

Do not guess board-specific routing when a known-good custom implementation already exists.

Before interpreting code belonging to these peripherals, use or request the established custom-driver definitions:

```text
GT911
SDMMC / internal SD-NAND
LCD/display controller and panel routing
backlight
other board-specific GPIO wiring
```

Static EXE2 analysis should verify and extend those known facts instead of replacing them with assumptions inferred from pin numbers alone.

---

## 28. Snapshot integrity notes

Do not regress to these obsolete or disproven assumptions:

- The MCU-to-FPGA interface is **not SPI**. It is an 8-bit parallel bus.
- DPOS350P hardware is not assumed to be single-revision: an external unit reports a GW1N-LV9EQ176C6/I5 FPGA and Zbit ZB25VQ16A NOR, while the local SPI baseline is Winbond `EF 40 15`.
- The original FPGA diagnostic Custom App did **not** reproduce all of `FPGA_Start()`.
- The functions taking `95,000,000` and `10,000,000` are part of MS5351M clock setup.
- `SMEC_Init()` is not an isolated opaque call. Its MCU-side command protocol, cache, retry logic, and subsystem ownership are substantially mapped.
- The unmarked device's exact purpose remains unconfirmed despite strong architectural similarity to the FNIRSI 1013D SMEC design.
- The original code around `0x40049350..0x400498E4` is **GT911 software I2C**, not FPGA waveform readout.
- GT911 software I2C is on **PG6/PG7**, not PE6/PE7.
- The custom driver's `GT911_INT_SETUP_MS = 100` is a conservative safety margin; hardware testing established approximately 50 ms as the point where touch coordinate reporting begins to work reliably enough to proceed.

### 29. Acquisition-control and post-ready update (2026-10-23)

This section extends the previous snapshot without replacing its hardware, GT911, FPGA-start, SMEC-map, vertical, horizontal, generator, SysConfig, and staged-test material.

#### Corrected SMEC response flow

**CONFIRMED:** The success path does not compare the final 32-bit value against the parameter ID.

```text
write timing through 0x69
write parameter ID through 0x6A
write input value through 0x6B
prepare and wait
read short echoed ID
read validity/status bit
if echoed ID and status are valid:
    read final 32-bit big-endian result through 0x66
```

The cache-aware updater and flush caller both discard the final ARM return value. Therefore it is not used by ARM as a waveform command, pointer, selector, or token.

**STRONG INTERPRETATION:** The final value is either consumed internally by the FPGA or must be read to complete/drain the transaction.

#### Timebase command sequence

**CONFIRMED:** A timebase profile is applied as:

```text
SMEC 0x21 + FPGA 0x04  one byte
SMEC 0x22 + FPGA 0x05  one byte
SMEC 0x23 + FPGA 0x06  one byte
FPGA 0x02              mode byte
FPGA 0x0D              32-bit big-endian timing count
FPGA 0x07              write 1, then 0 to latch/apply
```

The 24 decoded profiles cover 10 ns/div through 100 ms/div. For the large counter-driven range, FPGA command `0x0D` follows an `N - 1` sequence such as `49999, 19999, 9999, ... 9, 4, 1`. Slower profiles switch to mode `0x02 = 1`, use `0x0D = 0`, and alter the three SMEC-assisted timing-profile bytes.

#### Acquisition command map

**CONFIRMED:**

```text
0x08  control pulse: write 0, then 1
0x0A  one-bit status, polled up to 0x9C4 (2500) attempts
0x0B  second one-bit runtime acquisition status
0x14  read 16-bit big-endian auxiliary acquisition value
0x18  validity/status bit for the 0x14 value
0x20  write a 16-bit acquisition parameter
0x21  apply pulse associated with command 0x20
```

Direct helper/caller links:

```text
read 0x0B helper @ 0x4003F8B8  <- caller 0x400417FC
pulse 0x08 helper @ 0x4003F8E0 <- callers 0x4003C54C and 0x400419A8
read 0x0A helper @ 0x4003F9D0  <- polling caller 0x4003FCDC
write 0x20/0x21 @ 0x4003F988   <- tail-call 0x4003FAE0
read 0x18/0x14 @ 0x40063798    <- tail-call 0x4003FAE4
```

**STRONG INTERPRETATION:**

```text
0x08  acquisition arm/reset/acknowledge pulse
0x0A  acquisition-complete or data-ready status
0x0B  higher-level trigger/run/acquisition state
0x20/0x21 and 0x18/0x14  related write/read pair, possibly capture or trigger position
```

#### Post-ready 16-bit processing candidate

**CONFIRMED:** Function `0x4003FD10` selects one of two parallel state/buffer branches and processes arrays of 16-bit values. Its operations include:

```text
LDRH sample/point reads
STRH in-place updates
minimum and maximum tracking
accumulation and averaging
integer division
linear interpolation/rescaling
```

One inner transformation is structurally equivalent to:

```c
while (count--) {
    uint16_t point = *points;
    point = baseline + scale_num * (point - baseline) / scale_den;
    *points++ = point;
}
```

**STRONG INTERPRETATION:** The two branches are likely CH1 and CH2 waveform/display-point processing. This is not yet sufficient to name the buffers as raw ADC samples. They may contain decimated samples, display points, min/max values, or another intermediate representation.

#### Waveform transport status

The actual FPGA-to-ARM bulk waveform transfer primitive is still unknown.

Eliminated false leads:

- The `0x40049350..0x400498E4` region is the original GT911 software-I2C layer, not waveform readout.
- GPIO helpers `0x40063274` and `0x40063288` are general GPIO helpers, not FPGA-exclusive strobes.
- The final SMEC 32-bit result is not an ARM-side dynamic waveform command.

Current best static path:

```text
identify all callers of 0x4003FD10
resolve both selected 16-bit buffer bases
trace their producers
locate buffer fill/swap or FPGA transfer primitive
confirm CH1/CH2 mapping and point format
```

#### Updated RE estimate

```text
FPGA startup and physical interface       85-90%
SMEC protocol on ARM side                 85-90%
Timebase configuration                    80-85%
Vertical configuration                    40-50%
Trigger/common configuration              35-45%
Signal generator                          30-40%
Acquisition control flow                  50-60%
Waveform processing/buffer structures     25-35%
Waveform FPGA-to-ARM transport             10-20%
Practical custom oscilloscope application about 52%
Full FPGA internals / replacement         25-30%
```


### 30. Waveform buffer and segmented acquisition pipeline update (2026-10-23)

This section extends section 29 and records the confirmed buffer-processing and segmented-acquisition findings. It does not replace the earlier hardware, startup, SMEC, timebase, vertical, generator, SysConfig, or GT911 sections.

#### Channel waveform buffers

**CONFIRMED:** The acquisition state contains two channel-indexed pointers and a shared processing count/limit:

```text
state + 0x7B4  pointer to channel-0 uint16_t waveform points
state + 0x7B8  pointer to channel-1 uint16_t waveform points
state + 0x7BC  shared point count or processing limit
```

The direct access pattern is:

```c
uint16_t *points = state->channel_waveform[channel];
```

Both channels are processed consecutively:

```text
0x400416B4  call 0x4004006C(0)
0x400416BC  call 0x4004006C(1)
```

**STRONG INTERPRETATION:** Channel 0 and channel 1 correspond to physical CH1 and CH2. The exact physical assignment remains to be confirmed.

#### Per-channel processing pipeline

The only direct caller of `0x4003FD10` is `0x400400D4`, inside the per-channel wrapper beginning at `0x4004006C`.

Confirmed sequence:

```text
0x4004006C(channel)
    ├─ prepare/update channel processing window
    ├─ 0x4003FD10(channel)
    │    amplitude statistics and in-place rescaling
    ├─ 0x4005C7BC(channel, buffer, ...)
    │    work-buffer copy and resampling
    └─ tail-call 0x4003FE14(channel)
         final in-place channel normalization
```

The subsequent runtime pipeline invokes further symmetric channel functions:

```text
0x40040F18(0), 0x40040F18(1)
0x40040FC4(0), 0x40040FC4(1)
0x400403AC(0), 0x400403AC(1)
0x40041094(0, ...), 0x40041094(1, ...)
```

#### `0x4003FD10`: amplitude/statistical post-processing

**CONFIRMED:** The function selects one of two channel-specific state groups and processes `uint16_t` point arrays using:

```text
minimum and maximum tracking
accumulation and averaging
integer division
crossing/range checks
in-place positive-side rescaling
in-place negative-side rescaling
```

A representative transformation is:

```c
point = baseline + scale_num * (point - baseline) / scale_den;
```

Separate denominators are used for values above and below the reference level.

#### `0x4005C7BC`: common workspace and resampling

**CONFIRMED:** The function copies part of a channel buffer into a common 16-bit workspace at:

```text
0x416431E8
```

It then selects one of three general resampling algorithms and copies the result back into the channel buffer.

```text
mode 0      -> 0x4005C678
mode 1      -> 0x4005D048
other       -> 0x4005D2E4
```

Current interpretation:

```text
0x4005C678  simple integer index mapping / nearest-neighbour-like resampling
0x4005D048  floating-point interpolation/resampling
0x4005D2E4  fixed-point weighted or filtered resampling
```

These algorithms are general signal/array-processing helpers and are also called outside the acquisition path. They must not be named as specific oscilloscope acquisition modes without additional evidence.

For one channel/mode path, state values 12 through 16 produce edge corrections:

```text
12 -> 15
13 -> 7
14 -> 4
15 -> 2
16 -> 1
```

**STRONG INTERPRETATION:** These values compensate a filter/resampler edge or warm-up region.

#### `0x4003FE14`: final channel normalization

**CONFIRMED:** The function directly loads `channel_waveform[channel]`, selects one of two symmetric channel configuration groups, scans `uint16_t` points, tracks min/max and average values, and rescales the waveform in place around a channel-specific baseline.

The function therefore operates on an amplitude trace, not merely unrelated configuration data.

#### `0x40040114`: smoothing and crossing/alignment analysis

**CONFIRMED:** This function selects the currently active channel buffer, creates a moving-average waveform in a temporary buffer, and searches adjacent sample pairs for a crossing of a reference amplitude.

The crossing test is structurally equivalent to:

```c
(a <= level && b > level) || (a >= level && b < level)
```

It searches in both directions from a calculated nominal position and stores a signed displacement.

**STRONG INTERPRETATION:** The result is a trigger-alignment or horizontal waveform-alignment correction relative to the expected trigger position.

This function analyzes an already populated waveform buffer. It is not the FPGA bulk reader.

#### Segmented acquisition runtime

**CONFIRMED:** The runtime loop around `0x40041500..0x400416A8` processes acquisition data in multiple blocks or banks.

Per-block sequence:

```text
select block/bank through 0x4006338C
configure/start through 0x4003FA10
wait for command 0x0A ready through 0x4003FCC0
check acquisition-ready state
read command 0x18 / 0x14 through 0x4003FAE4
update the capture position/index through 0x4003FB90
record segment metadata
convert raw/intermediate data into channel waveform points
```

The loop maintains three parallel arrays indexed by block:

```text
segment start/offset
segment center
segment length
```

**STRONG INTERPRETATION:** The final waveform is assembled from several FPGA acquisition blocks or banks rather than one monolithic transfer.

#### Raw 8-bit to calibrated 16-bit conversion

**CONFIRMED:** Functions `0x400408FC` and `0x40040B20` repeatedly invoke the common core `0x4004073C`.

`0x4004073C` performs the characteristic conversion:

```c
uint8_t raw = *src++;
int value = calibration_offset + calibration_scale * raw;
value = baseline + gain * (value - baseline) / 32;
value = clamp(value, 0, upper_limit);
*dst = (uint16_t)value;
dst += output_stride;
```

Therefore:

```text
raw/intermediate acquisition input  8-bit
channel waveform representation     16-bit
```

The core combines raw bytes with calibration, baseline, gain, clamp, and output-stride parameters.

`0x400408FC` and `0x40040B20` each call the conversion core twelve times using parameter groups around:

```text
state + 0x6D4..0x6DB
state + 0x6DC..0x6E3
```

and lane/order values descending through `7..0`.

**STRONG INTERPRETATION:** These wrappers unpack/interleave acquisition lanes or sub-blocks into the two 16-bit channel waveforms. The precise meaning of all twelve calls is still unknown.

#### Current reconstructed waveform path

```text
FPGA acquisition block/bank selection
        ↓
command 0x08 arm/reset pulse
        ↓
command 0x0A ready polling
        ↓
command 0x18 validity + command 0x14 auxiliary position/index
        ↓
raw/intermediate 8-bit acquisition bytes
        ↓
0x4004073C calibration/gain/baseline/clamp conversion
        ↓
channel_waveform[0] / channel_waveform[1] as uint16_t
        ↓
0x40040114 moving average and crossing/alignment search
        ↓
0x4003FD10 amplitude/statistical normalization
        ↓
0x4005C7BC workspace copy and resampling
        ↓
0x4003FE14 final per-channel normalization
        ↓
additional symmetric measurement/display processing
```

#### Still unknown

```text
exact address and ownership of the raw 8-bit source buffer
exact low-level FPGA RAM/FIFO transfer primitive
exact command or selector used to fetch sample bytes
exact meaning of all twelve lane/sub-block conversion calls
exact distinction between 0x400408FC and 0x40040B20 modes
confirmed physical CH1/CH2 assignment for channel indices 0/1
```

#### Updated RE estimate

```text
FPGA startup and physical interface       85-90%
SMEC protocol on ARM side                 85-90%
Timebase configuration                    80-85%
Vertical configuration                    40-50%
Trigger/common configuration              40-50%
Signal generator                          30-40%
Acquisition control flow                  60-70%
Waveform processing/buffer structures     60-70%
Waveform FPGA-to-ARM transport             20-30%
Practical custom oscilloscope application about 57%
Full FPGA internals / replacement         25-30%
```

#### Next static-analysis targets

```text
0x4006338C  acquisition block/bank selector
0x4003FB90  consumer/translator of the 16-bit value read through command 0x14
0x4004073C  exact source-pointer provenance
0x400408FC  first conversion wrapper
0x40040B20  second conversion wrapper
```

The immediate goal is to identify the raw 8-bit source-buffer producer and the exact low-level FPGA RAM/FIFO readout sequence.

### 31. Raw acquisition-bank memory snapshot (2026-10-23)

This milestone extends section 30 with the first confirmed raw acquisition-memory layout and a tighter reconstruction of the FPGA bank/position protocol.

#### FPGA command `0x15`: acquisition bank/lane selector

**CONFIRMED:** Function `0x4006338C` issues FPGA command `0x15` followed by one selector byte.

In one acquisition mode the selector is exactly:

```c
selector = index & 7;
```

In the other mode, the selected channel changes the encoding and produces two interleaved selector groups.

**STRONG INTERPRETATION:** Command `0x15` selects an acquisition RAM bank, interleave lane, sample phase, or a combined bank/channel phase.

#### Commands `0x20/0x21` and `0x18/0x14`: cyclic position protocol

**CONFIRMED:** The value written through command `0x20` is reduced modulo the acquisition period/ring length before command `0x21` applies it with a `0 -> 1` pulse.

The 16-bit value read through command `0x14`, after command `0x18` reports validity, is used in 64-bit position calculations and modulo reduction. Normalized position values are stored at state offsets `+0x288` and `+0x28C`.

**STRONG INTERPRETATION:**

```text
0x20 / 0x21  requested capture/read position and apply pulse
0x18 / 0x14  validity plus actual/current capture position
```

The wrap-around handling strongly indicates a circular acquisition RAM or periodic capture-address space.

#### Confirmed raw acquisition-memory pool

**CONFIRMED:** The raw 8-bit source consumed by `0x4004073C` is addressed from the static base:

```text
RAW_ACQ_BASE = 0x42DF76D4
```

The bank stride is:

```text
RAW_BANK_SIZE = 0x2134 = 8500 bytes
```

The conversion core accepts bank indices `0..7`, giving eight raw slots:

```text
bank 0  0x42DF76D4 + 0 * 0x2134
bank 1  0x42DF76D4 + 1 * 0x2134
bank 2  0x42DF76D4 + 2 * 0x2134
bank 3  0x42DF76D4 + 3 * 0x2134
bank 4  0x42DF76D4 + 4 * 0x2134
bank 5  0x42DF76D4 + 5 * 0x2134
bank 6  0x42DF76D4 + 6 * 0x2134
bank 7  0x42DF76D4 + 7 * 0x2134
```

Total raw-pool span:

```text
8 * 8500 = 68000 bytes
```

A practical model is:

```c
#define RAW_BANK_COUNT 8
#define RAW_BANK_SIZE  0x2134

uint8_t raw_acq[RAW_BANK_COUNT][RAW_BANK_SIZE]
    /* located at 0x42DF76D4 */;
```

#### Raw source-pointer reconstruction

**CONFIRMED:** `0x4004073C` builds its source pointer as:

```c
uint8_t *src =
    (uint8_t *)0x42DF76D4
    + bank_index * 0x2134
    + state->raw_ring_origin_6f2
    + subblock_input_offset;
```

It then reads sequential bytes using `LDRB` and advances the source pointer by one byte per raw sample.

The same state field at `+0x6F2` participates in the capture-position modulo calculation.

**STRONG INTERPRETATION:** `state + 0x6F2` is a trigger-relative ring-buffer read origin or raw capture offset.

#### Bank-specific calibration

**CONFIRMED:** The bank-index switch in `0x4004073C` selects two bank-specific 32-bit calibration values from arrays based near:

```text
0x42E53A0C + 0x10..0x2C
0x42E53A0C + 0x30..0x4C
```

Each of the eight banks therefore has its own calibration pair.

A practical model is:

```c
struct raw_bank_calibration {
    uint32_t coeff_a[8];
    uint32_t coeff_b[8];
};
```

**STRONG INTERPRETATION:** The eight banks are likely interleaved ADC phases or parallel acquisition lanes requiring individual offset/scale correction.

#### Relationship between command `0x15` and ARM raw slots

The following relationship is now strongly supported:

```text
FPGA command 0x15 selector 0..7
        ↓
FPGA acquisition bank/lane 0..7
        ↓
ARM raw slot 0..7 at 0x42DF76D4
        ↓
0x4004073C bank-specific calibration and conversion
        ↓
16-bit channel waveform points
```

The wrappers process banks predominantly in descending order `7..0`, with some additional calls using different offsets or destinations.

#### Updated acquisition command map

```text
0x08  acquisition arm/reset/acknowledge pulse
0x0A  acquisition-ready/completion status
0x0B  higher-level runtime trigger/acquisition status
0x14  16-bit current capture/ring position
0x15  acquisition bank/lane/channel-phase selector
0x18  valid flag for command 0x14
0x20  requested capture/read position
0x21  apply pulse for command 0x20
```

Evidence levels:

```text
0x15 command and one-byte selector                  CONFIRMED
0x14 used as a cyclic position                      CONFIRMED
0x20 value reduced modulo acquisition period        CONFIRMED
0x20/0x21 and 0x18/0x14 as a write/read pair       STRONG INTERPRETATION
FPGA sample RAM organized as a circular buffer      STRONG INTERPRETATION
```

#### Updated reconstructed data path

```text
ARM computes requested cyclic capture/read position
        ↓
FPGA 0x20 + 16-bit position
FPGA 0x21 apply pulse
        ↓
FPGA 0x15 bank/lane selector
        ↓
FPGA capture and command 0x0A ready polling
        ↓
FPGA 0x18 validity + FPGA 0x14 actual position
        ↓
ring-origin and wrap-around correction
        ↓
raw slot at 0x42DF76D4 + bank * 0x2134
        ↓
0x4004073C bank calibration and 8-bit to 16-bit conversion
        ↓
channel_waveform[0] / channel_waveform[1]
        ↓
trigger alignment, resampling, normalization, measurement/display pipeline
```

#### Remaining transport gap at this historical milestone

**SUPERSEDED by sections 32-36:** R11 CSI is the confirmed producer. At this earlier milestone, the producer was still unknown:

```text
who writes FPGA sample bytes into 0x42DF76D4..0x42E08073
```

Leading possibilities remain:

```text
CPU bulk-read helper
DMA/peripheral transfer
memory-mapped FPGA transfer window
system helper invoked during acquisition setup
```

The next static-analysis target is every direct or derived write into the raw-pool range, including copy/DMA helpers and pointers computed from `0x42DF76D4`.

#### Updated RE estimate

```text
FPGA startup and physical interface       85-90%
SMEC protocol on ARM side                 85-90%
Timebase configuration                    80-85%
Vertical configuration                    40-50%
Trigger/common configuration              45-55%
Signal generator                          30-40%
Acquisition control flow                  75-80%
Waveform processing/buffer structures     70-75%
Waveform FPGA-to-ARM transport             45-50%
Practical custom oscilloscope application about 63%
Full FPGA internals / replacement         25-30%
```

### 32. CSI waveform-transport cross-check (2026-10-23)

This section records an independent external analysis supplied from commit `e55834596003e5b3feaae0fbfe1c1132a50666b4` in `fk0815/FNIRSI-DPOS350P-Hack` and reconciles it with the locally reconstructed EXE2 acquisition path. Earlier sections are intentionally retained as historical milestones; where they describe the waveform transport as unknown, this section supersedes that status.

#### Evidence boundary

The CSI findings below come from an independent analysis of the stock/update firmware and are highly consistent with the local EXE2 reverse engineering. They are marked as **EXTERNAL CROSS-CHECK / STRONG INTERPRETATION** until the relevant CSI routine and MMIO writes are independently re-disassembled from the local `exe2-from-backup.bin`.

#### PE port dual role: GPIO register bus and CSI waveform bus

**EXTERNAL CROSS-CHECK:** The same PE pins used for the FPGA GPIO register interface can be muxed to R11 CSI function 2:

```text
PE0       CSI_PCLK
PE2       CSI_HSYNC
PE3       CSI_VSYNC
PE8..PE15 CSI_D4..CSI_D11

PE1       CSI_MCLK       not used in the reported path
PE4..PE7  CSI_D0..D3     not used in the reported path
```

The resulting working architecture is:

```text
GPIO mode on PE pins
    ARM <-> FPGA command/register/status access

CSI function-2 mode on the same PE pins
    FPGA -> ARM bulk waveform stream
```

This explains why no large CPU loop calling `FPGA_read_byte()` was found for waveform transfer even though the later ARM processing pipeline consumes large raw sample buffers.

#### `FPGA_Start()` CSI initialization

The independent analysis identifies `0x400660C8`, called from `FPGA_Start()` with `0x1F40`, as an R11 CSI initialization routine rather than a delay helper.

Reported properties:

```text
function             0x400660C8
argument              0x1F40 = 8000
CSI MMIO base         0x01CB4000
reported role         initialize CSI capture/channel descriptor
```

The previous description of `0x400660C8` as a delay/system helper is therefore obsolete.

**STRONG INTERPRETATION:** `FPGA_Start()` initializes two complementary FPGA paths:

```text
control plane: PE GPIO parallel command bus
sample plane:  PE CSI waveform stream
```

#### Relationship to the locally confirmed raw acquisition pool

Local EXE2 analysis independently established:

```text
RAW_ACQ_BASE     0x42DF76D4
RAW_BANK_COUNT   8
RAW_BANK_STRIDE  0x2134 = 8500 bytes
raw sample type  uint8_t
```

The CSI-init argument is:

```text
0x1F40 = 8000 bytes
```

Difference between raw-slot stride and the reported CSI descriptor quantity:

```text
8500 - 8000 = 500 bytes = 0x1F4
```

**HYPOTHESIS:** The 500-byte difference may represent padding, guard space, metadata, alignment, or a distinction between the active CSI payload and allocated raw-slot stride. No semantic meaning is assigned yet.

The immediate static-analysis goal is to find the CSI destination/buffer-address registers and determine whether they reference `0x42DF76D4`, one of the eight `+ n * 0x2134` slots, or a staging buffer copied into that pool.

#### Updated end-to-end acquisition architecture

The combined evidence now supports the following working model:

```text
                         CONTROL PLANE
ARM/R11  <------------------------------------>  GOWIN FPGA
           PE pins in GPIO mode
           FPGA commands, status and positions
           0x08, 0x0A, 0x0B, 0x14, 0x15,
           0x18, 0x20, 0x21, ...

                          SAMPLE PLANE
GOWIN FPGA
    |
    | PE0 PCLK
    | PE2 HSYNC
    | PE3 VSYNC
    | PE8..PE15 DATA[7:0]
    v
R11 CSI @ 0x01CB4000
    |
    | hardware capture / memory write
    v
raw acquisition RAM
    likely associated with 0x42DF76D4 raw pool
    |
    v
0x4004073C
bank-specific calibration and uint8 -> uint16 conversion
    |
    v
channel_waveform[0] / channel_waveform[1]
    |
    v
trigger alignment, resampling, normalization,
measurements and display processing
```

The exact CSI destination programming remains to be locally confirmed, so the arrow from CSI to `0x42DF76D4` is currently a **STRONG INTERPRETATION**, not yet a local CONFIRMED fact.

#### External address-relocation cross-check

The independent analysis reports a constant `+0x80020` relocation between the local EXE2 FPGA-related block and the 2025-06-19 update image, including examples:

```text
EXE2        update image
0x400660C8  0x400E60E8
0x40049094  0x400C90B4
0x400490B8  0x400C90D8
0x4005F1BC  0x400DF1DC
0x4005F1F0  0x400DF210
0x4005F38C  0x400DF3AC
0x4004A15C  0x400CA17C
```

This relocation is useful as a cross-reference aid. It should not be assumed globally without byte-level/function-level comparison outside the verified block.

#### Clock-generator cross-check

The same independent analysis reports:

```text
PE20 = SCL
PE21 = SDA
7-bit I2C address = 0x60
8-bit write address = 0xC0
Si5351-style register protocol
register 177 = 0x80 PLL reset
multisynth P1/P2/P3 parameter blocks
```

This agrees with the locally reconstructed MS5351M path at `0x4005F1BC`, `0x4005F1F0`, and `0x4005F38C`. The main application programs 95 MHz and 10 MHz; the external analysis additionally reports that the updater contains the same style of driver and programs 125 MHz during its boot path.

The exact physical destination of each generated clock remains unknown.

#### FPGA-bitstream update caution

The independent analysis did not identify a firmware path that programs the GW1N-9 configuration in the examined bootloader/update images.

This document deliberately does **not** conclude that JTAG is the only possible FPGA update mechanism. The defensible current statement is:

> No FPGA-configuration programming path has yet been identified in the examined stock bootloader/update images. JTAG is a confirmed physical access route, while other factory/service update mechanisms remain possible unless excluded by further evidence.

#### Superseded waveform-transport assumptions

The following earlier search assumptions should no longer guide the primary analysis:

```text
waveform bulk transfer is probably a repeated FPGA_read_byte() GPIO loop
waveform source should necessarily be found by searching PE_DATA reads
0x400660C8 is a generic delay helper
```

The highest-priority working model is now CSI-based bulk capture with GPIO retained for the FPGA control plane.

#### Updated next static-analysis steps

1. Disassemble `0x400660C8` locally and identify every access to CSI MMIO around `0x01CB4000`.
2. Recover CSI pinmux, format, width, sync-polarity and capture-enable configuration.
3. Identify CSI destination/buffer-address registers and compare them against `0x42DF76D4` and the eight `0x2134`-stride raw slots.
4. Identify capture start/stop, frame-done/status and any CSI interrupt path used by the acquisition runtime.
5. Determine whether each FPGA `0x15` bank/lane selection maps to a CSI frame, CSI buffer slot, or lane/interleave phase within a frame.
6. Explain the `8000` active CSI quantity versus `8500` raw-slot stride without assigning meaning to the extra 500 bytes until proven.

#### Updated RE estimate after CSI cross-check

```text
FPGA startup and physical interface       90-95%
SMEC protocol on ARM side                 85-90%
Timebase configuration                    80-85%
Vertical configuration                    40-50%
Trigger/common configuration              45-55%
Signal generator                          30-40%
Acquisition control flow                  75-80%
Waveform processing/buffer structures     70-75%
Waveform FPGA-to-ARM transport             60-70%  (historical section-32 estimate; superseded below)
Practical custom oscilloscope application about 68%
Full FPGA internals / replacement         25-30%
```


#### 33. Locally confirmed CSI waveform transport (2026-10-23)

This milestone independently verifies the CSI hypothesis directly against the local `exe2-from-backup.bin`. Section 32 remains useful as the external cross-check history, but the core CSI transport facts below are now **CONFIRMED** locally.

##### `0x400660C8` is the CSI acquisition-init wrapper

**CONFIRMED:** `FPGA_Start()` calls `0x400660C8(0x1F40)`, and the argument `0x1F40 = 8000` is preserved and passed into the CSI setup path.

Observed call sequence:

```text
0x400660C8(8000)
    -> 0x40065F48(0)
    -> 0x400658BC(0, ...)
    -> 0x40065C68(0)
    -> 0x40065C64(0)
    -> 0x40065FA8(8000)
    -> tail-call 0x40066054
```

The previous historical description of `0x400660C8` as a delay/system helper is disproven.

##### Local CSI pinmux confirmation

**CONFIRMED:** Helper `0x40065EF8` configures the PE port for alternate function 2.

It programs individual pins:

```text
PE0  function 2
PE2  function 2
PE3  function 2
```

and writes:

```text
PE_CFG1 = 0x22222222
```

which places `PE8..PE15` on function 2.

Combined with the R11 CSI pin assignment independently identified in section 32, the waveform sample-plane pins are:

```text
PE0       CSI_PCLK
PE2       CSI_HSYNC
PE3       CSI_VSYNC
PE8..PE15 CSI_D4..CSI_D11  -> effective 8-bit sample/data bus
```

Thus the PE pins are intentionally dual-purpose:

```text
GPIO mode       FPGA command/register/status interface
CSI function 2  FPGA -> R11 waveform bulk stream
```

##### Direct CSI destination link to the raw acquisition pool

**CONFIRMED:** Function `0x40066054`, reached from the CSI initialization sequence, loads the literal raw-pool address:

```asm
ldr  r0, =0x42DF76D4
mov  r1, #0
strd r0, r1, [sp]
...
bl   0x40065A60
```

Before this call it invokes `0x40065EF8`, which switches the relevant PE pins to CSI function 2.

Therefore `0x42DF76D4`, previously recovered independently from the raw-sample consumer `0x4004073C`, is directly supplied to the CSI driver during capture-buffer setup.

The principal waveform transport is therefore locally established as:

```text
GOWIN FPGA
    |
    | PE0 PCLK
    | PE2 HSYNC
    | PE3 VSYNC
    | PE8..PE15 DATA[7:0]
    v
R11 CSI capture path
    |
    | hardware memory write
    v
RAW_ACQ_BASE = 0x42DF76D4
    |
    | raw uint8 samples / logical banks
    v
0x4004073C
    |
    | bank-specific calibration, gain, baseline and clamp
    v
uint16 channel waveform buffers
```

This closes the previous major gap between the FPGA sample source and the ARM raw acquisition memory. A CPU `FPGA_read_byte()` bulk-copy loop is not part of the primary waveform path.

##### CSI/CCU and driver state observations

**CONFIRMED:** The CSI helper block uses:

```text
0x01C20890  PE_CFG0
0x01C20100  CCU-area register used during CSI initialization
0x42E68C50  CSI driver/channel state base
0x42E5549C  CSI-related global/descriptor object
0x42E18074  CSI-related global object
0x42DF76D4  raw capture-buffer base
```

Function `0x40065F48` manipulates the CCU-area register at `0x01C20100` while setting up the CSI path.

**STRONG INTERPRETATION:** This performs CSI clock/reset gating. Exact bit names remain to be mapped to the R11/V3s CCU register definitions.

##### Capture/IRQ setup candidate

After configuring the CSI raw buffer, `0x40066054` passes a code address together with value `0x74` through helpers `0x40066BD4` and `0x40066B88`, then continues with a CSI helper using channel 0 and mode/type 2.

**STRONG INTERPRETATION:** This sequence registers/enables the CSI capture interrupt or frame-completion callback. The exact meaning of interrupt/vector value `0x74` remains to be locally named from the R11 interrupt map.

##### Reinterpreting the eight raw banks

The already-confirmed ARM consumer layout remains:

```text
RAW_ACQ_BASE     0x42DF76D4
RAW_BANK_COUNT   8
RAW_BANK_STRIDE  0x2134 = 8500 bytes
```

The CSI init/configuration uses the quantity:

```text
0x1F40 = 8000
```

The raw consumer addresses logical slots using `base + bank * 0x2134`, but the local CSI setup observed so far supplies the common base `0x42DF76D4` rather than eight literal destination addresses.

Current alternatives:

```text
A. CSI writes one larger/common capture region; ARM interprets it as eight logical 0x2134-stride lanes/banks.
B. CSI driver/descriptor logic derives or rotates additional destination addresses internally.
```

At this stage model A has less unproven machinery, but neither model is promoted to CONFIRMED until `0x40065A60` and its descriptor/MMIO writes are fully reconstructed.

The difference remains deliberately unexplained:

```text
8500-byte ARM logical bank stride
8000-byte CSI setup quantity
---------------------------------
 500 bytes = 0x1F4
```

Possible padding, guard, metadata, alignment, or active-payload explanations remain hypotheses only.

##### Updated architecture status

**CONFIRMED control plane:**

```text
ARM/R11 <-> GOWIN FPGA
PE GPIO parallel interface
registers, commands, status, bank/position control
```

**CONFIRMED sample plane:**

```text
GOWIN FPGA -> R11 CSI -> 0x42DF76D4 raw acquisition memory
```

**CONFIRMED downstream processing:**

```text
0x42DF76D4 raw uint8 data
    -> 0x4004073C calibration/conversion
    -> uint16 channel waveform buffers
    -> trigger alignment / resampling / normalization
    -> measurement and display processing
```

##### Superseded historical statements

Earlier sections are retained as milestone history, but the following statements are now obsolete:

```text
"waveform bulk reader has not yet been located"
"exact producer of 0x42DF76D4 is unknown"
"CSI -> 0x42DF76D4 is only a strong interpretation"
"0x400660C8 may be a generic delay/system helper"
```

The missing work is no longer locating the transport mechanism. It is reconstructing the detailed CSI configuration and runtime lifecycle.

##### Next static-analysis target

Priority target:

```text
0x40065A60
```

Goals:

```text
recover CSI destination descriptor layout
recover frame geometry and active byte count
recover hardware stride/buffer organization
recover CSI format and sync-polarity configuration
identify capture start/stop semantics
identify frame-done/status/IRQ handling
explain whether the eight 0x2134 slots are hardware or software logical banks
explain the 8000 versus 8500-byte relationship
```

##### Updated RE estimate after local CSI confirmation

```text
FPGA startup and physical interface       about 95%
SMEC protocol on ARM side                 85-90%
Timebase configuration                    80-85%
Vertical configuration                    40-50%
Trigger/common configuration              45-55%
Signal generator                          30-40%
Acquisition control flow                  75-80%
Raw acquisition memory layout             80-85%
Waveform processing/buffer structures     70-75%
FPGA-to-ARM waveform transport             80-85%
Practical custom oscilloscope application about 72%
Full FPGA internals / replacement         25-30%
```

## 34. CSI runtime buffer, capture-length and IRQ lifecycle (2026-10-23)

This milestone extends section 33 with the runtime CSI configuration and the locally reconstructed capture-completion interrupt path. Earlier milestone sections are retained for history; this section supersedes their remaining uncertainty about the CSI buffer setter, frame-completion flag and callback.

### Runtime CSI capture quantity and FPGA command `0x22`

**CONFIRMED:** Runtime function `0x400405E0` computes the value passed to `0x40065FA8` from two 16-bit acquisition fields:

```c
uint16_t csi_capture_quantity = field_a + 2 * field_b;
csi_configure_capture(csi_capture_quantity);  // 0x40065FA8
```

Immediately afterwards the same two fields are used to construct a 16-bit big-endian value sent with FPGA command `0x22`:

```c
uint16_t fpga_stream_quantity = 0x14 + 2 * (field_a + field_b);
fpga_write_command(0x22);
fpga_write_u16_be(fpga_stream_quantity);
```

Therefore the R11 CSI receiver geometry and FPGA stream producer geometry are configured together from the same runtime acquisition parameters.

**STRONG INTERPRETATION:** FPGA command `0x22` is a CSI stream/frame/line-length parameter. The exact hardware name remains open.

The startup value `8000` passed to `0x40065FA8` is therefore not treated as a fixed allocation size. The same descriptor quantity is dynamically reconfigured at runtime.

### CSI destination-address register

**CONFIRMED:** `0x40065A60` programs the CSI destination address as a 32-bit-word address:

```c
encoded_address = physical_address >> 2;
```

The destination-register address is structurally:

```text
CSI device base
+ channel * 0x100
+ buffer_index * 4
+ 0x50
```

Thus the hardware exposes destination registers at:

```text
buffer 0  +0x50
buffer 1  +0x54
buffer 2  +0x58
buffer 3  +0x5C
```

The analyzed stock EXE2 uses hardware buffer index 0 in the located call paths.

For the startup raw pool:

```text
physical address = 0x42DF76D4
register value   = 0x42DF76D4 >> 2 = 0x10B7DDB5
```

### Runtime destination setter `0x40066030`

**CONFIRMED:** `0x40066030` receives the runtime destination address in `r0`, promotes it to a 64-bit value with zero high word, and invokes `0x40065A60` for CSI instance 0 and hardware buffer index 0.

Practical model:

```c
void csi_set_runtime_buffer(uint32_t address)
{
    csi_set_buffer_address(
        0,      // CSI instance
        0,      // channel in the observed path
        0,      // hardware buffer index
        address);
}
```

This corrects the earlier transient interpretation that `0x40066030` necessarily clears the main destination address. In the acquisition runtime its caller moves a runtime value into `r0` before invoking this helper.

**STRONG INTERPRETATION:** The single hardware buffer-0 register can be retargeted between capture operations. The next priority is to prove whether the runtime address walks through `0x42DF76D4 + n * 0x2134`.

### CSI capture enable

**CONFIRMED:** `0x400660AC` loads the CSI descriptor field at `0x42E5549C + 0x10`, whose initialized value is 1, and invokes the `0x40065EA0 -> 0x40065A98` control path with enable value 1.

`0x40065A98` maps mode values 1..4 onto byte-wide fields at bit positions:

```text
mode 1  shift 0
mode 2  shift 8
mode 3  shift 16
mode 4  shift 24
```

and updates the CSI register at offset `+0x08`.

For the observed acquisition configuration:

```text
mode = 1
enable = 1
```

so the low field of CSI register `+0x08` is enabled.

**STRONG INTERPRETATION:** This is the capture/channel enable control for the active CSI output path.

### Exact CSI callback address and completion flag

**CONFIRMED:** The callback registered during CSI initialization is exactly:

```text
0x40065EB0
```

The address is formed at `0x40066084` by ARM PC-relative arithmetic and registered using interrupt/source value `0x74`.

The callback reads the CSI channel status register at offset:

```text
+0x74
```

Helper `0x40065BD4` expands the low eight status bits into eight individual byte flags.

The callback tests status bit 1:

```c
if (status[1] == 1) {
    state->csi_frame_done = 1;   // state + 0x70E
    csi_ack_status(1u << 1);    // write 0x02
}
```

Therefore:

```text
state + 0x70E                  CSI capture-completion flag
CSI channel register +0x74     interrupt/status flags
bit 1                          completion event used by acquisition runtime
acknowledge value              0x02
callback                        0x40065EB0
registered interrupt/source     0x74
```

**STRONG INTERPRETATION:** Status bit 1 is the CSI frame-done/capture-complete event. The exact official R11 register-field and interrupt-source names remain to be matched against the SoC documentation.

### Confirmed runtime synchronization

The acquisition runtime waits directly on `state + 0x70E` after enabling CSI and starting the FPGA-side acquisition:

```text
prepare capture/ring position
        ↓
prepare shared PE/FPGA state
        ↓
set runtime CSI destination address
        ↓
perform pre-capture memory/system helper
        ↓
0x400660AC: enable CSI capture
        ↓
start/arm FPGA acquisition
        ↓
FPGA emits CSI stream
        ↓
CSI writes to programmed runtime address
        ↓
CSI status +0x74 bit 1
        ↓
interrupt/source 0x74
        ↓
callback 0x40065EB0
        ↓
state +0x70E = 1
        ↓
write 0x02 to acknowledge completion
        ↓
runtime wait loop exits
        ↓
post-capture FPGA/memory handling
```

This establishes the complete ARM-side synchronization path from capture enable through interrupt acknowledgement and return to the acquisition runtime.

### CSI channel-register map recovered so far

```text
+0x08             capture/channel mode-enable fields
+0x50..+0x5C      destination buffer addresses, stored as address >> 2
+0x70             interrupt/status-control related register
+0x74             interrupt/status flags; bit 1 used for capture completion
+0x80             geometry/configuration field
+0x84             geometry/configuration field
+0x88             geometry/stride-related field
channel stride    0x100 bytes
```

The helpers around `0x40065AE8` and `0x40065B70` manipulate the geometry registers with 13-bit and packed 16-bit fields. Their precise official names remain open.

### Refined interpretation of 8000 versus 8500

The known values are now:

```text
raw logical-bank stride       0x2134 = 8500 bytes
startup CSI descriptor value  0x1F40 = 8000
runtime CSI descriptor value  field_a + 2 * field_b
```

Because the CSI descriptor quantity is dynamically reconfigured, `8000` must not be documented as the immutable physical CSI buffer size.

**STRONG INTERPRETATION:** `8500` is the allocated/logical lane stride, while the active capture quantity is variable and may leave space for trigger/ring offsets, edge processing or guard area. The 500-byte difference at startup remains unexplained and must not yet be assigned a fixed semantic meaning.

### Next static-analysis target

The highest-value next step is now to trace the runtime destination argument passed to `0x40040638 -> 0x40066030`.

Critical test:

```text
runtime_address == 0x42DF76D4 + bank_index * 0x2134 ?
```

If confirmed, this will prove that the eight logical raw banks are filled by successive CSI capture operations through the single hardware buffer-0 destination register.

Secondary remaining targets:

```text
identify where state +0x70E is cleared before rearm
name the pre/post capture helpers 0x40042C40 and 0x40042C1C
map official meaning of CSI +0x08, +0x70 and +0x74 fields
recover exact geometry semantics at +0x80/+0x84/+0x88
confirm whether capture enable auto-clears after frame completion
```

### Updated RE estimate

```text
FPGA startup and physical interface       about 95%
SMEC protocol on ARM side                 85-90%
Timebase configuration                    80-85%
Vertical configuration                    40-50%
Trigger/common configuration              45-55%
Signal generator                          30-40%
Acquisition control flow                  85-90%
Raw acquisition memory layout             85-90%
Waveform processing/buffer structures     70-75%
FPGA-to-ARM waveform transport             90-95%
Practical custom oscilloscope application about 76%
Full FPGA internals / replacement         25-30%
```

## 35. Confirmed per-bank CSI capture addressing (2026-10-23)

This milestone closes the remaining uncertainty about how the eight raw acquisition banks are populated. Earlier sections are retained as historical milestones; where sections 33-34 describe the runtime bank-address relationship as a hypothesis or critical test, this section supersedes that status.

### Exact runtime destination formula

**CONFIRMED:** Helper `0x400406C8` receives a bank index and computes the destination address using the same base and stride independently recovered from the raw-data consumer:

```c
uint8_t *destination =
    (uint8_t *)0x42DF76D4
    + bank_index * 0x2134;
```

The relevant instruction sequence is structurally:

```asm
movw    r3, #0x2134
ldr     r1, =0x42DF76D4
smulbb  ip, r3, r0
add     r1, r1, ip
b       0x40040638
```

`0x40040638` preserves `r1` as the runtime destination address and passes it through `0x40066030`, which programs CSI hardware destination buffer 0 at register offset `+0x50` using `address >> 2` encoding.

Thus the raw-bank producer and consumer use exactly the same addressing rule.

### Eight raw bank destinations

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

Each bank has a stride/allocation of `0x2134 = 8500` bytes. Eight banks occupy `0x109A0 = 68000` bytes and end at `0x42E08074` exclusive.

### One CSI frame/capture operation per logical raw bank

**CONFIRMED:** Function `0x400406E0` loops bank indices `0..7` and calls `0x400406C8` once for each bank.

Practical model:

```c
void capture_all_eight_banks(void)
{
    for (uint8_t bank = 0; bank < 8; ++bank)
        capture_raw_bank(bank);
}
```

Therefore the eight raw banks are **not** eight CSI hardware framebuffer registers. Stock firmware repeatedly reprograms CSI hardware buffer 0 to the next bank address and performs a separate capture operation for each logical bank.

Equivalent model:

```c
for (unsigned bank = 0; bank < 8; ++bank) {
    uint8_t *dst = RAW_ACQ_BASE + bank * RAW_BANK_SIZE;
    csi_set_buffer0(dst);
    capture_one_fpga_stream(bank);
}
```

### Four-bank acquisition variant

**CONFIRMED:** A second wrapper beginning at `0x400406FC` invokes the same bank-capture helper for indices `0..3` only.

```c
void capture_first_four_banks(void)
{
    for (uint8_t bank = 0; bank < 4; ++bank)
        capture_raw_bank(bank);
}
```

The exact semantic distinction between the 8-bank and 4-bank modes remains open.

**STRONG INTERPRETATION:** Likely candidates include channel count, ADC interleave/sample-rate mode, or another acquisition-lane configuration. This must be resolved from callers/state conditions rather than guessed from bank count alone.

### Producer/consumer symmetry

The CSI producer path is now:

```c
destination = RAW_ACQ_BASE + bank * RAW_BANK_SIZE;
```

while `0x4004073C` consumes:

```c
source = RAW_ACQ_BASE
       + bank * RAW_BANK_SIZE
       + state->raw_ring_origin_6f2
       + subblock_input_offset;
```

This is a direct producer/consumer match.

### Confirmed end-to-end per-bank transport

```text
bank index n
    ↓
FPGA bank/lane-selection path
    ↓
CSI runtime destination = 0x42DF76D4 + n * 0x2134
    ↓
CSI BUF0 register (+0x50) = destination >> 2
    ↓
CSI capture enable
    ↓
FPGA emits one 8-bit CSI acquisition stream
    ↓
CSI completion status +0x74 bit 1
    ↓
IRQ callback 0x40065EB0
    ↓
state +0x70E = 1; acknowledge 0x02
    ↓
next bank capture
```

The waveform transport mechanism is therefore no longer an open architectural question. Remaining work concerns the semantics and exact format of the transferred banks, not how the bytes reach ARM memory.

### Superseded questions

The following questions from earlier milestones are resolved:

```text
Who fills 0x42DF76D4?               R11 CSI capture hardware.
How are the eight slots selected?   Runtime destination = base + bank * 0x2134.
Are the slots CSI HW buffers?        No; stock firmware reuses hardware buffer 0.
How many captures fill all slots?    Eight separate capture operations.
Is there another capture variant?    Yes; a four-bank wrapper exists.
```

### Next static-analysis targets before active HW acquisition testing

The remaining high-value static questions are now narrower:

```text
1. Trace callers of the 8-bank and 4-bank wrappers and name the mode distinction.
2. Confirm how bank index reaches FPGA command 0x15 in the same per-bank transaction.
3. Identify where state +0x70E is cleared before each rearm.
4. Resolved later: 0x40042C40 / 0x40042C1C clear/set MMIO bit 0; they are not cache helpers.
5. Recover exact active capture quantity and bank payload format.
6. Determine the minimum initialization/configuration subset needed for a read-only custom acquisition test.
```

### Updated RE estimate

```text
FPGA startup and physical interface       about 95%
SMEC protocol on ARM side                 85-90%
Timebase configuration                    80-85%
Vertical configuration                    40-50%
Trigger/common configuration              45-55%
Signal generator                          30-40%
Acquisition control flow                  90-95%
Raw acquisition memory layout             95%
Waveform processing/buffer structures     70-75%
FPGA-to-ARM waveform transport             95-98%
Practical custom oscilloscope application about 78%
Full FPGA internals / replacement         25-30%
```

## 36. Final pre-Custom-App static baseline (2026-10-23)

This section is authoritative for implementation. Earlier sections preserve the RE chronology; where they conflict, the latest `CONFIRMED` statement and this section take precedence.

### Review corrections

```text
0x400660C8 is confirmed CSI initialization, not a delay/candidate helper.
R11 CSI is the confirmed raw-buffer producer.
Eight logical banks are not eight CSI hardware buffers.
0x40042C40 / 0x40042C1C clear/set MMIO bit 0, not cache state.
0x4003F9F8 / 0x4003FA04 change a PE GPIO configuration/direction state.
Normal runtime directly calls the bank 0..3 capture wrapper.
```

### Authoritative transport model

```text
FPGA control: PE GPIO command/status bus
FPGA samples: PE0 PCLK, PE2 HSYNC, PE3 VSYNC, PE8..PE15 DATA[7:0]
    -> R11 CSI
    -> CSI BUF0 +0x50 = physical_address >> 2
    -> 0x42DF76D4 + bank * 0x2134
    -> CSI status +0x74 bit 1, ACK 0x02
    -> 0x4004073C uint8 calibration/conversion
    -> uint16 waveform processing
```

### Authoritative bank layout

```text
RAW_ACQ_BASE    0x42DF76D4
RAW_BANK_STRIDE 0x2134 = 8500 bytes
RAW_BANK_COUNT  8
TOTAL_SPAN      0x109A0 = 68000 bytes
END_EXCLUSIVE   0x42E08074

bank 0  0x42DF76D4
bank 1  0x42DF9808
bank 2  0x42DFB93C
bank 3  0x42DFDA70
bank 4  0x42DFFBA4
bank 5  0x42E01CD8
bank 6  0x42E03E0C
bank 7  0x42E05F40
```

Stock firmware retargets CSI hardware buffer 0 for each logical bank. Wrappers exist for banks `0..3` at `0x400406FC`, banks `4..7` at `0x4004071C`, and banks `0..7` at `0x400406E0`. The normal acquisition runtime calls the `0..3` wrapper. The low/high grouping must not yet be labeled CH1/CH2 or a particular interleave mode.

### Confirmed per-bank primitive

```text
prepare CSI/capture state
apply FPGA-side bank selection
switch shared PE interface state
program CSI buffer 0
clear bit 0 at 0x01C20C10 + index * 0x10
enable CSI
assert PE capture-handshake/configuration state
wait for completion
release PE state
set bit 0 at 0x01C20C10 + index * 0x10
restore shared PE state
finish CSI state
```

The PE handshake signal and MMIO block at `0x01C20C10` remain semantically unnamed.

### Completion and Custom App simplification

Stock callback `0x40065EB0`, registered for source `0x74`, tests CSI status `+0x74` bit 1, sets `state +0x70E`, and acknowledges `0x02`. No independent direct byte store clearing `state +0x70E` was found, so the first Custom App must not depend on that software flag lifecycle.

Use direct polling:

```text
ACK stale status 0x02
program CSI BUF0
configure and enable CSI
assert capture handshake
poll status +0x74 bit 1 with finite timeout
release handshake
ACK 0x02
inspect destination buffer
```

### Minimal first Custom App

```text
1. display diagnostics
2. FPGA GPIO init and known-good ID probe
3. required MS5351M clock init
4. CSI clock/reset, geometry and PE pinmux
5. fill one 8500-byte bank-0 buffer with 0xAA
6. select bank 0
7. program CSI BUF0
8. ACK stale completion
9. enable CSI and assert handshake
10. poll with timeout
11. release and ACK
12. show CSI registers, result, changed-byte count and first 64 bytes
```

Success means: valid FPGA ID, completion bit observed, no timeout, and a repeatably changed nontrivial buffer. A recognizable waveform is not required yet.

### Safety constraints

```text
No infinite waits.
Bank 0 only in the first test.
Do not enable full SMEC/vertical/trigger/processing at once.
Always display status and destination registers on failure.
Change one variable per iteration.
Keep the original app recoverable through the boot manager.
```

### Remaining practical unknowns

```text
1. Minimum FPGA full-state initialization required before FPGA acquisition/CSI streaming becomes operational in the standalone Custom App.
2. Whether `0x64..0x6B` is required for acquisition, and whether it is a secure-device service, measurement engine, or general asynchronous FPGA job protocol.
3. Exact fast-timebase interleave/ADC-core ordering across the eight slots. External hardware testing establishes CH1=`0..3`, CH2=`4..7` in dual-channel mode and CH1=`0..7` in one single-channel test.
4. Exact meaning of the 2356 bytes outside the externally verified 6144-byte active payload within each 8500-byte logical slot.
5. CSI active-quantity relationship to FPGA command 0x22 across acquisition modes.
6. Electrical names of shared PE controls.
7. Official R11 CSI field and interrupt names.
8. Minimum calibration/conversion needed for a recognizable waveform.
```

### Decision

```text
GO: single-bank polling CSI smoke test with timeout.
NO-GO: full oscilloscope clone in the first iteration.
```

### Current estimate

```text
FPGA startup/physical interface           about 95%
SMEC ARM-side protocol                    85-90%
Timebase configuration                    80-85%
Vertical configuration                    40-50%
Trigger/common configuration              45-55%
Signal generator                          30-40%
Acquisition control flow                  90-95%
Raw acquisition memory layout             about 95%
Waveform processing/buffers               70-75%
FPGA-to-ARM waveform transport             95-98%
Practical custom oscilloscope application about 78%
Full FPGA internals/replacement            25-30%
```

### 37. Runtime CSI destination trace and low-level MMIO confirmation (2026-10-25)

This milestone extends sections 34-36 with runtime DPOSDBG evidence and instruction-level reconstruction of the complete CSI destination-address write path. It does not change the no-commit policy for the current acquisition experiment branch.

#### Runtime confirmation of the logical-bank destination sequence

**HW-CONFIRMED:** The v6 minimum CSI baseline trace records repeated `CSI_DESTINATION` events at call site `0x40040660`. Each observed batch walks the same eight destination addresses in order:

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

The stride is exactly `0x2134 = 8500` bytes. The eight-entry sequence repeats as a batch during normal RUN operation. This independently confirms at runtime the bank-address formula previously recovered statically.

The repeated trace form is:

```text
CSI_DESTINATION  <destination>  0x40040660  0x40040660
```

**STATIC-CONFIRMED:** `0x40040660` is not a stored buffer pointer. It is the ARM call site:

```asm
4004065c: mov r0, r5
40040660: bl  0x40066030
```

Thus `r0` contains the runtime destination address at this call. A raw search for little-endian `0x40040660` in EXE2 correctly finds no pointer occurrence because the address identifies the `BL` instruction itself.

#### Exact bank-address generator and capture wrappers

**STATIC-CONFIRMED:** `0x400406C8` computes:

```c
destination = 0x42DF76D4 + bank_index * 0x2134;
```

The base literal at `0x400413B4` is exactly `0x42DF76D4`.

The function sequence is structurally:

```asm
400406c8: movw   r3, #0x2134
400406cc: ldr    r1, =0x42DF76D4
400406d0: smulbb ip, r3, r0
400406d4: add    r1, r1, ip
400406d8: b      0x40040638
```

`0x400406DC` loops over indices `0..7`, while `0x400406FC` loops over indices `0..3`. The previously documented high-bank wrapper remains separate. The semantic distinction between the four-bank and eight-bank modes must still be derived from their callers/state conditions rather than inferred solely from bank count.

#### Complete runtime destination path

**STATIC-CONFIRMED:** The per-bank destination path is now reconstructed end to end:

```text
0x400406C8
    destination = 0x42DF76D4 + bank * 0x2134
        |
        v
0x40040638
    preserves index and destination
        |
        v
0x40040660
    r0 = destination
    BL 0x40066030
        |
        v
0x40066030
    forwards destination as the low word of a stack-passed 64-bit address
        |
        v
0x40065A60
    computes CSI register address
    encodes destination as address >> 2
        |
        v
0x40065894
    generic read-modify-write helper
        |
        +--> 0x4006582C  read32(addr)
        |
        +--> 0x40065834  write32(addr, value)
        v
MMIO
```

At `0x40066030`, the wrapper stores `{destination, 0}` on the stack. After the callee prologue, `0x40065A60` reloads this pair using `LDRD [sp,#8]`, confirming that the low 32 bits are the physical destination and the high 32 bits are zero in this path.

#### CSI destination-register addressing and encoding

**STATIC-CONFIRMED:** `0x40065A60` obtains a controller/device base through the pointer table rooted at `0x42E68C50`, then computes the destination-register address as:

```c
reg = controller_base[controller]
    + channel * 0x100
    + buffer_index * 4
    + 0x50;
```

For the runtime path reached from `0x40066030`, the observed indices are zero, so the effective register is:

```c
CSI_DST_REG = controller_base[0] + 0x50;
```

The physical destination is encoded as a 32-bit-word address:

```c
register_value = destination >> 2;
```

For bank 0:

```text
physical destination  0x42DF76D4
register value         0x10B7DDB5
```

The complete observed effect is therefore:

```c
MMIO32(controller_base[0] + 0x50) = destination >> 2;
```

**IMPORTANT:** `0x42E68C50` is a RAM-resident pointer table/root used to obtain the controller base. It is not itself the CSI MMIO base. The runtime value `*(uint32_t *)0x42E68C50` remains a useful independent confirmation target. Existing external/local evidence identifying the R11 CSI block at `0x01CB4000` is retained, but this particular reconstruction derives the register address through the runtime pointer table rather than assuming that constant.

#### Direct MMIO primitives

**STATIC-CONFIRMED:** The low-level helpers are trivial direct memory accesses:

```asm
4006582c: ldr r0, [r0]
40065830: bx  lr
```

Equivalent to:

```c
uint32_t read32(uint32_t addr)
{
    return *(volatile uint32_t *)addr;
}
```

and:

```asm
40065834: str r1, [r0]
40065838: bx  lr
```

Equivalent to:

```c
void write32(uint32_t addr, uint32_t value)
{
    *(volatile uint32_t *)addr = value;
}
```

`0x40065894` is a generic read-modify-write helper:

```c
void reg_update_bits(uint32_t addr, uint32_t mask, uint32_t value)
{
    uint32_t old = read32(addr);
    uint32_t updated = (old & ~mask) | value;
    write32(addr, updated);
}
```

The CSI destination path supplies `mask = 0xFFFFFFFF`, making the operation an effective full-register write of `destination >> 2`.

#### Capture-completion synchronization cross-check

**STATIC-CONFIRMED:** The acquisition transaction waits on the byte at:

```text
0x40545576 = 0x40544E68 + 0x70E
```

until it becomes nonzero. The tail path at `0x40066108` clears the same byte back to zero before continuing through the CSI helper layer. This is consistent with the callback/IRQ completion lifecycle documented in section 34.

The observed transaction around `0x40040638` is therefore structurally:

```text
prepare/select bank
program CSI destination
prepare capture state
submit/enable CSI operation
assert FPGA/shared-PE capture state
wait for CSI completion flag
post-capture cleanup
release capture state
clear/finalize CSI completion state
```

Exact semantic names of every helper in this sequence remain intentionally conservative where not separately proven.

#### Evidence boundary and implementation consequence

The following are now **CONFIRMED** locally by combined static analysis and runtime trace:

- the eight raw destinations are generated by `base + bank * 0x2134`;
- the runtime trace observes those exact destinations at `0x40040660`;
- stock firmware retargets the same CSI hardware destination register between logical-bank captures;
- destination addresses are written to hardware as `physical_address >> 2`;
- `0x4006582C` and `0x40065834` are direct 32-bit read/write primitives;
- `0x40065894` is the read-modify-write helper used by the CSI register layer.

Still **OPEN / not to be overnamed**:

- the official vendor field name for CSI offset `+0x50` in the exact R11 documentation;
- the runtime contents of `*(uint32_t *)0x42E68C50` as an independent confirmation of the controller base;
- the semantic distinction selecting the four-bank, high-four-bank and eight-bank wrappers;
- the exact relationship between configured CSI active quantity and the externally observed 6144-byte active sample payload;
- the minimum FPGA/service initialization needed to make the same capture sequence succeed in the standalone Custom App.

#### Immediate implementation relevance

For a later standalone acquisition implementation, programming CSI destination buffer 0 must use the encoded word address, not the byte address:

```c
uint32_t dst = RAW_ACQ_BASE + bank * RAW_BANK_STRIDE;
MMIO32(CSI_BASE + 0x50) = dst >> 2;
```

Writing `0x42DF76D4` directly to the destination register would therefore be incorrect for bank 0; the stock encoding is `0x10B7DDB5`.

This milestone strengthens the section-36 implementation baseline but does not yet justify committing an acquisition implementation. The current strategy remains to keep accumulating confirmed pieces until a minimal acquisition path works end to end, then commit the coherent milestone.

#### Addendum: local confirmation of the physical CSI MMIO base

**STATIC-CONFIRMED:** Function `0x400658BC` is the two-instance controller-base setter. It validates `instance < 2` and stores the supplied base into the table rooted at `0x42E68C50`:

```c
int csi_set_base(unsigned instance, uint32_t base)
{
    if (instance >= 2)
        return -1;
    ((uint32_t *)0x42E68C50)[instance] = base;
    return 0;
}
```

The CSI initialization path at `0x400660C8` constructs `0x01CB4000` directly in `r1` as `0x01C00000 | 0x000B4000`, sets `r0 = 0`, and calls `0x400658BC`. Therefore:

```text
controller_base[0] = 0x01CB4000
```

This locally closes the remaining base-address ambiguity from the preceding section. The stock destination-register path is consequently:

```text
CSI instance 0 base       0x01CB4000
CSI destination register  0x01CB4050  (base + 0x50)
raw bank destination      0x42DF76D4 + bank * 0x2134
hardware register value   destination >> 2
```

Equivalent operation:

```c
uint32_t dst = 0x42DF76D4u + bank * 0x2134u;
*(volatile uint32_t *)0x01CB4050u = dst >> 2;
```

For bank 0:

```text
destination          0x42DF76D4
encoded destination  0x10B7DDB5
MMIO write target    0x01CB4050
```

`0x42E68C50` is therefore confirmed as the RAM-resident controller-base table, not the CSI peripheral itself. Entry 0 is initialized to the physical R11 CSI base `0x01CB4000`. The setter accepts indices 0 and 1, establishing two slots in this driver-level base table; this fact alone does not require assigning a specific physical role to instance 1.

With this confirmation, the CSI destination-address chain is considered **closed for the current acquisition implementation baseline**. Further reverse engineering should prioritize the FPGA/bank-selection and shared-PE capture steps surrounding `0x40040638`, rather than spending additional effort proving the already established BUF0 address write.

### 38. Final per-bank FPGA/PE capture sequencing before standalone smoke test (2026-10-25)

This milestone closes the remaining statically required FPGA/PE-side sequencing around `0x40040638` before the next timeout-protected single-bank Custom App acquisition test. It supersedes the remaining vague references in sections 35-37 to generic bank selection, PE capture state, and unnamed pre/post-capture MMIO transitions.

#### FPGA register 0x1F receives the per-capture bank index

**STATIC-CONFIRMED:** `0x4003FA94(value)` writes its argument directly through the FPGA parallel command bus:

```text
FPGA selector 0x1F
FPGA data     value
```

`0x40040638` calls this helper with its original `r0`, which is the same bank index used by `0x400406C8` to derive the CSI RAM destination. Therefore a single per-bank capture binds the same index to both sides:

```c
fpga_write_reg(0x1F, bank);
uint32_t dst = 0x42DF76D4u + bank * 0x2134u;
MMIO32(0x01CB4050) = dst >> 2;
```

This is distinct from the previously mapped FPGA command `0x15`. Command `0x15` participates in a higher-level acquisition selector path; command `0x1F` is the directly observed bank-index write inside the per-bank CSI capture primitive. Their exact semantic relationship remains open and they must not be collapsed into one register meaning.

#### Cyclic capture/read-position preparation

**STATIC-CONFIRMED:** `0x4003FAB8` computes a position from three acquisition-state fields:

```c
uint32_t period = state->field_520;
uint32_t offset = state->field_63C;
uint16_t origin = state->field_6F2;
uint32_t position = (period + offset - origin) % period;
```

The modulo is implemented explicitly with `UDIV`, multiply and subtraction. The result is tail-called into the already mapped `0x4003F988` path, which applies the cyclic acquisition position through FPGA commands `0x20` and `0x21`.

The same `state + 0x6F2` field also contributes to the ARM raw-buffer source address used by `0x4004073C`. This establishes a direct producer/consumer synchronization relationship around the common ring-origin value, while its exact user-facing semantic name remains conservative.

#### Shared PE interface switches between FPGA GPIO control and CSI sample modes

**STATIC-CONFIRMED:** `0x40049094(mode)` is a cached interface-mode dispatcher. It stores the current mode in the byte at `0x42E68D19` and performs no hardware transition when the requested mode already matches the cached state.

For nonzero mode:

```text
0x40049094(1)
    -> 0x400660C0
    -> 0x40065EF8(0)
```

The destination helper configures the known CSI pins to PE alternate function 2. Thus mode `1` transfers the shared PE sample pins from the FPGA GPIO control-plane configuration into the R11 CSI sample-plane configuration.

For zero mode:

```text
0x40049094(0)
    -> 0x40049110
```

`0x40049110` uses `PE_CFG0 = 0x01C20890` and restores PE0, PE1, PE2 and PE3 to GPIO function 0. It then establishes the observed GPIO idle levels:

```text
PE0 = HIGH
PE1 = LOW
```

and clears the FPGA parallel-bus software state at:

```text
0x42E68D18 <- 0
0x42E68D14 <- 0
```

Combined with the already confirmed CSI pin configuration, the shared interface lifecycle is:

```text
GPIO control mode
    PE0..PE3 controls
    PE8..PE15 DATA[7:0]
        |
        | 0x40049094(1)
        v
CSI sample mode
    PE0       CSI PCLK
    PE2       CSI HSYNC
    PE3       CSI VSYNC
    PE8..PE15 CSI DATA[7:0]
    PE1       remains available as the observed GPIO capture-level signal
        |
        | 0x40049094(0)
        v
GPIO control mode restored
```

#### PE1 is held HIGH for the active CSI capture interval

**STATIC-CONFIRMED:** Both wrappers use the literal `PE_CFG0 = 0x01C20890`; the generic helpers operate on `base + 0x10 = PE_DATA = 0x01C208A0`.

`0x4003F9F8` passes pin index 1 to the GPIO-set helper `0x40063274`, producing:

```c
MMIO32(0x01C208A0) |= (1u << 1);   /* PE1 HIGH */
```

`0x4003FA04` passes pin index 1 to the GPIO-clear helper `0x40063288`, producing:

```c
MMIO32(0x01C208A0) &= ~(1u << 1);  /* PE1 LOW */
```

Within `0x40040638`, PE1 is set HIGH immediately after CSI capture enable and remains HIGH throughout the wait for CSI completion. It is set LOW immediately after completion. Its exact electrical name remains **OPEN**; `capture gate/handshake` is a useful working description, not an asserted vendor signal name.

#### Exact pre/post-capture MMIO transition

**STATIC-CONFIRMED:** `0x40042C40(instance)` and `0x40042C1C(instance)` operate on:

```c
reg = 0x01C20C10u + instance * 0x10u;
```

The capture path passes `instance = 0`.

Before CSI enable, `0x40042C40(0)` clears bit 0:

```c
MMIO32(0x01C20C10) &= ~1u;
```

After CSI completion and PE1 deassertion, `0x40042C1C(0)` sets bit 0:

```c
MMIO32(0x01C20C10) |= 1u;
```

The physical register address and polarity are therefore confirmed. The official hardware-block/register-field name remains deliberately unnamed until matched against authoritative R11/V3s register documentation.

#### Final statically reconstructed per-bank capture primitive

The essential stock sequence around one logical bank is now:

```text
1. Compute cyclic position:
       (state[0x520] + state[0x63C] - state[0x6F2]) % state[0x520]

2. Apply cyclic position through FPGA 0x20 / 0x21.

3. FPGA 0x1F <- bank index.

4. Switch shared PE sample pins from FPGA GPIO control mode to CSI function 2.

5. Program CSI BUF0:
       destination = 0x42DF76D4 + bank * 0x2134
       MMIO32(0x01CB4050) = destination >> 2

6. Clear bit 0 at 0x01C20C10.

7. Enable CSI capture.

8. Set PE1 HIGH.

9. Wait for CSI completion:
       CSI status 0x01CB4074 bit 1 / stock callback state +0x70E.

10. Set PE1 LOW.

11. Set bit 0 at 0x01C20C10.

12. Restore shared PE pins to FPGA GPIO control mode and reset the bus software state.

13. Finalize/clear CSI completion state and acknowledge completion as required.
```

The stock firmware waits through its callback-backed software completion flag. The first standalone Custom App must instead poll the CSI hardware completion status with a finite timeout so that a missing FPGA stream cannot hang the device.

#### Smoke-test readiness

**DERIVED:** No additional static target is currently mandatory before assembling the next single-bank acquisition smoke test. The critical address, ordering and polarity information required to reproduce the per-bank transport is now locally established.

The first test should remain deliberately narrow:

```text
bank 0 only
known-good FPGA GPIO/ID baseline
required clock and CSI initialization
fill destination with sentinel bytes
apply the reconstructed FPGA position/bank setup
perform GPIO -> CSI transition
program CSI BUF0 with word-address encoding
clear 0x01C20C10 bit 0
ACK stale CSI completion
CSI enable
PE1 HIGH
poll CSI +0x74 bit 1 with finite timeout
PE1 LOW
set 0x01C20C10 bit 0
ACK completion
restore GPIO control mode
count changed destination bytes and dump a small prefix
```

A first success criterion remains intentionally modest: CSI completion must occur and the bank-0 destination must change repeatably from its sentinel pattern. A calibrated or recognizable waveform is not required for this first transport smoke test.

#### Remaining risks after static closure

The principal remaining unknown is no longer the mechanics of the CSI transfer. It is whether the standalone Custom App has reproduced enough of the stock FPGA/acquisition initialization state to make the FPGA emit a valid stream. In particular, the role, if any, of the broader `0x64..0x6B` asynchronous FPGA job/service subsystem in acquisition remains unproven. A timeout or unchanged buffer in the next smoke test must therefore be treated as evidence of missing initialization/state, not as evidence against the now-confirmed CSI transport sequence.

### 39. HW-confirmed dynamic CSI capture window and RAM payload (2026-10-26)

This milestone supersedes the remaining uncertainty in sections 36-38 about whether the stock FPGA actually emits a live waveform stream over the PE/CSI sample plane and whether CSI writes real acquisition bytes into the raw-bank RAM. The result is locally **HW-CONFIRMED** with resident DPOSDBG instrumentation while the stock EXE2 remained operational.

#### Startup checkpoint result: continuous PE activity is not present between init functions

A read-only DPOSDBG checkpoint experiment sampled PE0, PE2, PE3 and PE8..PE15 one million times immediately after each of these confirmed startup calls:

```text
FPGA_Start
SMEC_Init
PowerInit
HorizontalInit
VerticalInit
HardCostTimeInfoInit
Gen_Init
0x4004F134
0x40051CF0
SysConfigExec
0x4003D6AC
0x40049A9C
0x40052098
0x4003CF58
```

No checkpoint observed GPIO-register activity during its bounded window:

```text
changed_mask = 0x00000000
```

This does **not** mean the physical sample pins are inactive during acquisition. The later CSI-window experiment proves that the sample pins are dynamically muxed to alternate function 2 only inside a short capture transaction, and PE_DATA does not provide a useful physical-pad activity indication while CSI owns the pins.

Useful pinmux transitions observed in the startup checkpoints include:

```text
through VerticalInit:
    PE_CFG0 = 0x77771111
    PE_CFG1 = 0x00000000

following HardCostTimeInfoInit / later startup work:
    PE_CFG1 = 0x11111111

later runtime transition:
    PE_CFG1 returns to 0x00000000
```

`SMEC_Init()` changes the static PE_DATA state from the earlier value around `0x00000001` to `0x00000701`, but does not itself create a continuously observable GPIO sample stream.

#### HW-CONFIRMED three-phase CSI capture window

DPOSDBG instrumented the stock per-bank capture path around:

```text
0x40040660  runtime CSI destination programming
0x4004066C  CSI/capture-control path
0x40040684  post-completion path before final disable/restore
```

The first captured bank produced the following three phases.

```text
DESTINATION_READY:
    PE_CFG0          0x77772212
    PE_CFG1          0x22222222
    CSI IF config    0x01060400
    CSI mode         0x00000000
    CSI hsize        0x00000A00
    CSI destination  0x10B7DDB5
    CSI IRQ enable   0x00000002
    CSI status       0x00000000
    geometry         0x18000000 / 0x00010000 / 0x2374287C

CAPTURE_ACTIVE:
    PE_CFG0          0x77772212
    PE_CFG1          0x22222222
    CSI mode         0x00000001
    CSI destination  0x10B7DDB5
    CSI status       0x00000000

CAPTURE_DONE_PRE_DISABLE:
    PE_CFG0          0x77772212
    PE_CFG1          0x22222222
    CSI mode         0x00000000
    CSI destination  0x10B7DDB5
    CSI status       0x00000081
```

This locally proves the dynamic capture lifecycle:

```text
PE sample pins -> function 2
CSI destination programmed
CSI mode 0 -> 1
capture transaction completes
CSI mode 1 -> 0
post-completion status = 0x81
```

The destination register uses the already reconstructed word-address encoding:

```text
0x10B7DDB5 << 2 = 0x42DF76D4
```

Thus the hardware state observed during the live capture agrees exactly with the bank-0 destination derived statically and seen in the earlier runtime destination trace.

#### Important correction: PE_DATA is not a valid activity probe while CSI owns the pins

During all three CSI phases, direct reads of PE_DATA remained static over 4096 reads even though the capture completed successfully. Therefore:

```text
PE_DATA changed_mask == 0
```

must not be interpreted as absence of physical PE sample traffic when:

```text
PE_CFG0 = 0x77772212
PE_CFG1 = 0x22222222
```

The R11 CSI peripheral owns the alternate-function pads at that time. CSI mode/status and destination-RAM modification are the appropriate independent evidence of the physical sample transfer.

#### HW-CONFIRMED CSI write into raw bank 0

A second read-only DPOSDBG experiment compared the programmed destination buffer immediately after destination setup and again after CSI completion. CPU cache lines covering the destination were invalidated before each comparison so the post-DMA read reflected RAM contents.

Observed destination:

```text
CSI destination register  0x10B7DDB5
physical RAM address      0x42DF76D4
logical slot size         0x2134 = 8500 bytes
```

Whole-buffer FNV-1a hashes:

```text
before capture  0x6355A6D5
after capture   0xB91D382E
changed         YES
```

The buffer was divided into 34 blocks of 256 bytes for coarse change localization. Exactly these blocks changed:

```text
0..23
```

Therefore at least the first:

```text
24 * 256 = 6144 bytes = 0x1800
```

of the 8500-byte logical slot were modified by the capture, while blocks 24..33 remained unchanged in this experiment.

This locally reproduces the previously external observation that the active payload is `0x1800 = 6144` bytes inside an `0x2134 = 8500` byte logical slot. The remaining `2356` bytes must still not be assigned a fixed meaning without further evidence.

The numeric match between the observed 6144-byte active range and:

```text
CSI geometry register snapshot  0x18000000
```

is notable, but the official field semantics of that geometry register remain open.

#### Raw sample appearance

Before the first capture, the tested bank-0 prefix was zero-filled. After completion the prefix contained dense 8-bit values, predominantly around `0x86..0x89` in this particular acquisition:

```text
before:
00 00 00 00 00 00 00 00 ...

after:
00 00 88 87 88 88 87 88 88 87 87 86 88 88 87 88
88 89 87 87 88 87 88 89 87 88 87 88 88 88 88 87
...
```

The far end of the 8500-byte allocation remained zero in this capture, consistent with the block-hash result that only blocks 0..23 changed.

**DERIVED:** The observed changed region contains real acquisition payload bytes rather than merely register/configuration state. Determining exact ADC-core/interleave/channel semantics remains separate work.

#### Completed local transport proof

The following end-to-end path is now locally **HW-CONFIRMED**, not merely statically reconstructed:

```text
GOWIN FPGA
    |
    | PE0 / PE2 / PE3 / PE8..PE15 on alternate function 2
    v
R11 CSI @ 0x01CB4000
    |
    | destination +0x50 = physical_address >> 2
    | mode +0x08 transitions 0 -> 1 -> 0
    | completion lifecycle observed
    v
0x42DF76D4 bank-0 RAM
    |
    | at least 0x1800 = 6144 bytes changed
    v
raw uint8 acquisition payload
```

This closes the previous active-hardware uncertainty about whether the FPGA sample producer actually reaches ARM RAM through CSI.

#### Consequence for standalone Custom App acquisition

A first standalone single-bank CSI smoke test is now justified. It no longer needs to use PE_DATA transitions as the success criterion.

The minimum success criteria should be:

```text
1. valid FPGA ID / control-plane baseline
2. required FPGA/service initialization applied
3. PE sample pins configured to CSI function 2
4. CSI destination programmed using address >> 2
5. CSI mode enters active state
6. completion observed with a finite timeout
7. destination cache invalidated after capture
8. destination buffer differs from its pre-capture/sentinel state
9. changed region contains nontrivial raw bytes
```

Useful stock reference values from the confirmed capture are:

```text
PE_CFG0             0x77772212
PE_CFG1             0x22222222
CSI IF config       0x01060400
CSI hsize           0x00000A00
CSI IRQ enable      0x00000002
CSI destination     physical_address >> 2
CSI active mode     0x00000001
post-completion     CSI status 0x00000081 observed
active payload      0x1800 = 6144 bytes in this tested mode
logical slot stride 0x2134 = 8500 bytes
```

The stock callback path still establishes status bit 1 as the completion event that is acknowledged with `0x02`; the later `0x81` value is the state observed at the post-completion checkpoint after the stock callback/ack path has already participated.

#### Revised immediate next step

Proceed with the single-bank Custom App smoke test. Reproduce only the minimum confirmed state needed for one bank, poll completion with a finite timeout, invalidate the destination cache, and report buffer change/hash/prefix diagnostics. If this fails, treat the result as evidence of missing FPGA/SMEC/acquisition initialization rather than reopening the already HW-confirmed PE/CSI/RAM transport path.

## 40. Standalone custom FPGA acquisition milestone (2026-10-29)

This section is authoritative for the currently supported fixed-profile standalone runtime. It supersedes earlier statements that the Custom App had not observed CSI completion, that the minimum FPGA/service initialization remained unknown, or that a stock-assisted acquisition path was still required.

### 40.1 Current code-level architecture

Where the mapping is established, use the current project names instead of address-only or historical phase names:

```text
FPGA initialization                    fpga_init()
FPGA parallel control bus              fpga_bus_*()
FPGA register/protocol operations      fpga_protocol_*()
MS5351M clock initialization           ms5351_*()
R11 CSI initialization and control     r11_csi_*()
minimal SMEC startup path              smec_init()
fixed-profile configuration            profile_init()
acquisition readiness                  acq_prepare()
stream geometry and quantity setup     acq_configure_stream()
single-bank CSI transfer               acq_capture()
```

Historical stock addresses remain valid reverse-engineering anchors, but descriptions of the working custom runtime should prefer these code-level names.

### 40.2 Standalone initialization chain

The successful standalone path is:

```text
custom Timer0 + AVS initialization
        -> fpga_init()
        -> smec_init()
        -> profile_init()
        -> acq_capture()
        -> CSI completion
        -> full 0x1800-byte RAM payload
```

No stock EXE2 acquisition function executes in this path.

### 40.3 System_Init Part 2 resolved as AVS timebase setup

The previously unnamed `System_Init` Part 2 prerequisite at `0x40042CD8` has now been identified precisely enough for standalone reproduction.

Static call chain:

```text
System_Init              0x40042D28
  Part 1                 0x40042C94
  Part 2 / AVS init      0x40042CD8
    channel 0 setup      0x40042D40  divisor 24000
    channel 1 setup      0x40042D4C  divisor 24
    common setup         0x40042D54
```

Confirmed register effects:

```text
0x01C20144 |= 0x80000000
AVS channel 0 divisor = 24000
AVS channel 1 divisor = 24
0x01C20C8C = 0x00175DBF
```

The stock AVS delay primitive clears counter 0 at `0x01C20C84`, enables counter 0 through bit 0 of `0x01C20C80`, and waits for `(counter0 << 1)` to reach the requested delay units.

This matches the custom `timer_avs_delay_units()` implementation. The missing state was the AVS timebase initialization, not the delay loop itself.

**STATIC-CONFIRMED:** stock `System_Init` configures both AVS divider channels before the relevant delay path is used.

**HW-CONFIRMED:** after adding the AVS setup to custom timer initialization, `profile_init()` completes all seven fixed-profile steps instead of stalling in `timer_avs_delay_units(2)`.

### 40.4 FPGA initialization status

The custom FPGA initialization path is now sufficient for real acquisition, not only for the diagnostic ID handshake.

The working custom path includes:

```text
FPGA control-bus setup and ID probe
R11 CSI initialization
shared PE GPIO/CSI mode handling
MS5351M clock setup
fixed CSI receiver state required by the supported profile
```

**HW-CONFIRMED:** `fpga_init()` succeeds as part of the standalone path that reaches CSI completion and transfers the complete active payload into RAM.

The earlier boundary remains important: the minimal diagnostic ID sequence alone is not equivalent to `fpga_init()`. The proven result applies to the complete current custom initialization path.

### 40.5 Current fixed-profile stream configuration

The supported standalone profile uses:

```text
APP_STATE +0x640          0x17FA
APP_STATE +0x6F2          0x0003
CSI payload / geometry    0x1800 bytes
FPGA selector 0x22 value  0x300E
```

The corresponding arithmetic remains:

```text
CSI geometry  = A + 2*B       = 0x17FA + 2*3 = 0x1800
FPGA quantity = 20 + 2*(A+B)  = 0x300E
```

The custom implementation expresses the downstream operation through:

```text
r11_csi_configure_fixed_stream_geometry()
fpga_protocol_stream_set_quantity(0x300E)
```

The current values are fixed-profile constants. Their general derivation for other timebases, channel modes and acquisition modes remains open.

### 40.6 Current custom per-bank capture sequence

The working `acq_capture()` path performs the following sequence:

```text
1. configure fixed CSI stream geometry
2. write FPGA stream quantity 0x300E
3. prepare cyclic capture position
4. select the FPGA bank
5. switch the shared PE interface to CSI mode
6. program CSI buffer 0
7. acknowledge stale CSI completion
8. stop/gate Timer0 as required by the reconstructed capture sequence
9. enable CSI capture
10. assert the FPGA capture gate
11. wait for CSI completion with a finite bounded poll
12. release the FPGA capture gate
13. restore Timer0 state
14. acknowledge CSI completion
15. disable CSI capture
16. restore the shared PE interface to FPGA GPIO mode
```

This is the custom equivalent of the previously reconstructed stock per-bank transport role. The implementation no longer calls stock `0x400405E0`, stock acquisition prepare `0x4003FA10`, or stock per-bank primitive `0x400406C8`.

### 40.7 Hardware-confirmed standalone result

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

The observed CSI buffer register is the word-address encoding of the standalone capture buffer. The complete expected range changed:

```text
0x0000 .. 0x17FF = 0x1800 bytes
```

**HW-CONFIRMED:** the FPGA emits a valid stream through the shared PE/CSI sample plane, the R11 CSI peripheral signals completion, and the complete fixed-profile active payload reaches custom-app RAM.

This proof is stronger than `FPGA READY`, CSI register parity, or a one-byte destination change. It establishes the full custom control-plane and sample-plane path for the supported fixed profile.

### 40.8 SMEC and FPGA protocol boundary

The successful path still includes `smec_init()`. Therefore this result does not show that SMEC is optional.

It does show that the custom minimal SMEC implementation is sufficient for proper FPGA operation through a real completed acquisition. No unreconstructed stock SMEC function is required by the current fixed-profile capture path.

The key-like / "magic" register sequences used by `profile_init()` remain part of the working FPGA apply protocol. Their exact semantic role is still open. Similar sequences may recur when additional FPGA commands are identified, so they should be treated as protocol landmarks rather than discarded as one-off SysConfig artifacts.

### 40.9 Superseded statements and plans

The following older statements are now superseded for the supported fixed profile:

```text
"No Custom App test has yet observed CSI completion or changed capture bytes."
"The minimum FPGA full-state initialization required for streaming is unknown."
"Proceed with the first single-bank CSI smoke test."
"Custom acquisition still depends on stock stream/acquisition helpers."
"A stock-assisted fallback is required for the current capture path."
```

The following are now resolved:

```text
Does custom FPGA initialization support real capture?     YES
Does custom SMEC support real capture?                    YES
Does the fixed custom profile reach completion?           YES, 0x7F steps
Does CSI signal hardware completion?                      YES
Does RAM receive the full active payload?                 YES, 0x1800 bytes
Is stock EXE2 required for this capture path?              NO
```

### 40.10 Current authoritative fixed-profile architecture

```text
CONTROL PLANE
ARM/R11
  -> fpga_bus_*()
  -> fpga_protocol_*()
  -> custom FPGA registers, profile and capture commands
  -> GOWIN FPGA

CLOCK / SERVICE CONFIGURATION
ARM/R11
  -> ms5351_*()
  -> smec_init()
  -> profile_init()
  -> GOWIN FPGA and associated front-end/service path

SAMPLE PLANE
GOWIN FPGA
  -> shared PE pins in CSI function 2
  -> R11 CSI @ 0x01CB4000
  -> custom 0x1800-byte aligned RAM buffer
  -> HW DONE / CSI status
```

### 40.11 Updated development direction

The transport and fixed-profile acquisition mechanics are no longer the principal blocker. The next FPGA work should proceed in this order:

```text
1. preserve the current standalone acquisition as the golden baseline
2. generalize profile derivation beyond fixed +0x640/+0x6F2 values
3. identify and name additional FPGA commands
4. correlate additional commands with SMEC jobs and key/magic apply sequences
5. restore analog frontend/channel configuration
6. validate sample interpretation, calibration and measurement correctness
7. expand from one fixed capture profile to selectable operating modes
```

The current baseline must continue to require all of:

```text
FPGA INIT       YES
SMEC PASS       YES
PROFILE PASS    YES
STEPS DONE      0x0000007F
CAPTURE PASS    YES
HW DONE         YES
RAM CHANGED     YES
CHANGED BYTES   0x00001800
FIRST CHANGE    0x00000000
LAST CHANGE     0x000017FF
```
