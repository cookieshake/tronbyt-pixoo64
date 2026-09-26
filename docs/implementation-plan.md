# Implementation plan

No phase after Phase 0 is authorized merely by this document. Each hardware-changing phase requires an explicit go/no-go decision.

## Phase 0 — preserve current state

Deliverables:

- keep the stock Pixoo operational;
- preserve links to the stock renderer and recovery runbook in `../infra`;
- record current device network identity without storing credentials;
- decide whether the current unit or a second Pixoo will be the experiment target.

Exit criteria:

- known-good stock screen remains visually verified;
- recovery command and native-clock rollback remain available.

## Phase 1 — read-only physical inspection

Actions:

- disconnect power;
- open the enclosure using the disassembly video as a visual reference;
- photograph enclosure steps, all connectors, both sides of each board, antenna routing, and every silkscreen revision;
- identify TX, RX, GND, and IO0 pads without connecting an adapter;
- close and retest the stock unit.

Exit criteria:

- exact mainboard and LED-board revisions recorded;
- board matches a supported/reference design or differences are understood;
- stock screen still works after reassembly.

Stop condition:

- undocumented board revision, damaged connector, unclear UART pads, or inaccessible test points.

## Phase 2 — UART access and backup only

Actions:

- use a 3.3 V logic USB-UART adapter;
- connect TX↔RX, RX↔TX, and GND; strap IO0 to GND only for download mode;
- leave adapter VCC/5 V disconnected;
- power the Pixoo through its own USB-C input;
- identify the ESP32 with `esptool`;
- read the entire 0x800000-byte flash twice;
- calculate and compare SHA-256 hashes;
- store an offline copy.

Exit criteria:

- two matching 8 MB dumps;
- chip/flash properties recorded;
- device boots stock firmware after disconnecting IO0;
- current display remains functional.

Stop condition:

- mismatched dumps, brownouts, unstable serial, unexpected flash size, secure boot/encryption, or inability to restore normal boot.

## Phase 3 — WLED proof

Actions:

- pin a reviewed `pixoo64-wled` commit/release;
- independently build or verify downloaded artifact hashes;
- flash the required application slots, partition table, and otadata exactly as documented;
- configure one 64×64 non-serpentine matrix;
- verify Wi-Fi persistence and OTA;
- run the repository DDP soak sender;
- test static colors, brightness, orientation, frame ordering, and 30 fps stability;
- record thermal and network behavior.

Exit criteria:

- correct full-panel RGB output;
- no pixel mapping or color-order errors;
- at least a multi-hour DDP soak without reboot or corruption;
- OTA works;
- stock restore remains available.

Rollback:

- UART-write the verified full stock dump at offset 0x0 and visually verify the stock screen.

## Phase 4 — offline compositor prototype

Implement a host-side service with no live device dependency.

Components:

- `LogicalDisplay` state for top and bottom;
- animated-WebP decoder preserving frame durations and loop metadata;
- monotonic-clock timeline selector per logical display;
- 64×64 compositor;
- DDP encoder/output abstraction;
- unchanged-frame suppression;
- last-good-frame and black-frame fallbacks;
- metrics and structured logs.

Tests:

- static + static;
- animated + static;
- animations with different frame rates;
- app changes at unrelated times;
- one endpoint unavailable while the other continues;
- loop boundaries and zero/invalid duration handling;
- exact row placement: top 0–31, bottom 32–63;
- no cross-pane pixel contamination;
- deterministic frame selection under a fake clock.

Exit criteria:

- generated frame fixtures prove independent timing and correct composition;
- no credentials are required in source control.

## Phase 5 — Tronbyt integration

Preferred proof-of-concept behavior:

- register `pixoo-top` and `pixoo-bottom` as separate logical devices;
- preserve each device's independent app list, display duration, update interval, and schedule;
- consume the same WebP output used by existing devices;
- atomically replace one pane's playback state without resetting the other;
- continue showing the last good frame when one logical device is temporarily unavailable.

Open design question:

- whether to implement the bridge as a first-class Tronbyt server output, a standalone authenticated client, or a small companion service using a stable server API. Choose only after inspecting the deployed server's authentication and `/next` contract.

Exit criteria:

- both logical devices appear and schedule independently;
- changing one app does not interrupt the other;
- refresh and animation cadence remain correct for both panes.

## Phase 6 — live WLED bridge test

Safety defaults:

- start at low brightness;
- cap output at 10 fps initially;
- raise toward 30 fps only after stability checks;
- keep a watchdog and explicit black/static fallback;
- stop output on repeated DDP/network faults rather than flooding retries.

Acceptance tests:

- top and bottom show different static apps simultaneously;
- one pane can animate while the other remains static;
- both panes can animate at different frame rates;
- app rotation deadlines differ and remain independent;
- network interruption recovers without device reboot;
- at least 24-hour soak with no reboot, corruption, memory growth, or schedule drift;
- visual confirmation of correct colors, orientation, and row boundary.

## Phase 7 — native Tronbyt target decision

Only pursue if the external bridge is operationally burdensome.

Potential work:

- add `BOARD_PIXOO64` in `tronbyt/firmware-esp32`;
- port the Pixoo SPI packet protocol and padding requirement;
- add 64×64 double buffering and brightness control;
- define paired logical feeds or on-device composition;
- add a `pixoo64` device type and firmware artifact in Tronbyt server;
- implement OTA and stock-recovery documentation.

Decision criteria:

- measured bridge maintenance burden;
- upstream willingness to accept a Pixoo target;
- confidence in long-term board compatibility;
- whether on-device composition materially improves reliability.

## Definition of done

The project is operational only when:

- the Pixoo runs replacement firmware with a verified stock recovery image;
- two 64×32 logical Tronbyt devices are visible simultaneously;
- timelines, animations, app rotations, and refresh schedules are independent;
- 24-hour soak passes;
- reboot and network-loss recovery are verified;
- visual acceptance is confirmed;
- build inputs, firmware hashes, hardware revision, flashing procedure, rollback, and service deployment are committed without secrets.
