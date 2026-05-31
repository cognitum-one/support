# ESP32 CSI sensor onboarding

How to flash and connect an ESP32 WiFi‑CSI sensor node ("RuView" node) so a
Cognitum fleet (Seeds + V0 appliance) can consume its stream.

> Scope: this complements the Pi‑Zero **Seed** flow in
> [`getting-started.md`](./getting-started.md). ESP32 nodes are **data‑plane
> sensors** — they stream CSI/vitals over UDP to a consumer; they are not
> attested mesh members like Seeds.

## Supported hardware

| Chip | Flash | Status | Image set |
|------|-------|--------|-----------|
| **ESP32‑S3** | 8 MB | **production** | `bootloader-s3.bin`, `partition-table-s3.bin`, `ota_data_initial-s3.bin`, `esp32-csi-node-s3-8mb.bin` |
| **ESP32‑C6** | 4 MB | research (Wi‑Fi 6 / 802.15.4 / TWT / LP‑core) | `bootloader-c6.bin`, `partition-table-c6.bin`, `ota_data_initial-c6.bin`, `esp32-csi-node-c6-4mb.bin` |

Recommended S3 boards: ESP32‑S3‑DevKitC‑1, XIAO ESP32‑S3 (8 MB).

## Firmware

Prebuilt binaries are published as GitHub releases tagged `vX.Y.Z-esp32` on
[`ruvnet/RuView`](https://github.com/ruvnet/RuView/releases). **Use `v0.6.3-esp32`
or later** — older firmware emits raw audio amplitudes only and the RuView CSI
features (head‑height proxy, subcarrier amplitudes, motion vectors) silently
degrade. Verify downloads against the release `SHA256SUMS.txt`.

## Flash (esptool ≥ 5.0)

```bash
pip install 'esptool>=5.0' esp-idf-nvs-partition-gen

# ESP32-S3 (8 MB)
python -m esptool --chip esp32s3 --port <PORT> --baud 460800 \
  write_flash --flash_mode dio --flash_size 8MB \
  0x0     bootloader-s3.bin \
  0x8000  partition-table-s3.bin \
  0xf000  ota_data_initial-s3.bin \
  0x20000 esp32-csi-node-s3-8mb.bin

# ESP32-C6 (4 MB): same offsets, --chip esp32c6, --flash_size 4MB, -c6 bins.
```

Offsets are identical for both chips: `bootloader=0x0`, `partition-table=0x8000`,
`otadata=0xf000`, `app(ota_0)=0x20000`. If sync fails, hold **BOOT**, tap
**RESET**, release **BOOT**, and retry.

## Provision (NVS, no reflash)

`provision.py` (in the firmware tree) writes WiFi + target config to NVS. The
release README under‑documents it — the full flag set includes Seed‑attach and
swarm options:

```bash
python provision.py --port <PORT> [--chip esp32s3|esp32c6] \
  --ssid "<WIFI_SSID>" --password "<WIFI_PASSWORD>" \
  --target-ip <CONSUMER_IP> --target-port <PORT> --node-id <N> \
  # optional Seed attach:
  --seed-url http://<SEED_IP> --seed-token <SEED_BEARER_TOKEN> \
  --zone <zone-name> --swarm-hb 30 --swarm-ingest 5
```

`provision.py` merges with prior per‑port state by default (`--reset` to wipe).

## Where to point the node (`--target-ip` / `--target-port`)

The node emits one ADR‑069 UDP packet (magic `0xC5110003`, 8 LE‑f32 features).
Point it at **whatever consumes that packet** — the bind port differs by consumer:

| Consumer | Port |
|----------|------|
| Firmware default | `5005` |
| Seed agent / a CSI cog on a Seed (`--source esp32-udp=0.0.0.0:5006`) | `5006` |
| V0 appliance RVF aggregator | `5008` |

## Which cogs use the CSI stream

The same packet feeds any cog calling `cog_sensor_sources::fetch_sensors()`;
whether a cog "uses ruview" is how it interprets the 8 features
(see `RUVIEW-CAPABILITY-MATRIX`). Documented integration:

- **Required (won't run without CSI):** `package-detect`, `parking-occupancy`, `ppe-compliance` (needs `ruview-densepose` upstream).
- **Dedicated CSI cogs:** `ruview-densepose` (CSI → 17‑keypoint skeleton), `health-monitor` (vitals/presence/apnea).
- **Optional (`--ruview-mode`):** `fall-detect`, `gunshot-detect`, `slip-fall-zone`, `smoke-fire`.
- **Not CSI (audio/other):** cough-detect, baby-cry, snore-monitor, glass-break, water-leak, frost-warning, beehive-monitor, predictive-maintenance.

The node is cog‑agnostic — capability is decided by which cog you deploy and where.

## Known gaps

- The V0 `/edge` "Download RuView firmware" link / registration default points at
  `v0.6.0-esp32`, below the `v0.6.3-esp32` CSI minimum. Pick a newer release.
- The mapping between the V0 `/edge` PSK (device‑id‑bound, for
  `/api/v1/v0/swarm/esp32/*`) and `provision.py --seed-token` (a Seed pairing
  bearer token) is undocumented. See issue #30.
