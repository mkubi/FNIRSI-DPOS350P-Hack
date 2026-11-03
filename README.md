# FNIRSI DPOS350P Firmware Reverse Engineering

Reverse-engineering notes, analysis tools, and technical documentation for the FNIRSI DPOS350P oscilloscope firmware and hardware interfaces.

This repository is the **documentation and reverse-engineering repository**. It preserves stock-firmware analysis, hardware findings, protocol reconstruction, and experiment results. The separately maintained **FNIRSI-DPOS350P-Boot** repository contains the custom boot chain, bare-metal drivers, standalone applications, and implementation work derived from these findings.

## Disclaimer

This is experimental reverse-engineering work. DPOS350P hardware revisions and firmware builds differ, so absolute addresses, component vendors, and recovered behavior must always be interpreted in the context of the analyzed build and physical unit.

Firmware or flash modification can brick the device. Keep a verified full SPI-NOR backup, verify writes by independent readback/hash, and retain a tested FEL recovery path.

## Current status

The project has moved substantially beyond initial firmware-container and peripheral bring-up. The current locally confirmed baseline includes:

- stock update-container layout and checksum algorithm understood;
- stock EXE2 loaded at `0x40000000` and extensively mapped by static analysis;
- local SPI-NOR baseline confirmed as JEDEC `EF 40 15`;
- LCD and GT911 touch paths reconstructed and reproduced by bare-metal code in the companion Boot repository;
- ARM/R11-to-GOWIN FPGA 8-bit parallel control interface reconstructed and hardware-tested;
- FPGA diagnostic ID `0xA5A55A5A` reproduced without stock EXE2 execution;
- FPGA startup, external clock-generator, SMEC/job-service, horizontal, SysConfig, acquisition-control, and CSI paths substantially reconstructed;
- R11 CSI confirmed as the stock bulk waveform transport from the FPGA into ARM RAM;
- native raw acquisition layout confirmed at `0x42DF76D4`, with eight logical slots at `0x2134` stride;
- active raw payload confirmed as `0x1800` bytes in the tested fixed profile;
- standalone custom fixed-profile acquisition in the companion Boot repository reaches FPGA READY, CSI completion, and transfers the complete `0x1800`-byte payload without executing stock EXE2 acquisition code;
- stock full-acquisition and repeated runtime-sweep behavior reproduced under DPOSLAB, including runtime rearm/progress state;
- stock calibration persistence, serialization, selector, and calibrated `uint8_t` -> `uint16_t` conversion path reconstructed;
- DPOSLAB D0.4W13 HW-confirmed that loading persisted calibration restores all `8/8` active coefficient pairs and changes the unchanged stock calibrated CORE from `0/8` to `8/8` non-zero outputs.

The important current boundary is therefore:

```text
RAW TRANSPORT
GOWIN FPGA -> shared PE/CSI sample plane -> R11 CSI -> raw uint8 banks

CALIBRATION
persistent calibration -> calibration matrix -> selector -> per-lane coefficients

CONVERSION
raw uint8 + active coefficients -> stock calibrated CORE -> uint16 waveform data
```

Raw FPGA/CSI transport is no longer the primary unknown. Current work is focused on generalizing acquisition profiles and recovering stock-compatible sample calibration, physical voltage/time interpretation, lane/interleave semantics, and the remaining frontend/measurement behavior.

## Repository split

### This repository: FNIRSI-DPOS350P-Hack

Use this repository for:

- stock firmware reverse engineering;
- disassembly and analysis tooling;
- hardware/interface documentation;
- canonical recovered addresses and state fields;
- FPGA/CSI/SMEC/acquisition protocol reconstruction;
- DPOSLAB and DPOSDBG experiment evidence;
- historical milestones, corrections, and evidence levels.

The documentation intentionally distinguishes **HW-CONFIRMED**, **STATIC-CONFIRMED**, derived/inferred interpretations, and open questions.

### Companion repository: FNIRSI-DPOS350P-Boot

The companion Boot repository contains implementation work, including:

- custom Stage 1 / Stage 2 boot chain;
- bare-metal board and SoC support;
- display, touch, storage, FPGA, CSI, SMEC, profile, and acquisition drivers;
- standalone test applications and boot targets;
- custom fixed-profile acquisition code derived from the findings documented here.

Do not treat implementation status in the Boot repository and reverse-engineering evidence in this repository as interchangeable. New semantic claims should be proved here from stock/static or hardware evidence before becoming stable implementation assumptions.

## Confirmed hardware and interface summary

- **SoC:** Allwinner R11 / V3s-family ARM Cortex-A7.
- **DRAM:** 64 MiB DDR2 mapped from `0x40000000` on the analyzed unit.
- **FPGA:** GOWIN GW1N-9 family; exact package/revision is not assumed universal across all DPOS350P units.
- **Local SPI NOR baseline:** JEDEC `EF 40 15`; other DPOS350P revisions with different NOR vendors have been reported and are documented separately.
- **Managed storage:** SD-protocol internal managed NAND/eMMC-like device accessed through the R11 SD/MMC path.
- **Display:** 1024x600 RGB panel.
- **Touch:** Goodix GT911.
- **FPGA control plane:** shared PE GPIO parallel interface, using an 8-bit data bus plus control signals.
- **FPGA sample plane:** the same PE region switches to R11 CSI function 2 for bulk sample transport.
- **Clock generator:** MS5351M-compatible device on a separate bit-banged two-wire path, with stock 95 MHz and 10 MHz programming recovered.
- **External 8-pin device:** hardware evidence strongly points to an SMEC88SP-class secure/service IC candidate, but exact role and protocol boundaries remain documented with explicit evidence labels.

