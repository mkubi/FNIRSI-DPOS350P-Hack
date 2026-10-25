# Hardware

This document separates confirmed findings from open hypotheses.

## Confirmed components

### Allwinner R11 / V3s-compatible SoC

- ARM Cortex-A7, physically marked R11
- FEL ID `0x00168100`
- 64 MiB DDR2 at `0x40000000`
- `xfel ddr v3s` works on the tested unit
- UART0 PB8/PB9 is not known to be exposed on a header

### GOWIN FPGA

- GW1N-9 family, marked `GW1N-LV9MG256C5/I5`
- used for acquisition-related logic and generator-related functions
- exact configuration-storage path remains unconfirmed

### W25Q16JV SPI NOR

- 2 MiB, 3.3 V, SOP-8
- JEDEC `EF 40 15`
- connected to the Allwinner SPI0 path
- contains BT0/SPL, updater, logo, main ARM application and preserved device data

### CSNP1GCR01-B0W managed NAND

- SD-protocol managed NAND
- connected to **SDC0** through **PF0..PF5**
- tested unit contains a partition of type `0x06` beginning at LBA 63
- tested filesystem is FAT16
- used by the original USB/update environment and by the Custom App loader

### Display and touch

- 1024x600 RGB565 panel
- Goodix GT911 touch controller
- GT911: PG6 SDA, PG7 SCL, PE22 INT, PE24 RESET, address `0x5D`

## Unmarked 8-pin device near the FPGA / SMEC88SP candidate

The previous identification as a Winbond W25N512GV SPI NAND containing the FPGA bitstream is withdrawn. Current evidence instead makes a programmable secure IC in the Sinormous SMEC88 family, specifically an SMEC88SP-class SOP-8 device, the strongest identification candidate. The package has no readable marking, so this remains an evidence-based identification rather than a marking-based confirmation.

### Evidence from one measured unit

SOP-8 pin measurements reported by Marco Menti (`mmame`) on one unit:

```text
pin 1   floating / NC-like
pin 2   floating / NC-like
pin 3   steady about 3.0 V
pin 4   GND
pin 5   two-wire signal with 3.3 kOhm pull-up to VCC
pin 6   two-wire signal with 3.3 kOhm pull-up to VCC
pin 7   steady about 3.0 V
pin 8   about 3.1 V VCC
```

This closely matches the reported SMEC88SP SOP-8 assignment:

```text
1..3  NC
4     GND
5     SDA
6     SCL
7     RST / reset-related pin
8     VCC
```

Because the package marking is unreadable, keep this as **strong identification by pinout/electrical evidence**, not an absolute part-number confirmation.

### Two-wire bus observations

Logic-analyzer observations on the nearby four-pin header report:

```text
sample rate used successfully   50 MS/s
logic threshold                  1 V
SCL low                          about 160 ns
SCL high                         about 330 ns
clock                            about 2 MHz
startup fast phase               about 2.35 s
steady transaction cadence       about 2.79 ms
```

Framing is I2C-style. Captures decode an initial byte `0x00` for write-like phases and `0x01` for read-like phases. This resembles a 7-bit address zero plus the R/W bit, but the exact protocol and command layer remain unresolved. Sampling at 2 or 12 MS/s was reported too slow for reliable decoding. Vertical/horizontal scale-button changes did not add or delay the observed steady frames in the reported test.

### Firmware/runtime correlation

The stock power-on self-test contains an `encryption chip check` after the FPGA check and stops on failure. The literal string `SMEC` has not been identified in the stock firmware or SPI flash; the project name comes from reverse-engineering terminology and the `smec.v` module visible in developer material.

Local DPOSDBG runtime tracing confirms a repeating ARM-to-FPGA service protocol using selectors `0x64..0x6B`. ARM submits values through the FPGA, starts an operation, polls selector `0x65` until bit 0 is set, and reads returned values. DPOSDBG observes only the ARM-to-FPGA boundary; it does not prove the byte-level mapping of those selectors onto the physical FPGA-to-8-pin-device bus.

```text
ARM/R11
    |
    | PE GPIO parallel commands/status
    v
GOWIN FPGA
    |
    | probable separate two-wire master
    v
unmarked SOP-8 secure-device candidate
```

