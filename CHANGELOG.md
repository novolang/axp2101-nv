# Changelog

## 0.1.2 — 2026-11-27

The power-monitor example, in `examples/power_monitor.nv`, calls `pmu.enable_irq` without a binding.  No signature of the package changed.

## 0.1.1 — 2026-11-23

The package asks for embedded-hal-nv ^0.4.0 (was ^0.3.2) and devices-nv ^0.4.0 (was ^0.3.2), the minor of both trait packages that the other drivers and the board packages ask for, so one version of each serves a program that takes this package beside any of them.  Under the older ranges, a program that named this package and a driver on 0.4 stopped with "no version of 'embedded-hal-nv' satisfies every requirement".

No signature of the package changed and the suite passes without an edit.  `examples/power_monitor.nv` adds the `ChargeActive` arm that devices-nv 0.4.0 gives `ChargeState`, which its `match` on the charging phase needs.

The device program builds for the boards its README names, flash nrf52840-dk 16,680 B before and 16,724 B after; nrf54lm20-dk 17,760 B before and 17,784 B after; frdm-mcxn947 18,480 B before and 18,544 B after; nrf52-qemu 8,580 B before and 8,588 B after.

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
  the nRF52840-DK it is 21,160 B of flash, for the nRF54LM20-DK 22,120 B and
  for the FRDM-MCXN947 23,376 B; before the Nordic buses landed, with the
  register paths left out, the two DK images were 13,328 B and 14,156 B.

### Known

- Hardware verification: NOT VERIFIED. No board carries an AXP2101.
- The M5Stack CoreS3, the board of the candidates page, has an ESP32-S3, and
  the BSPs of the novo-lang repository have no ESP32-S3 target. The device
  program builds for the nRF52840 DK, the nRF54LM20 DK and the FRDM-MCXN947
  instead. The nRF52840 DK drives I2C with TWIM0 and the nRF54LM20 DK
  with TWIM22, so their images carry the register paths; a reply from the part
  over either bus is NOT VERIFIED.
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