Hardware variants exist. Component vendor/package observations from one unit must not be generalized automatically to every DPOS350P.

## Key current acquisition facts

The confirmed stock/custom architecture is:

```text
CONTROL PLANE
ARM/R11 <-> GOWIN FPGA
PE GPIO parallel command/register interface

SAMPLE PLANE
GOWIN FPGA
    -> PE pins in CSI function 2
    -> R11 CSI @ 0x01CB4000
    -> raw acquisition RAM

STOCK RAW RING
base            0x42DF76D4
logical slots   8
slot stride     0x2134
active payload  0x1800 bytes in the tested profile

STOCK CONVERSION
raw uint8
    -> per-lane calibration
    -> stock conversion core around 0x4004073C
    -> uint16 waveform data
```

The companion Boot repository additionally has a HW-confirmed standalone fixed-profile capture into a caller-provided RAM buffer, so stock EXE2 is not required for the proven raw transport path.

## Documentation map

Start with the document matching the question being investigated:

- `docs/SYS_INFO.md` - canonical stock `SYS_INFO` field map plus related calibration/serializer globals.
- `docs/ACQUISITION.md` - authoritative acquisition, runtime-sweep, raw-ring, calibration, and converted-waveform model.
- `docs/FPGA_PROTOCOL.md` - ARM-visible FPGA command/selector protocol.
- `docs/FPGA.md` - canonical FPGA reverse-engineering record, including historical milestones and the current FPGA/CSI boundary.
- `docs/SMEC.md` - ARM-visible asynchronous FPGA job/service and SMEC-related reconstruction.
- `docs/STOCK_LIKE_MINIMAL_RUNTIME.md` - dependency-reduction experiments for stock-like execution.
- `docs/DPOSLAB.md` - controlled stock-vs-custom hardware experiments and milestone evidence.
- `docs/hardware.md` - board components, buses, revisions, and hardware evidence.
- `docs/display_init.md` - display and touch initialization findings.
- `docs/fel_mode.md` - FEL/recovery access notes.
- `docs/iso_format.md` - stock update-container format.
- `docs/checksum.md` - update-container checksum reconstruction.

When older statements in `docs/FPGA.md` conflict with later explicitly confirmed milestones, the later confirmed statement wins. The document preserves historical RE context deliberately.

## Analysis workflow

For the analyzed EXE2 image, prefer the companion vander-mello/dpos350p-analysis tools for first-pass static analysis over raw `objdump` output for first-pass function analysis:

```text
python -I analysis/disas.py <address> --func
python -I analysis/disas.py <address> <instruction-count>
python -I analysis/find_field_refs.py firmware/exe_load.bin 40544E68:<offset>
```

Use `objdump` as a secondary cross-check when needed. Keep firmware-build context attached to absolute addresses because other firmware builds can relocate otherwise equivalent functions and globals.

### Analysis tooling credit

The preferred static-analysis workflow uses the companion
[`vander-mello/dpos350p-analysis`](https://github.com/vander-mello/dpos350p-analysis)
toolset, including its Capstone-based `analysis/disas.py` and field-reference
helpers.

The analysis scripts are maintained by vander-mello and are not part of the
original FNIRSI-DPOS350P-Hack tooling. This repository uses them extensively
for function disassembly, fixed-range inspection, and cross-referencing the
recovered stock state layout. `arm-none-eabi-objdump` remains useful as an
independent secondary cross-check.

## Quick start: replace the stock boot logo
 
```text
python3 tools/dpos_tool.py replace original.iso your_logo.png custom.iso
python3 tools/dpos_tool.py verify custom.iso
```

Copy the resulting update container to the oscilloscope USB-storage root, safely eject the device, and reboot. Keep the original container and a verified flash backup before performing update experiments.

## Update-container tooling

The repository also retains tooling for analyzing and rebuilding the stock update container. Refer to `docs/iso_format.md` and `docs/checksum.md` before using it.

Flash/update experiments should always retain an independent recovery route and verify programmed contents after writes.

## Project principles

1. Separate **fact from interpretation**.
2. Prefer controlled A/B hardware experiments over assumptions.
3. Preserve raw addresses/offsets even when a symbolic name is introduced.
4. Do not promote a working semantic name to a confirmed one without evidence.
5. Treat hardware revisions and firmware builds explicitly.
6. Keep the Hack repository as the canonical RE/evidence record and the Boot repository as the implementation consumer of confirmed findings.
7. Preserve historical milestones, but clearly mark superseded conclusions so old experiments are not mistaken for current state.

## Licensing and firmware material

Original FNIRSI binaries are not distributed by this repository. Documentation and project-authored tools are provided under the repository's license. Users must supply any required firmware images from hardware or other lawful sources.
