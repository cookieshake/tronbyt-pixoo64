# Research notes

Last verified: 2026-09-26

## Objective

Use one physical Pixoo64 as two simultaneous Tidbyt/Tronbyt-style displays:

- top viewport: 64×32, logical Tronbyt device `pixoo-top`;
- bottom viewport: 64×32, logical Tronbyt device `pixoo-bottom`;
- each viewport has independent apps, app-rotation timing, refresh timing, and animated-WebP playback;
- a compositor produces a physical 64×64 frame and sends it to the Pixoo.

This is simultaneous composition, not alternating the whole Pixoo between two screens.

## Hardware assessment

### Pixoo64 evidence

The Pixoo64 WLED port documents and demonstrates:

- ESP32-WROVER-IE, ESP32-D0WD-V3 rev 3.1;
- 8 MB flash;
- 4 MB PSRAM;
- 64×64 RGB panel;
- separate LED-driver board controlled by the ESP32 over SPI;
- panel bus: CLK GPIO25, MOSI GPIO33, CS GPIO26, SPI mode 0 at 8 MHz;
- a full RGB888 frame is 12,288 bytes plus protocol framing/padding;
- approximately 29–30 fps sustained DDP operation on real hardware;
- 2.4 GHz Wi-Fi through the ESP32;
- USB-C is power-only on the documented board;
- OTA works after the first UART installation.

Known board revisions in the two replacement-firmware projects:

- `Pixoo64-wifi MainBoard REV1-2` in the WLED port;
- `Pixoo64-Wifi Mainboard REV1.1 (20210318)` and LEDBoard REV1.0 in `simplyRoba/pixoo-firmware`.

The revision inside the local Pixoo is unknown until the enclosure is opened and photographed.

### Comparison with Tronbyt hardware

The current Tronbyt ESP32 firmware builds Tidbyt Gen1 and Gen2 for the classic `esp32` target with:

- 8 MB flash configuration;
- PSRAM enabled;
- 64×32 logical output;
- animated WebP download and playback.

Therefore the Pixoo CPU, flash, and PSRAM are in the same class as the existing Tidbyt targets. It drives twice as many pixels, but the real WLED port proves that the panel path has ample throughput. Hardware capacity is not the main blocker.

## Software compatibility assessment

### Why stock Tronbyt firmware is not drop-in

Tronbyt's existing display implementation drives HUB75 panels directly. Pixoo64 instead sends framed RGB888 packets to a separate LED-driver board over its private SPI protocol. A Pixoo-specific output backend is required.

The local Tronbyt server currently renders a 64×32 logical canvas in its normal path. That is useful for this project: two unmodified outputs can be composed into the Pixoo's 64×64 frame. Existing Pixlet apps do not need to become native 64×64 apps.

### Recommended first architecture: WLED plus bridge

1. Flash the proven Pixoo64 WLED build after backup and board validation.
2. Register two logical devices in Tronbyt: `pixoo-top` and `pixoo-bottom`.
3. A bridge consumes each device's `/next` WebP output and maintains independent playback state:
   - current app;
   - decoded frames;
   - per-frame durations;
   - app display deadline;
   - refresh/poll deadline;
   - last good frame.
4. At a shared output tick, choose the current frame from each independent timeline.
5. Copy the top frame to rows 0–31 and the bottom frame to rows 32–63.
6. Send the composed 64×64 RGB frame to WLED over DDP port 4048.
7. Skip transmission when the composed frame has not changed.

Different app durations and animation frame rates are not a problem: the compositor keeps two clocks and merges only the selected frames at output time.

Physical power and brightness remain shared. Apps, schedules, transitions, and animation clocks can be independent.

### Longer-term architecture: native Tronbyt Pixoo target

A native port can eliminate the external bridge:

- add `BOARD_PIXOO64` to `tronbyt/firmware-esp32`;
- port the verified Pixoo SPI packet writer from WLED or ESPHome;
- add the Pixoo board/pin configuration and safe brightness handling;
- add a device type and firmware artifact to Tronbyt server;
- either keep two logical feeds and compose on-device, or add a first-class paired-device protocol.

This is cleaner operationally but requires more embedded work than the bridge proof of concept.

## Replacement-firmware candidates

### 1. acidmiku/pixoo64-wled — recommended prototype base

- Repository: <https://github.com/acidmiku/pixoo64-wled>
- Guide: <https://github.com/acidmiku/pixoo64-wled/blob/pixoo64/PIXOO64.md>
- Inspected commit: `b93858a86f0e47243b6b8c1ee4706801e74c47f8`
- Provides board photographs, UART/flash procedure, stock restore, WLED OTA, 64×64 matrix setup, and a DDP soak sender.
- Reports about 29 fps sustained DDP and preserves stock partition offsets.
- Best fit for proving the two-device compositor because DDP already accepts full frames reliably.

### 2. simplyRoba/pixoo-firmware — native dashboard reference

- Repository: <https://github.com/simplyRoba/pixoo-firmware>
- Inspected commit: `1b84bab4bca3449ab1d7a822c2ede97a4d79931b`
- ESPHome-based complete replacement firmware with dashboards and OTA.
- Useful as a second hardware/panel reference.
- Very young project as of the research date: created in August 2026, no formal GitHub release when checked, and only one board revision documented as tested.

