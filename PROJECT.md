# Project notes

## Goal

Integrate a Divoom Pixoo64 with Tronbyt as two simultaneous, independently scheduled 64×32 logical displays on one 64×64 panel.

## Safety

- Treat the existing Pixoo as working production equipment until a separate decision authorizes opening or flashing it.
- Do not flash before recording the board revision and obtaining two matching full-flash backups.
- Use 3.3 V UART logic only. Connect TX, RX, GND, and temporarily strap IO0 to GND; do not connect adapter VCC/5 V.
- Keep credentials, API keys, Wi-Fi passwords, and private endpoint tokens out of Git.
- Preserve a tested path back to stock firmware.

## Engineering principles

- Prefer the proven Pixoo64 WLED driver and DDP path for the first prototype.
- Keep the two 64×32 logical devices on independent timelines; combine only at the final compositor.
- Treat HTTP/DDP acknowledgement and visual correctness as separate acceptance gates.
- Test long-duration stability before calling the device operational.
- Record hardware revision, firmware hashes, build inputs, and restoration commands.
