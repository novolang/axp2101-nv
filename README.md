# axp2101-nv

A driver for the AXP2101, a single-cell power-management chip from X-Powers
(Shenzhen X-Powers Technology) with a linear charger, five DC-DC converters,
nine LDOs and a battery gauge, specified in the data sheet "AXP2101 Single
Cell NVDC PMU with E-gauge", version 1.4, 13 October 2022
([third-party mirror](https://whycan.com/files/members/7106/AXP2101_Datasheet_V1.4.pdf);
X-Powers publishes no copy that could be fetched). The driver speaks I2C and
implements the `PowerManager` and `Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv). The M5Stack CoreS3,
Core2 v1.1 and the LilyGO T-Watch S3 carry the part.

**Hardware verification: NOT VERIFIED.** No board on the bench carries this
chip, so the driver has run only against a double of the part, in the host
suite. The device program below builds for the nRF52840-DK, the
nRF54LM20-DK and the FRDM-MCXN947 and has not been flashed. The BSP of this
repository has no ESP32-S3 target, which the CoreS3 needs, so the program is
built for the Nordic and NXP boards and the CoreS3 is not a target.

## The device program

`examples/power_monitor.nv` reads an AXP2101 once a second and prints the
cell voltage, the gauge's percentage, the charging phase, the supply voltage
and the die temperature over the board's UART, and reports a press of the
power key:

```
bat 3874 mV 87 %  charge cc  vbus 5012 mV  die 31250 mC
key short
```

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use dev_power.{ PowerManager }
use axp2101
use axp2101.{ Axp2101 }

@tier(embedded)
fn main() [hw]
    // The part answers at 0x34; its IRQ output is on pin 29.
    let w = dev_bus.wiring_irq(dev_bus.wiring_i2c(0, 0x34), 29)
    let pmu = Axp2101.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, w)
    hal.timer.delay_ms(10)
    if pmu.begin() != 0
        hal.uart.write("axp2101: no answer\n")
        return
    loop
        match pmu.battery()
            Some(b) =>
                // b.voltage.mv is the cell voltage in millivolts and
                // b.percent the gauge's whole percent.
                hal.uart.write(str.from_int(b.voltage.mv))
                hal.uart.write(" mV\n")
            None => hal.uart.write("no battery\n")
        hal.timer.delay_ms(1000)
```

The full program, with the supply voltage, the die temperature and the key,
is `examples/power_monitor.nv`. A board package that depends on this package
builds it as its `main`. `tests/device_build.sh` does so from a scratch
package for the boards below and for the emulated board, where it runs the
image and checks that the program reports that the part does not answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which the program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 21,160 B | 5,636 B | 3,252 B |
| `nrf54lm20-dk` | 22,120 B | 5,788 B | 3,236 B |
| `frdm-mcxn947` | 23,376 B | 7,560 B | 2,748 B |

The numbers are from the build's memory report. Flash also holds the runtime,
the compiler's support library and the program's tables. The arena, 65,536 B
on the Nordic boards, is the runtime's heap and is reserved whether or not the
program uses it. The Nordic DKs drive I2C with a TWIM (the nRF52840 DK's TWIM0 on SDA P0.26 and
SCL P0.27, the nRF54LM20 DK's TWIM22 on SDA P1.03 and SCL P1.04), on the pins
`docs/hal/README.md` names under "The Nordic DKs' buses", so all three rows
are the image with the driver's register paths in it. Before the Nordic buses
landed the two DK rows were 13,328 and 14,156 bytes of flash, of which the
program was 1,568 and 1,602 bytes: the board packages answered for I2C as
absent hardware, and the build removed the code that follows the first bus
read. A reply from the part over a DK's bus is NOT VERIFIED until the driver
rig carries one.
The driver's own code allocates nothing: every buffer is
a fixed-size `Vec` in the caller's frame.

### The effects

Every function of the driver that reaches the bus or the pin declares `[hw]`,
and nothing else. The driver performs no input or output of its own and
allocates nothing of its own. The bus functions of devices-nv build one short
frame per transaction inside a `system` block and release it when the call
returns. The module checks at every tier, `embedded` included. The program
above declares `[hw]` for the same reason.

## What it is

The AXP2101 turns a USB supply (VBUS) and a single lithium cell into the
rails of a processor board. Its registers are bytes at the addresses of the
data sheet's section 6.13, read and written in sequence over I2C at the 7-bit
address 0x34 (the data sheet gives 0x68 and 0x69, the 8-bit forms).

| Group | What the driver reaches |
|---|---|
| Rails | DCDC1 to DCDC5, ALDO1 to ALDO4, BLDO1, BLDO2, CPUSLDO, DLDO1, DLDO2: an enable bit and a voltage register each |
| Charger | the cell charge enable, the pre-charge, constant and termination currents, the voltage limit, the button-cell charger, the phase |
| Gauge | the percentage, the enable, the reset, the two low-battery levels |
| ADC | the battery, VBUS and system voltages, the TS pin and the die temperature, 14 bits each |
| Interrupts | 23 named sources in three enable and three status registers |
| Power key | the long-press time, the power-off time, the power-on time, the power-off action |
| Commands | power-off and restart |

| Rail | Number | Range | Step |
|---|---|---|---|
| DCDC1 | 0 | 1500 to 3400 mV | 100 mV |
| DCDC2, DCDC3 | 1, 2 | 500 to 1200 mV, then 1220 to 1540 mV | 10 mV, then 20 mV |
| DCDC4 | 3 | 500 to 1200 mV, then 1220 to 1840 mV | 10 mV, then 20 mV |
| DCDC5 | 4 | 1200 mV, and 1400 to 3700 mV | 100 mV |
| ALDO1 to ALDO4, BLDO1, BLDO2 | 5 to 10 | 500 to 3500 mV | 100 mV |
| CPUSLDO | 11 | 500 to 1400 mV | 50 mV |
| DLDO1 | 12 | 500 to 3300 mV | 100 mV |
| DLDO2 | 13 | 500 to 1400 mV | 50 mV |

## Install

```sh
novo pkg add axp2101-nv
```

## What the package contains

| Module | What is in it |
|---|---|
| `axp2101` | The voltage tables, the register access over the trait `AxpLink`, the identification and start, the status, the ADC, the rails, the gauge, the charger, the TS pin and the watchdog, the interrupts, the power key and the power-off command, the bus and pin framing, the types `Axp2101` and `Axp2101Link`, and the `PowerManager` and `Identifiable` implementations |

## How to choose an entry point

A program on a board uses `Axp2101`, as the board's device table builds it:

```novo norun:fragment
    devices
        pmu  Axp2101  from "axp2101-nv"  on i2c0  addr 0x34  irq pmu_irq
```

A program that holds its own bus, or a test with a double of the part,
implements `AxpLink` over its bus and calls the module's functions, which take
the link: `axp2101.rail_set_mv(link, rail, mv)`, `axp2101.irq_service(link)`.
The framing of the bus is `i2c_read`, `i2c_write` and `link_ready`, generic
over `I2cBus` and `GpioOut`.

| Method of `Axp2101` | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr` and, if the wiring has one, the IRQ pin `w.irq`; nothing is sent |
| `begin()` | checks the identification and switches on the ADC channels; 0, -1 (no part) or -2 |
| `rail_voltage`, `rail_set_voltage`, `rail_enable`, `rail_state`, `rail_count` | `PowerManager`: the 14 rails |
| `battery()`, `power_source()` | `PowerManager`: the cell, and USB, battery or none |
| `device_id()` | `Identifiable`: the identification register, 0x4A |
| `percent()`, `battery_mv()`, `vbus_millivolts()`, `vsys_millivolts()`, `die_temp()` | the gauge and the ADC |
| `charge()`, `set_charge_current(ma)`, `set_charge_voltage(mv)`, `set_charging(on)` | the charger |
| `set_rail_mv(rail, mv)` | a rail's voltage, answering the voltage set or a negative code |
| `service()`, `enable_irq(mask, on)`, `irq_pin()` | read and clear the interrupt flags; the sources that raise the pin; the pin |
| `shut_down()`, `reboot()` | the power-off and restart commands |

The module's functions add `status_read`, `adc_read`, `battery_warning_set`,
`gauge_enable` and `gauge_reset`, `precharge_current_set`, `termination_set`,
`button_charge_enable` and `button_charge_voltage_set`, `ts_pin_set`,
`watchdog_start`, `watchdog_feed` and `watchdog_stop`, `irq_status` and
`irq_clear`, and `key_long_press_set`, `key_off_time_set`, `key_on_time_set`
and `key_power_off_set`. The tables are `voltage_code` and `voltage_of_code`
for the rails, `icc_code_of_ma` and `icc_ma_of_code` for the charge current,
and `cv_code_of_mv` and `cv_mv_of_code` for the charge voltage.

## The rules a user needs

1. The identification register, 0x03, is not in the data sheet. Zephyr's
   `mfd_axp2101` and XPowersLib both read it and expect 0x4A, and so does the
   driver. `begin` answers -1 for any other value.
2. `rail_set_voltage` sets a rail to the highest step at or below the
   request, never above it, and answers the voltage it set. A request above a
   rail's top is held to the top; a request below its lowest voltage is
   refused with nothing written (`None`). Only the voltage field of the
   register is written: DCDC2 and DCDC3 keep their dynamic-voltage enable in
   bit 7 and DCDC5 its compensation bit in bit 5.
3. DCDC2 to DCDC4 have two ranges. A request between 1200 and 1220 mV is 1200
   mV. DCDC5 has 1200 mV as a code of its own, between its 1400 mV range and
   the reserved codes.
4. The data sheet's table for CPUSLDO and DLDO2 ends at "10011: 1.40V" (code
   19), while its range, 0.5 to 1.4 V in 50 mV steps, ends at code 18. The
   driver takes the range, as XPowersLib does, so code 19 reads as no voltage.
   For DLDO1 the data sheet's features say 0.5 to 3.3 V and its table ends at
   "11100: 3.3V" with 11101 reserved, while its range line says 0.5 to 3.4 V;
   XPowersLib accepts 3400 mV. The driver follows the table, so the top of
   DLDO1 is 3300 mV.
5. Switching a rail is a read-modify-write of its enable register, which holds
   other rails' bits. The rails the part's fuses switch on at power-up are the
   board's; the driver does not know which.
6. `battery()` answers `None` with no battery present (STATUS1 bit 3). The
   gauge's percentage reads 0 with no battery, which would pass for an empty
   one. The current is always 0: the part reports the direction of the battery
   current (STATUS2) and not its size. The phase is STATUS2's charging status;
   the trickle charge is reported as `ChargePrecharge`.
7. An ADC value is 14 bits, the high six in the first register and the low
   eight in the next, read in one transaction. The top two bits of the first
   register of VBAT and TS are not data, so they are masked off. VBAT, VBUS
   and VSYS count 1 mV a step, TS 0.5 mV.
8. The die temperature is `22 + (7274 - raw) / 20` degrees, from XPowersLib;
   the data sheet gives the channel's step and no conversion.
9. The interrupt status bits clear when 1 is written to them. `irq_clear`
   writes the bits of its mask and `irq_service` the bits it just read, in one
   transaction each; a flag that rose after the read stays. A
   read-modify-write would write back every set bit and clear a flag the
   program has not seen.
10. The 23 sources are one 24-bit mask, register 0x48 or 0x40 in bits 0 to 7,
    0x49 or 0x41 in bits 8 to 15 and 0x4A or 0x42 in bits 16 to 23. The two
    state-of-charge levels are named for their status bits: warning in bit 7
    and shutdown in bit 6. The data sheet calls their enable bits level 2 and
    level 1.
11. The charge voltage limit is one of 4000, 4100, 4200, 4350 and 4400 mV. The
    register's code 000 is 5.0 V, a limit for a supply, and the driver does not
    write it.
12. The charge current steps are 25 mA up to 200 mA (the steps below 100 mA
    are reserved) and 100 mA from there to 1000 mA. `charge_current_set` takes
    the highest step at or below the request and refuses a request from 1 to
    99 mA rather than turn charging off. The register's power-up value is the
    board's, set in the part's fuses.
13. With the TS pin set as the battery thermistor input, a battery with no
    thermistor stops the charger. `ts_pin_set(false)` makes it an external
    fixed input; `ts_pin_set(true)` makes it the thermistor input at 50 microamps
    for a 10 kilohm thermistor, and switches the TS ADC channel on.
14. The watchdog timeout is a power of two of seconds, 1 to 128; `watchdog_start`
    takes the shortest not below the request. The watchdog must be fed
    (`watchdog_feed`) within the timeout.
15. A long press of the power key reports after 1, 1.5, 2 or 2.5 s; the
    key switches the system off after 4, 6, 8 or 10 s when enabled
    (`key_power_off_set`); the system switches on after the key is held for
    128 ms, 512 ms, 1 s or 2 s. The setters take the shortest time not below
    the request.
16. `power_off` and `restart` set bits of the common configuration register
    that the part clears itself. The part stops answering as it powers off,
    so the acknowledge of the write is all the driver sees. A board that is
    powered through the part switches itself off.

## Running on a microcontroller

Every function of the module builds at the embedded tier. The driver's own code
builds no list: its bytes move through devices-nv's `DevBytes`, a 32-byte buffer
held in the caller's frame. The registry shows the package on the host layer,
the layer of the trait packages it depends on.

## What is not included

- The RTCLDO1 and RTCLDO2 backup supplies: they have no register and are not
  rails. NOT IMPLEMENTED.
- A rail fault: the part has one over-current flag for all LDOs and no flag
  per rail, so `rail_state` never answers `RailFault`.
- A battery current: the part measures none.
- The JEITA temperature profile (registers 0x54 to 0x5D), the TS hysteresis
  settings, the charge safety timers (0x67), the thermal regulation threshold
  (0x65), the input voltage and current limits (0x15, 0x16) and the minimum
  system voltage (0x14). The registers can be written with `reg_write`.
  NOT IMPLEMENTED.
- The fast power-on sequence, the sleep and wake-up control (0x26), the
  power-on sources and power-off sources status (0x20, 0x21), the
  power-off thresholds (0x23, 0x24), the DCDC mode and frequency spread
  (0x80, 0x81), the CHGLED pin control (0x69) and the four data buffer
  registers (0x04 to 0x07). NOT IMPLEMENTED.
- The battery model of the gauge: the battery parameter register (0xA1) and
  the gauge control (0xA2). The gauge runs its default model. NOT IMPLEMENTED.
- The RSB serial interface; the driver speaks I2C.
- The async form of `PowerManager`.

## Reference drivers read

Each of these was read for its fixes, and none was ported. The fix is named at
its site in `src/axp2101.nv`.

| Driver | What was taken from it |
|---|---|
| Zephyr `drivers/mfd/mfd_axp2101.c` | the identification register 0x03 and its value 0x4A; the interrupt flags cleared by writing back the bits read |
| Zephyr `drivers/charger/charger_axp2101.c` | the charge-current table, the voltage-limit table with its 5.0 V entry, the pre-charge and termination steps of 25 mA, the TS pin disable |
| XPowersLib `XPowersAXP2101` | the battery presence check before the percentage and the voltage; the die temperature formula; the TS pin set as an external input at start; the power-off and the power key bits; CPUSLDO and DLDO2 ranges to 1400 mV |
| `axp2101` crate 0.1.0 | the register names; its bit fields for the interrupt enables lay the bits from the opposite end to the data sheet (it puts the battery under-temperature source of register 0x40 at bit 7 and the data sheet at bit 0), so the data sheet's bit order is used here |

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  `PowerManager` and `Identifiable` traits, the wiring record a board's device
  table builds, and the register access this driver uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names the
  `I2cBus` and `GpioOut` traits and the board handles `BoardI2c` and
  `BoardGpio`.
- [max17048-nv](https://novo-lang.org/packages/max17048-nv) implements the
  battery side of `PowerManager` for a fuel gauge with no rails.

## Tests

`novo test tests` runs the module over a double of the part that answers on an
I2C bus. The double keeps a register file read and written in sequence, clears
the interrupt status bits that are written as 1, clears the command bits that
clear themselves and counts them, keeps the ADC data registers read-only, and
records every transaction. It can refuse the n-th read or write, a register, or
be absent.

The suite checks the voltage tables against the data sheet's endpoints and
round-trips every code of every rail, the current and voltage tables, the exact
bytes of every transaction of the rails, the charger, the ADC, the interrupts,
the power key and the commands, the preservation of the bits that share a
register with a voltage, and each refusal. `bash tests/coverage.sh` merges the
suites' line coverage over `src/`.

## Licence

Apache-2.0.
