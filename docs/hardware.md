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

## Unmarked 8-pin device near the FPGA

**Status: identified with moderate confidence as a Sinormous SMEC88SP, a customer-programmable secure ("encryption") IC with an I2C interface.** The device carries no readable marking, so this rests on three independent lines of evidence below. It is not SPI NAND and it does not store the FPGA bitstream.

History: the device was previously documented as a Winbond W25N512GV SPI NAND containing the FPGA bitstream. That claim was withdrawn:

- `EF 40 15` is the confirmed JEDEC ID of the 2 MiB W25Q16JV NOR
- `xfel spinand` returning those bytes does not prove a second SPI-NAND device

### Evidence 1: pinout (measured on one unit, SOP-8, pin 1 at the package dot)

| Pin | SMEC88SP per vendor page | Measured |
|---|---|---|
| 1, 2 | NC (float, or tie to VCC/GND) | floating; about 40 mV and drifting when probed with a DMM (powered) |
| 3 | NC | steady about 3.0 V (powered); diode mode 1.4 V / 0.52 V to GND |
| 4 | GND | GND (continuity to ground/shield) |
| 5 | SDA | 3.3 kOhm pull-up to VCC; available on a nearby 4-pin header |
| 6 | SCL | 3.3 kOhm pull-up to VCC; available on the same header |
| 7 | RST, active low; may float or go to VCC, must not go to GND | steady about 3.0 V (powered); diode mode 0.9 V / 0.52 V to GND |
| 8 | VCC, 1.62 V to 5.5 V | about 3.1 V |

The nearby 4-pin header carries SDA, SCL, VCC and GND of this device (it is not UART). Pins 3 and 7 are tied high through resistor paths and are not tied directly to VCC or GND.

### Evidence 2: firmware

The stock EXE2 runs a power-on self-test screen. Its labels (UTF-8 strings) include `FPGA检测 ...` (FPGA check) followed by `加密芯片检测 ...` ("encryption chip check"), then ADC, channel multiplexing, range, DC/AC, bandwidth-limit and flatness checks. If the encryption-chip check fails, the screen shows a failure and the firmware stops. The string "SMEC" itself does not appear anywhere in the firmware image or the SPI flash; the name `SMEC_Init` comes from this project's labelling and from the FPGA project filename `smec.v` seen in developer screenshots. The FPGA, not the CPU, drives the two-wire link: no CPU GPIO routine for those lines was found (the CPU's own two-wire buses go to the GT911 and to the MS5351M clock chip), and the firmware reaches the device only through FPGA registers.

### Evidence 3: vendor datasheet page

The vendor page (Shenzhen Zhongju Weiye, <http://www.sinormous.com/product-item-19.html>, "SMEC88 (SMEC88SP/SMEC88ST)") describes: 32-bit smart-card core, SOP-8 (SMEC88SP) and SOT23-6 (SMEC88ST) packages, standard I2C with hardware support up to 3.4 Mbit/s, customer-defined program downloaded into the chip, MCU sends a command and receives a result from that program, memory encryption, tamper detection, and an RST pin that may float or go to VCC but not GND. The pinout above matches this description. Note: the site is served over HTTP and its TLS certificate does not match the hostname (checked 2026-10-24), so verify the page independently.

### Bus observations (DSLogic Plus, 50 MHz, threshold 1 V, SDA/SCL on the 4-pin header)

- both lines idle high; SCL is a regular clock of about 160 ns low and 330 ns high (about 2 MHz), which is inside the chip's rated 3.4 Mbit/s; SDA changes while SCL is low
- I2C-style framing: START, 8 data bits plus an acknowledge bit per byte
- every frame starts with byte `00` (write) or `01` (read), consistent with a 7-bit slave address of 0x00
- after power-up there is a fixed fast phase of 2.35 s with short frames, then a steady state of write/read pairs every 2.79 ms (about 358 pairs per second), 7 data bytes per frame
- the timing is identical across power-ups; the data bytes change on every exchange and differ between power-ups
- button presses (vertical and horizontal scale) did not add or delay any frames
- sampling at 2 MS/s or 12 MS/s is too slow for this clock; 20 MS/s decodes clock edges, 50 MS/s is comfortable. Edges are slow because of the 3.3 kOhm pull-ups, so a low logic threshold gives cleaner timing

### Still unconfirmed

- the chip marking itself (identification is by evidence, not by reading the package)
- what the chip's program computes and how its results are used downstream; per the vendor description that program is the product vendor's own and is not publicly documented
- whether the FPGA depends on the chip once running
- that other units or hardware revisions use the same part

### Cautions

- probe pins 5 and 6 passively only; never tie pin 7 (RST) to GND
- do not short pins or apply abnormal voltage, clock or glitches to the device: the vendor states that tamper detection destroys the internal program, which cannot be restored

The GW1N family is non-volatile, so the FPGA configuration is most likely stored inside the FPGA itself.

## W25Q16JV map

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

## Debug headers

### SPI/FEL header

The SPI header exposes the W25Q16 signals and reset. Shorting SPI CS to ground during power-on forces FEL on the tested unit. Keep a full backup before any write.

### FPGA JTAG header

A nearby 2x3 header is believed to expose GOWIN JTAG signals, but package-pin claims and tool compatibility should remain qualified until a complete trace and successful transaction are documented.

### Four-pin header near FPGA

Not UART. It carries SDA, SCL, VCC and GND of the unmarked 8-pin device (very likely a SMEC88SP secure IC) and shows I2C-style activity at about 2 MHz during normal operation. See the unmarked-device section above.

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
