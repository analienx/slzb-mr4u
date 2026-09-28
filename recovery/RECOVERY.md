# MR4U MG26 Thread recovery procedure

This procedure exists to recover the EFR32MG26 / `if00` Thread path without modifying the CC2674P10 production Zigbee radio firmware or Zigbee network state.

## Hard safety boundary

- `if02` / CC2674P10 is the production Zigbee coordinator.
- Never flash the P10 radio.
- Never change its IEEE address, channel, PAN, network key or Zigbee2MQTT database.
- A core ESP32-S3 reboot temporarily interrupts both USB serial interfaces, so Zigbee2MQTT must be stopped cleanly before entering core bootloader mode.
- Always take a **full ESP32 flash backup before writing anything**.

## Verified target Thread configuration

SMLIGHT's official Home Assistant USB Thread instructions specify:

- MG26 runs OpenThread RCP firmware.
- Home Assistant OTBR uses `if00`.
- Baud: **460800**.
- Host hardware flow control: **off**.
- OTBR firmware flashing: **off** (radio must already be flashed).

## Current live state

- P10 / `if02`: production Zigbee, working.
- MG26 / `if00`: upstream MR4U OpenThread RCP 2026.6.1 / 3.1.1 was flashed successfully.
- The existing ESP32 bridge still cannot communicate with the MG26 at runtime.
- Generic CDC RTS/DTR and baudrate-magic bootloader resets do not reach the MG26 through this bridge.
- HA OTBR is configured for 460800 / flow-control off and must remain stopped until Spinel communication is verified.

## Recovery entry point

For U/MRxU devices SMLIGHT documents ESP32-S3 USB bootloader entry as:

1. Stop Zigbee2MQTT cleanly.
2. Remove power from the MR4U.
3. Hold the top button.
4. Connect/power the device over USB while keeping the button held.
5. Release only after the flasher has detected/started erasing when actually flashing.

Because this MR4U is normally PoE powered, entering ROM bootloader requires a deliberate brief production Zigbee outage.

## Phase 1 — backup only

Do **not** write firmware immediately.

Once the ESP32-S3 ROM bootloader enumerates:

1. Identify the new ESP32-S3 serial/JTAG/ROM USB device.
2. Install/run `esptool` in a temporary environment.
3. Read chip info and flash size.
4. Read the complete flash (expected 16 MiB) to a dated backup file.
5. Calculate SHA256.
6. Copy the backup to persistent storage and record the hash in issue #1.
7. Parse the partition table and inspect firmware/version strings before deciding what to flash.

## Phase 2 — restore management path

Preferred order:

1. If the existing core image can be repaired/configured without replacement, preserve it.
2. Otherwise use an official **U/MRxU full SLZB-OS image**.
3. Stable firmware is preferred for the production bridge; development builds require a concrete fix we need.
4. After core boots, establish SLZB-OS management (fallback AP/Wi-Fi or data LAN if available).
5. Confirm P10 identity and USB mapping before changing MG26.

## Phase 3 — configure only MG26 for Thread

In SLZB-OS:

1. Keep P10 / radio1 as the existing production Zigbee coordinator.
2. Select MG26 / radio2 only.
3. Set firmware mode to **Thread to remote OTBR**.
4. Let SLZB-OS perform the MG26 bootloader/reset/firmware/baud transition.
5. Return the device to USB mode.
6. Verify `if02` is still the P10 and `if00` is the MG26.

## Phase 4 — Home Assistant

1. Start Zigbee2MQTT and verify production traffic first.
2. Probe MG26/if00 for Spinel at 460800 with host flow control off.
3. Start HA OpenThread Border Router only after the probe succeeds.
4. Confirm OTBR initializes without RCP timeouts.
5. Confirm Home Assistant Thread integration has a preferred network.
6. Confirm Matter Server is running.
7. Sync Thread credentials to the phone.
8. Commission the Matter-over-Thread device.

## Rollback

If the ESP32 core fails to boot or USB mapping is wrong, restore the **full flash backup** byte-for-byte before doing anything to the P10 radio.

If MG26 Thread configuration fails but the core is healthy, use SLZB-OS radio firmware management to retry or restore MG26 only. Do not use P10 as part of MG26 recovery.
