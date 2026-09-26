# Tronbyt Pixoo64

Turn one Divoom Pixoo64 into two independently scheduled 64×32 Tronbyt-compatible virtual displays stacked vertically on the physical 64×64 panel.

## Status

Research and planning only. No custom firmware has been flashed. The current Pixoo remains on stock firmware and its verified operational renderer lives in the separate `infra` repository.

## Target architecture

```text
Tronbyt server
├── logical device: pixoo-top    (64×32, independent apps/timeline)
└── logical device: pixoo-bottom (64×32, independent apps/timeline)
                  ↓
            compositor bridge
                  ↓ 64×64 DDP
        Pixoo64 running WLED
```

Each logical device keeps its own app rotation, animated-WebP clock, and refresh schedule. The bridge selects the current frame for each device, stacks top and bottom, and transmits only changed 64×64 frames.

## Documents

- [`docs/research.md`](docs/research.md) — verified hardware, firmware options, stock-firmware experiments, sources, and risks.
- [`docs/implementation-plan.md`](docs/implementation-plan.md) — phased execution plan and acceptance criteria.

## Related local projects

- `../tronbyt-server` — current Tronbyt server source.
- `../infra/scripts/render-pixoo-feed-screen.py` — verified stock-firmware renderer; preserve as rollback/reference material.
- `../infra/runbooks/pixoo-direct-render.md` — stock API recovery history and limitations.

## Safety rule

Do not flash this Pixoo until its board revision is photographed, a full 8 MB stock dump has been read twice, and the two dumps have matching SHA-256 checksums. Never connect UART VCC/5 V to the board.
