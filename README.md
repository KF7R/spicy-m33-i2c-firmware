# XXtra Spicy E22-900M33S / I2C firmware

Target: `spicy_m33_i2c`, based on official Meshtastic `v2.8.1.8e6a88d` / commit `8e6a88d06f44cad26f1e8d7cd402939ccacb9b4c`.

This build extends the unmodified upstream `seeed_xiao_nrf52840_kit_i2c` environment and adds `SX126X_MAX_POWER=8`. No upstream radio implementation or kit variant file was edited. The separate E22P custom environment is not used.

## Intended board and pin assignments

- XXtra Spicy v2.2b with Seeed XIAO nRF52840 and EBYTE E22-900M33S, as identified in the user's photo and the [manufacturer setup guide](https://www.r48d.com/docs/xxtra%20spicy%20v2.2b.pdf).
- I2C SDA: D6 / P1.11. I2C SCL: D7 / P1.12. These match the Wire initialization and native pin map recovered from the known `spicy_2w_i2c-2.8.1.d4482a2` UF2.
- Standard Seeed kit radio pinout: CS D4, DIO1 D1, BUSY D3, RESET D2, RXEN D5; radio DIO2 provides TX switching, with MCU TXEN not connected.
- SPI: SCK D8, MISO D9, MOSI D10. TCXO: 1.8 V via DIO3.
- GPS remains enabled on NFC pins D30/D31, as in upstream's I2C kit environment; button on D0.

The manufacturer's guide directs users to the Seeed nRF52840 Kit firmware. It is a setup guide, not a schematic; individual radio wiring has not been independently traced on this board.

## Power behavior

The app's transmit-power value remains chip power, matching the manufacturer's guide. No PA gain compensation is added (`TX_GAIN_LORA` remains the upstream default of 0). This deliberately does not enable `EBYTE_E22_900M33S`, whose upstream configuration would subtract 25 dB and change the meaning of saved app values.

The chip maximum is 8 dBm. Upstream `SX126xInterface::reinitChip()` and `programModemParams()` call `limitPower(SX126X_MAX_POWER)` before `begin()` / `setOutputPower()`. The upper clamp in `RadioInterface::limitPower()` also applies in licensed mode. Source review covers initial setup, reconfiguration, and chip recovery. Hardware behavior is untested.

According to the manufacturer's table, an app value of 5 dBm corresponds to approximately 1 W output, and 8 dBm to approximately 2 W. Those are the manufacturer's nominal values, not measurements of this unit. This build does not change the saved region or TX-power setting; values above 8 are clamped before programming the radio. Keep the configured value at 8 or below.

## Reproduction

Check out the official commit above, place the supplied `spicy_m33_i2c/platformio.ini` in `variants/nrf52840/spicy_m33_i2c/`, and run `pio run -e spicy_m33_i2c`.

The portable configuration inherits upstream's pinned framework and compiler dependencies. The local build uses the same framework commit `0fd295f13203e93df19d578073646ec32f2bf45a` via a workspace-local package override, with its pinned TinyUSB and nRFCrypto dependencies. The override is omitted from the delivered portable configuration.

## Update procedure

1. Back up your configuration and keys if desired. This is an application update; do not use a factory-erase image for an already configured node.
2. Disconnect the Spicy board's power and remove the XIAO from its socket. The manufacturer's guide recommends flashing it separately.
3. Connect the removed XIAO to USB, double-tap reset, and copy the supplied UF2 onto its bootloader drive. Wait for completion and restart.
4. Confirm the firmware version and set TX power to 8 dBm or less before returning the XIAO to the Spicy board.
5. Disconnect USB, reinstall the XIAO, attach the antenna, and power the board. Never connect the Spicy board's battery and USB simultaneously.

Normal UF2 application updating is intended to preserve configuration, but preservation and RF/sensor operation have not been tested on the user's hardware. No device was flashed or modified by this task.

## Build verification

Build succeeded: flash 692,508 / 815,104 bytes (85.0%); RAM 96,540 / 248,832 bytes (38.8%). Upstream warm-region, ISR and variant guards passed. UF2 has 2,706 valid blocks, nRF52840 family ID 0xADA52840, application addresses 0x27000 through 0xD0200 (padded), and embedded version 2.8.1.8e6a88d / environment spicy_m33_i2c.

Compiled disassembly confirms limitPower receives 8 in both radio startup and modem reconfiguration, and Wire initializes pins D6/D7. initVariant does not contain the E22P D5-high customization.

UF2 SHA256: `6d074f397a9c6bc612bdbf04666c6cd388622ab7b11e6b7dac5cc7d53926fa8c`.