The FPGA-master statement remains a strong inference from the firmware/runtime architecture and external measurements, not direct electrical proof of which silicon drives each bus phase.

### What is now unlikely

- `EF 40 15` belongs to the main 2 MiB SPI NOR and is not evidence for this unmarked device.
- No current evidence supports the old claim that this device is SPI NAND storing the FPGA bitstream.
- Independent searches of NOR dumps and update firmware also failed to locate a GOWIN bitstream on the SoC side.

### Safety note

The secure-device vendor documentation reportedly describes tamper and program/data-protection behavior. Do not short, glitch or experimentally force the reset-related pin. Prefer passive logic-analyzer observation and ARM-to-FPGA DPOSDBG tracing.

### Still open

- definitive marking-based part identification
- exact FPGA-to-device packet/command encoding
- exact relation between ARM-visible `0x64..0x6B` transactions and physical bus frames
- what customer program/algorithm the device performs
- whether successful service transactions directly gate CSI acquisition

## Local Winbond-compatible NOR map

```text
0x000000..0x0067FF  eGON.BT0 / original or custom Stage 1
0x018000..0x03C7FF  original EXE1 / USB updater
0x058000             original logo block region
0x0A8000             original EXE2 header/application region
0x1DB000..0x1EAFFF  custom Stage 2 slot
0x1EB000..0x1EEFFF  calibration/configuration, preserve
0x1F0000..0x1FDFFF  metadata/version/configuration, preserve
```

Offsets not explicitly allocated must be preserved. All-`0xFF` data is not sufficient evidence that a region is generally free.

## Hardware variants and recovery notes

Known revisions must be treated separately. The local SPI baseline is `EF 40 15`; an external unit reports `GW1N-LV9EQ176C6/I5` plus Zbit `ZB25VQ16A` / manufacturer `0x5E`.

An external researcher reports a jumper-less FEL method on the Zbit revision: issue NOR deep-power-down `0xB9`, then watchdog-reset the SoC. BootROM reportedly sees no responding boot flash and enters FEL. BootROM reportedly does not send `0xAB`, so host-side flash access must wake the NOR before `spiflash-*` operations. A full power cycle restores normal flash response. This has not been reproduced on the local Winbond unit.

## Debug headers

### SPI/FEL header

The SPI header exposes the W25Q16 signals and reset. Shorting SPI CS to ground during power-on forces FEL on the tested unit. Keep a full backup before any write.

### FPGA JTAG header

A nearby 2x3 header is believed to expose GOWIN JTAG signals, but package-pin claims and tool compatibility should remain qualified until a complete trace and successful transaction are documented.

For the externally reported `GW1N-LV9EQ176C6/I5` revision, package-specific GOWIN data was reported to provide JTAG/configuration pin numbers. Those pin numbers must not be applied to the local FPGA until its exact package marking is re-checked.

### Four-pin header near FPGA

Not UART. Current captures show a roughly 2 MHz two-wire I2C-style bus during normal operation, with a deterministic startup phase and steady approximately 2.79 ms transaction cadence. See the unmarked-device / SMEC88SP-candidate section above.

## Original EXE2 startup flow

Confirmed call sequence around the application startup:

```text
sys_clock_init
sys_mmu_init
System_Init
AllInfoInit
spi_flash_init
GetSysSavedInfo
LanguageInit
FPGA_Start
GUI_Init
GT911_Init
Win_Init
UI_ColorInit
FatFsInit
SMEC_Init
PowerInit
HorizontalInit
VerticalInit
HardCostTimeInfoInit
Gen_Init
CheckAndUpdateNewSystem
FactoryCheck
SysConfigExec
```

Key confirmed addresses:

```text
main                    0x40034700
FPGA_Start              0x40049188
GUI_Init                0x400344E0
GT911_Init              0x40049A74
SMEC_Init               0x4004A15C
CheckAndUpdateNewSystem 0x4004F134
FactoryCheck            0x40051CF0
SysConfigExec           0x4003E708
```

The earlier address table shifted several names by one call and is superseded by this sequence.
