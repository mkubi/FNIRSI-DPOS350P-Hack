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

## Analog and mixed-signal parts (community reports and firmware view)

The parts below come from photos and measurements posted in the EEVblog thread "New oscilloscope from FNIRSI DPOS350P". They were not re-checked on the local unit. Each item names the reply and author so it can be traced. Permalinks use the form `https://www.eevblog.com/forum/testgear/new-oscilloscope-from-fnirsi-dpos350p/msgNNNNNNN/#msgNNNNNNN`.

Some items add what static analysis of the stock firmware (build 2025-06-19) shows about how the part is driven. Those items say so and were not checked on hardware.

Status labels: **marking read** (part number read from the package), **measured**, **likely** (identification from layout or datasheet match, not from a full marking), **firmware** (read in the stock firmware; "probably" marks an inference).

### ADCs

- four dual-channel ADC packages
- likely MXT2088; the visible marking includes `2445` (pcprogrammer, reply #185, msg5925644)
- the MXT2088 datasheet shows a dual-input part laid out like the older AD9288 (mawyatt, reply #188, msg5925778)
- with one channel enabled, all four ADCs appear to receive the signal (wolfy007, reply #183, msg5925540); **firmware:** in single-channel mode the stock code reads all eight raw sample banks for that channel, in dual mode four banks per channel
- **firmware:** each of the eight banks holds consecutive samples of one converter lane, one byte per sample. The code calibrates gain and offset per lane, keeps a sampling-phase setting per lane, and rebuilds the waveform by interleaving the lanes in an order selected by channel mode and sample rate (details in `docs/FPGA.md`). This fits mawyatt's suggestion of two chips per input with eight clock phases (reply #188)
- a 125 MHz ADC clock on ADC pins 14 and 47 was requested for measurement to check the interleave phases; no result was posted (pcprogrammer, reply #184, msg5925636)
- in 250 MHz captures, pcprogrammer reads a slight phase difference between ADC1 and ADC4 (reply #184, from the captures in wolfy007's reply #181)

### ADC reference

Measured by several users:

- the reference reaches the ADCs through analog switches from two sources (Martin Brubaker, reply #456, msg5992415)
- the stock divider gives about 1.237 V on MXT2088 pins 5 and 7; the datasheet value is 1.18 V (reply #456; reply #596, msg6037511)
- on the reporting units this made readings about 7% low against a ±2% specification (Martin Brubaker, reply #436, msg5991319)
- the divider uses 3.3 kOhm 0603 resistors (replies #462 and #481)
- reported fixes: 33 kOhm soldered across the ground-side 3.3 kOhm resistor gave 1.178 V (Martin Brubaker, replies #462 and #596); BillyO replaced that resistor with 3 kOhm and reported a clear improvement (reply #463); on onefabis's unit 3 kOhm read low and 3.3 kOhm with 51 kOhm in parallel worked better (reply #531, msg6018833)
- after the change, offsets drifted until autocalibration was run again (reply #456)

Changing this divider changes every vertical reading. Re-run autocalibration afterwards.

### Clock synthesizer

- marking read: `MS5351M` (pcprogrammer, reply #185; wolfy007, reply #186, msg5925672)
- pcprogrammer expects two clocks to the FPGA: a fixed sampling clock and a variable generator clock (reply #185). **firmware:** confirmed. The stock code programs two outputs, 95 MHz (100 MHz for the 1 GSa/s setting) and 10 MHz; the second output is then re-programmed by the signal generator, so it is probably the generator's sample clock
- the generator output frequency was measured about 0.02% high on two units (+0.0192% PA2ER, reply #705, msg6118535; +0.0208% BillyO, replies #707 and #722, msg6118977), which points to the reference clock; a third unit was reported about 0.03% off at 10 MHz (KiethWallace, reply #795, msg6163483)
- **firmware:** the clock driver uses a fixed 25 MHz reference constant and does not set the chip's crystal load capacitance. Scope and generator clocks come from the same chip, so one reference error would shift both by the same fraction. A load-capacitance setting or a per-unit reference correction may remove most of the offset (not tested)

### Generator DAC

- marking read: `MS9714`, 14-bit, 125 MSa/s (pcprogrammer, reply #185)
- at low generator amplitude, ripple becomes more visible when the DAC package is touched (Ivont, reply #783, msg6132674)

### Offset generation

- the channel offsets and the generator DC offset come from FPGA PWM outputs filtered to DC, not from the DAC (Martin Brubaker, reply #436)
- the ICs to the right of the FPGA do this PWM-to-DC conversion; the function of a fourth IC there is unknown (Martin Brubaker, reply #561, msg6025809)
- channel offsets are calibrated by autocalibration; the generator offset is not (reply #456)
- **firmware:** each channel offset is one 16-bit value sent to the FPGA. Autocalibration searches that value until the mean ADC code reaches two target bands and derives the offset step per code from the two points

### Input stage

- there is no 50 Ohm input option; reflections from the BNC add to the signal, and the front-end gain is not flat (yangyao, reply #211, msg5938958)
- 270 Ohm resistors were added as a production rework, stacked on existing parts to the right of the FPGA (ststefanov13, reply #553, msg6023261; Martin Brubaker, reply #554, msg6023313); Martin Brubaker guesses this was done to speed up the front ends (reply #561)
- channel-to-channel skew is a little under 0.7 ns (BillyO, reply #327, msg5948070)

### Front-end control (firmware)

All front-end settings go from the SoC to the FPGA as bus commands, one per channel and setting:

- **V/div (attenuation):** sent only when the range changes; the code then re-sends the offset and waits at least 60 ms before continuing. A wait this long fits mechanical attenuator relays settling (not confirmed on hardware)
- **bandwidth limit:** three states, full, 150 MHz and 20 MHz
- **coupling:** a two-state switch, probably AC/DC
- **offset:** the 16-bit value above
- **calibration channel select:** set only during system calibration (CH1, then CH2), followed by a 50 ms wait; zero in normal use

The SoC drives no front-end pin directly in this path. So the relay, switch and offset lines are probably FPGA outputs, and a pin hunt can start at the FPGA I/O that change when V/div, coupling or bandwidth change.

### Display connection (firmware)

- the panel is a parallel RGB panel driven by the SoC's LCD timing controller TCON0 (`0x01C0C000`), with the display engine (`0x01000000`) composing the frame; no panel command interface (SPI or I2C) is used at start-up
- 20 pins of port D, PD1..PD21 except PD12, are set to function 2. Port D is not listed in the V3s manual. Twenty lines fit 16 colour bits (RGB565) plus clock, data enable, HSYNC and VSYNC, but the role of each pin is not confirmed
- the stock code programs an active area of 1022 x 600, two columns less than the documented 1024; whether two columns are unused is not known
- dot clock: video PLL 24 MHz x 80 / 4 = 480 MHz, divided by 7 in TCON0, so about 68.6 MHz
- backlight control is not part of this path

### Still open

- full ADC marking and confirmation of the MXT2088 identification
- ADC clock frequency and phase per converter, and which MS5351M output feeds the ADCs
- attenuator topology, relay, analog-switch and amplifier part numbers; stepping V/div while listening for a click or probing coil drivers would identify the relays
- which FPGA pins drive the ADCs, the DAC, the PWM offset outputs and the front-end relays and switches
- the role of each display pin on port D

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
