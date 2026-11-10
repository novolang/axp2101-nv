# Changelog

## 0.1.0

The first release: a driver for X-Powers' AXP2101 power-management chip on an
I2C bus.

- `Axp2101`, the type a board's device table builds from the part's address
  and its optional IRQ pin, implementing devices-nv's `PowerManager` over 14
  rails and `Identifiable`, with its own methods for the gauge, the ADC, the
  charger, the interrupts and the power commands. The rails are DCDC1 to
  DCDC5, ALDO1 to ALDO4, BLDO1, BLDO2, CPUSLDO, DLDO1 and DLDO2, each with
  its voltage table, its enable bit and a voltage write that keeps the other
  bits of its register.
- `battery()` as the cell voltage from the 14-bit ADC, the charging phase from
  STATUS2 and the gauge's percentage, and `None` with no battery present;
  `power_source()` as USB, battery or none.
- The charger: the cell charge enable, the constant, pre-charge and
  termination currents, the voltage limit (one of five values; the 5.0 V code
  is refused), the button-cell charger and its termination voltage, and the
  TS pin as a thermistor input or an external input.
- The fuel gauge's enable, reset and low-battery levels, and the watchdog as
  an option.
- The 23 interrupt sources as named bits of one 24-bit mask, the enables, the
  status, the clearing by writing ones and a service function that reads the
  flags and clears exactly those.
- The power key's long-press, power-off and power-on times and its power-off
  action, and the power-off and restart commands.
- The suite runs over a double of the part and covers every line of `src/`.
- The device program `examples/power_monitor.nv`, a power monitor. Built for
  the nRF52840-DK it is 13,328 B of flash, for the nRF54LM20-DK 14,156 B and
  for the FRDM-MCXN947 23,324 B.

### Known

- Hardware verification: NOT VERIFIED. No board carries an AXP2101.
- The M5Stack CoreS3, the board of the candidates page, has an ESP32-S3, and
  the BSPs of the novo-lang repository have no ESP32-S3 target. The device
  program builds for the nRF52840 DK, the nRF54LM20 DK and the FRDM-MCXN947
  instead. The Nordic board packages answer for I2C as absent hardware, so on
  those images `begin` answers -1 until the boards drive the bus.
- The data sheet read is version 1.4 (October 2022) from a third-party mirror.
  It does not list the identification register (0x03, value 0x4A, from Zephyr
  and XPowersLib), gives no conversion for the die temperature (XPowersLib's
  is used), and disagrees with itself on the top of CPUSLDO and DLDO2 (code 19
  or 18) and of DLDO1 (3.3 or 3.4 V); the driver follows the register tables
  and the ranges, as the README says.
- Not implemented: the RTC LDOs, a rail fault flag (the part has no per-rail
  flag), a battery current (the part measures none), the JEITA profile, the
  charge safety timers, the sleep and wake-up control, the battery model of
  the gauge, and the RSB interface.