### 3. esp32-divoom — not replacement firmware

- Repository: <https://github.com/d03n3rfr1tz3/esp32-divoom>
- Inspected commit: `2e8e51a6b3dd16bfdaa045ec2240b99b8db41771`
- This is a separate ESP32 Bluetooth/network proxy for Divoom devices, not firmware installed on the Pixoo64. It does not remove the stock firmware's renderer limitations.

### Tronbyt firmware reference

- Repository: <https://github.com/tronbyt/firmware-esp32>
- Inspected commit: `5013f42acd4772a0c4fdf47bb78311b47d8e4c0b`
- Supports classic ESP32 Tidbyt Gen1/Gen2 plus several ESP32-S3 matrix targets.
- Includes a 64-row panel option for supported HUB75 boards, but no Pixoo64 SPI display backend.

## Stock-firmware experiments and conclusions

The working stock renderer is maintained in `../infra` and should not be duplicated here.

Verified facts from live testing:

- Native clock plus HTTP text items works reliably with a two-frame 64×64 background and `[type 5, type 23, type 23]` ItemList.
- Official Galmuri7 and Galmuri9 BDF files can be parsed and rasterized correctly without TTF conversion.
- A finite preloaded Korean ticker works while its complete frame set fits the device's accepted batch.
- A larger Galmuri9 batch failed late in upload; a reduced 63-frame batch succeeded.
- Per-frame live streaming is not operationally safe:
  - one temporary run acknowledged 519 independent `PicNum: 1` frames;
  - the requested repeat caused the Pixoo itself to reboot after only three frames.
- Consequently, HTTP acknowledgement alone is not evidence of stable live rendering.
- The stock device was restored to the static clock/FEED/AT screen, with the bottle icon removed and the background set to RGB `(0,0,0)`.

This is the primary motivation for moving animation and composition to replacement firmware.

## Custom pages and built-in rotation

Official Divoom API documentation confirms:

- `Channel/SetCustomPageIndex` selects one of three app-uploaded custom-page slots (`0..2`).
- The HTTP draw payload does not include a target page slot and cannot store multiple independent dynamic ItemList layouts for later switching.
- `RotationFlag` rotates built-in faces and gallery GIF content; it does not rotate multiple HTTP Custom layouts.
- `GyrateAngle` is the physical screen orientation setting.

Thus built-in page selection does not solve independent dynamic Tronbyt-style panes.

## Physical access and flashing

### Disassembly references

- Screen replacement/disassembly video: <https://youtu.be/JWu1Zo8wW14>
- WLED firsthand report: <https://www.reddit.com/r/WLED/comments/1v6ktlf/flashed_wled_to_a_pixoom64/>
- WLED guide with front/back board photographs: <https://github.com/acidmiku/pixoo64-wled/blob/pixoo64/PIXOO64.md>

There is no complete iFixit-style shell-opening guide in the sources found. Use the screen-replacement video for enclosure access and the WLED photos for board/pad identification.

### Required equipment

- 3.3 V logic USB-to-UART adapter, such as a genuine CP2102/CH340/FT232 implementation;
- TX, RX, GND, and IO0 access using soldered fine wire or reliable pogo pins;
- soldering equipment if the test pads cannot be held reliably;
- normal Pixoo USB-C power supply;
- Mac USB adapter as required.

Do not connect UART VCC or 5 V. The Pixoo remains powered through its normal USB-C input.

### Mandatory stock backup gate

Before any write:

1. identify and photograph mainboard and LED-board revisions;
2. confirm UART logic level and pad labels against the exact revision;
3. read the full flash twice;
4. verify both files are exactly 8 MB;
5. verify their SHA-256 hashes match;
6. store one copy outside the development machine;
7. test the exact stock-restore command syntax without executing it;
8. only then consider flashing WLED.

Reference read command from the WLED guide:

```sh
esptool --port <PORT> --baud 921600 read-flash 0x0 0x800000 stock_backup.bin
```

Treat the port, baud rate, chip detection, and board revision as values to verify on the actual unit, not assumptions.

## Risks

- Unknown local board revision may not match either documented project.
- Incorrect 5 V UART logic or wiring can permanently damage the ESP32.
- Flashing without a verified full backup can make stock recovery impossible.
- Replacement firmware removes Divoom app/cloud and the current stock HTTP API.
- Antenna pigtail damage can cause weak Wi-Fi and DDP drops.
- A bridge introduces another always-on service and must recover cleanly from stale/missing logical-device frames.
- Long animated WebPs may stress decode buffers; enforce limits and test soak behavior.
- Top and bottom share physical brightness, power, network, and reboot state.

## Current recommendation

Hardware feasibility is high; software is not drop-in. The lowest-risk technical route is:

1. board-revision inspection only;
2. verified stock backup;
3. WLED hardware validation;
4. host-side two-device compositor bridge;
5. extended soak test;
6. only then decide whether a native Tronbyt firmware target is worth maintaining.
