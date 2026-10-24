# FNIRSI DPOS350P Firmware Reverse Engineering

Reverse-engineering notes, tools and documentation for the FNIRSI DPOS350P 350 MHz / 1 GSa/s oscilloscope.

## Disclaimer

This is experimental work on one unit. Hardware revisions and firmware versions may differ. Firmware or flash modifications can brick the device. Keep a verified full SPI NOR backup and a tested FEL recovery path.

## Current status

- original update container and checksum algorithm understood
- custom boot logo update confirmed
- W25Q16JV SPI NOR dumped and mapped
- working bare-metal 1024x600 display and GT911 touch support
- autonomous custom two-stage boot available in the companion `FNIRSI-DPOS350P-Boot` repository
- internal managed NAND read through SDC0 and FAT filesystem access confirmed
- MCU-to-FPGA parallel interface partly reconstructed and experimentally verified

## Confirmed hardware summary

- **SoC:** Allwinner R11 / V3s-compatible ARM Cortex-A7, FEL ID `0x00168100`
- **DRAM:** 64 MiB DDR2 at `0x40000000`
- **FPGA:** GOWIN GW1N-9 family
- **SPI NOR:** Winbond W25Q16JV, 2 MiB, JEDEC `EF 40 15`
- **managed NAND:** CSNP1GCR01-B0W, SD protocol, connected through SDC0/PF0..PF5
- **display:** 1024x600 RGB565
- **touch:** Goodix GT911
- **unmarked 8-pin device near FPGA:** very likely a Sinormous SMEC88SP secure ("encryption") IC on I2C, identified by pinout, firmware self-test label and vendor datasheet rather than by chip marking; see [docs/hardware.md](docs/hardware.md)

The former W25N512GV identification was based on interpreting `EF 40 15` as SPI-NAND output. Hardware and firmware tests show that this is the W25Q16JV NOR ID. The unmarked device is therefore not SPI NAND; the evidence for its actual identity, and what is still open, is collected in `docs/hardware.md`.

## Quick start: custom boot logo

```fish
python3 tools/dpos_tool.py replace original.iso your_logo.png custom.iso
python3 tools/dpos_tool.py verify custom.iso
```

Switch the oscilloscope to USB storage mode, copy the resulting `.iso` container to the root, safely eject and reboot. The updater scans for an update container; preserve the original file naming convention when in doubt.

## Documentation

- `docs/hardware.md`: confirmed hardware, buses, flash map and startup flow
- `docs/display_init.md`: working bare-metal display and touch pipeline
- `docs/fel_mode.md`: FEL access and verified XFEL commands
- `docs/iso_format.md`: update-container layout
- `docs/checksum.md`: checksum algorithm and reference implementation
- `docs/FPGA.md`: confirmed FPGA interface findings and open hypotheses

Original FNIRSI binaries are not included. Documentation and tools are MIT licensed.
