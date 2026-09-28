# SLZB-MR4U — MG26 Thread / Home Assistant lab

Dedicated firmware and integration workspace for the **SMLIGHT SLZB-MR4U** dual-radio coordinator.

## Goal

Keep the **CC2674P10** radio untouched as the production Zigbee coordinator while converting the second **EFR32MG26** radio into an **OpenThread RCP** for Home Assistant / Matter.

Target architecture:

```text
CC2674P10 (if02) -> Zigbee2MQTT -> production Zigbee network
EFR32MG26 (if00) -> OpenThread RCP -> HA OpenThread Border Router -> Matter
```

The production P10 Zigbee network is a hard boundary: firmware, IEEE identity, Zigbee2MQTT serial configuration and network state must not be changed by work in this repository.

## Current hardware mapping

| Interface | Radio | Intended role |
| --- | --- | --- |
| `if02` | TI CC2674P10 | Production Zigbee coordinator |
| `if00` | Silicon Labs EFR32MG26 | Thread/OpenThread RCP |

MR4U is connected to Home Assistant by USB. In this installation the Ethernet connection provides PoE power only; it is **not** used as a management/data path.

## Current Thread work

The vendor/upstream MR4U OpenThread build uses **460800 baud + RTS/CTS hardware flow control**. The current MR4U USB bridge path was operating with the radio side at **115200**, so this repository carries a dedicated MR4U OpenThread RCP build variant intended to match the existing USB path rather than modifying the production P10 radio.

The custom variant keeps the MR4U MG26 pin mapping but uses:

- OpenThread RCP
- EFR32MG26B420F3200IM48
- 115200 baud
- no UART flow control
- EUSART0
- MR4U TX/RX mapping: PA5 / PA6

## Build

GitHub Actions builds the custom firmware using the existing Nerivec Silicon Labs firmware-builder toolchain container. This avoids rebuilding or downloading the Silicon Labs toolchain on every run.

The workflow only produces firmware artifacts. **It does not flash Home Assistant or the MR4U automatically.**

## Safety rules

1. Never flash or reconfigure `if02` / CC2674P10 from this project.
2. Never alter the production Zigbee IEEE address or Zigbee2MQTT network database.
3. All firmware experiments target **MG26 / if00 only**.
4. Verify the generated GBL metadata and target before any live flash.
5. Keep Home Assistant OTBR stopped until MG26 responds correctly as a Spinel/OpenThread RCP.
6. Any recovery/reset procedure must be specific to the EFR32MG26 path and must not reset the full MR4U or P10.

## Status

- Production Zigbee on P10: preserved and operational.
- MG26: converted away from its former diagnostic Ember/Zigbee role toward Thread.
- HA OpenThread Border Router: installed/configured, but kept stopped until the MG26 serial/runtime path is validated.
- Next milestone: produce and validate the 115200/no-flow MR4U OpenThread RCP image, then recover/flash MG26 through a radio-specific path and verify Spinel communication before starting OTBR.

## Upstream references

- SMLIGHT SLZB-MR4U hardware / SLZB firmware ecosystem
- Nerivec `silabs-firmware-builder`
- Home Assistant OpenThread Border Router
- OpenThread RCP / Spinel
