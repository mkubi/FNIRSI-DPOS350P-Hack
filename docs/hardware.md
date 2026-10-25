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

### Evidence 1: pinout

Measured by Marco Menti (`mmame`) on one unit, SOP-8, pin 1 at the package dot:

| Pin | SMEC88SP per vendor information | Measured |
|---|---|---|
| 1, 2 | NC (float, or tie to VCC/GND) | floating; about 40 mV and drifting when probed with a DMM while powered |
| 3 | NC | steady about 3.0 V while powered; diode mode 1.4 V / 0.52 V to GND |
| 4 | GND | GND; continuity to ground/shield |
| 5 | SDA | 3.3 kOhm pull-up to VCC; available on the nearby four-pin header |
| 6 | SCL | 3.3 kOhm pull-up to VCC; available on the nearby four-pin header |
| 7 | RST, active low; may float or go to VCC, must not go to GND | steady about 3.0 V while powered; diode mode 0.9 V / 0.52 V to GND |
| 8 | VCC, 1.62 V to 5.5 V | about 3.1 V |

The nearby four-pin header carries the two signal lines plus VCC and GND associated with this device. Pins 3 and 7 were reported tied high through resistor paths rather than directly to VCC or GND.

This closely matches the reported SMEC88SP SOP-8 assignment:

```text
1..3  NC
4     GND
5     SDA
6     SCL
7     RST / reset-related pin
8     VCC
```

Because the package marking itself is unreadable, this remains a strong identification by electrical/pinout evidence, not an absolute marking-based part-number confirmation.

### Evidence 2: firmware

The stock EXE2 power-on self-test contains the labels:

```text
FPGA检测 ...
加密芯片检测 ...
```

corresponding to an FPGA check followed by an "encryption chip check". Additional ADC, channel-multiplexing, range, DC/AC, bandwidth-limit and flatness checks follow. The reported behavior is that failure of the encryption-chip check is displayed and the firmware stops.

The literal string `SMEC` has not been identified in the stock firmware or SPI flash. The historical project terminology `SMEC_Init` comes from reverse-engineering labelling and the `smec.v` FPGA project filename visible in developer material.

The CPU's independently identified software two-wire buses serve the GT911 and MS5351M. Secure-device-related ARM code instead accesses the subsystem through the FPGA parallel-register interface.

Together with the runtime evidence below, this strongly supports an FPGA-mediated link to the external secure-device candidate.

### Evidence 3: SMEC88 vendor information

The Sinormous SMEC88 product information describes the SMEC88SP/SMEC88ST family as customer-programmable secure devices.

Reported properties include:

- SOP-8 SMEC88SP and SOT23-6 SMEC88ST packages
- two-wire/I2C interface
- hardware support up to approximately 3.4 Mbit/s
- customer-defined program downloaded into the device
- command/result exchanges with the host
- program/data protection and memory-encryption features
- tamper-related protection mechanisms
- a reset pin which may float or be pulled to VCC but must not be tied to GND

The measured electrical pinout above closely matches the reported SMEC88SP SOP-8 pinout.

Because the package marking itself cannot be read, the identification remains evidence-based rather than marking-confirmed.

### Two-wire bus observations
 
DSLogic Plus captures on the nearby four-pin header report:
 
```text
sample rate used successfully    50 MS/s
logic threshold                  1 V
SCL low about                    160 ns
SCL high                         about 330 ns
clock                            about 2 MHz
startup fast phase               about 2.35 s
steady transaction cadence       about 2.79 ms
steady pair rate                 about 358 pairs/s
reported payload                 7 data bytes/frame
```

Observed characteristics:

- both signal lines idle high
- SDA changes while SCL is low
- I2C-style START and byte/ACK framing is visible
- frames begin with 0x00 for write-like phases or 0x01 for read-like phases
- this is consistent with a 7-bit address value of zero plus the R/W bit, although the exact command-layer semantics remain unresolved
- after power-up there is a deterministic fast phase lasting approximately 2.35 seconds
- steady operation then produces write/read pairs approximately every 2.79 ms
- the timing was reported identical across multiple power-ups
- frame data changes between individual exchanges and differs between power-ups
- vertical and horizontal scale-button changes did not add or delay observed frames
- 2 MS/s and 12 MS/s were insufficient for reliable decoding
- 20 MS/s resolved clock edges, while 50 MS/s provided comfortable timing resolution
- the relatively slow edges are consistent with the measured 3.3 kOhm pull-ups

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

The secure-device vendor information describes tamper and program/data-protection behavior.

- probe the two-wire signal pins passively
- do not tie the reset-related pin to GND
- do not deliberately short pins
- do not apply abnormal voltage, clock or glitch conditions
- prefer passive logic-analyzer capture and ARM-to-FPGA DPOSDBG tracing

The device may contain customer-specific program material which cannot be reconstructed from public vendor documentation.

### Still open

- definitive marking-based part identification
- exact physical packet/command encoding on the FPGA-to-device two-wire bus
- exact relationship between ARM-visible `0x64..0x6B` transactions and physical two-wire frames
- what the customer-defined program inside the secure device computes
- how the changing transaction inputs and returned values are generated and consumed
- whether successful service transactions are a hard gate for CSI acquisition or indirectly related
- whether every DPOS350P hardware revision uses the same secure device

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

Not UART. The header carries the two signal lines plus VCC/GND associated with the unmarked 8-pin SMEC88SP candidate. Captures show I2C-style activity at about 2 MHz, with a deterministic startup phase and a steady approximately 2.79 ms transaction cadence during normal operation. See the unmarked-device section above.

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
