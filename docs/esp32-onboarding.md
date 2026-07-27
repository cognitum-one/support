# ESP32 CSI sensor onboarding

How to flash and connect an ESP32 WiFi‑CSI sensor node ("RuView" node) so a
Cognitum fleet (Seeds + V0 appliance) can consume its stream.

> Scope: this complements the Pi‑Zero **Seed** flow in
> [`getting-started.md`](./getting-started.md). ESP32 nodes are **data‑plane
> sensors** — they stream CSI/vitals over UDP to a consumer; they are not
> attested mesh members like Seeds.

## Supported hardware

| Chip | Flash | Status |
|------|-------|--------|
| **ESP32‑S3** | 8 MB | **production** |
| **ESP32‑C6** | 4 MB | research (Wi‑Fi 6 / 802.15.4 / TWT / LP‑core) |

Recommended S3 boards: ESP32‑S3‑DevKitC‑1, XIAO ESP32‑S3 (8 MB).

Recent releases publish S3 images only; C6 images are not built for every tag.

## Firmware

Prebuilt binaries are published as GitHub releases tagged `vX.Y.Z-esp32` on
[`ruvnet/RuView`](https://github.com/ruvnet/RuView/releases). **Use the latest
`v0.8.x-esp32` release.** Anything below `v0.6.3-esp32` emits raw audio
amplitudes only, and the CSI features (head‑height proxy, subcarrier amplitudes,
motion vectors) silently degrade.

You need four artifacts. **Asset names have changed between releases**, so take
them from the release you are installing rather than copying names from here:

| Role | Flash offset | Typical name |
|------|--------------|--------------|
| Bootloader | `0x0` | `bootloader.bin` |
| Partition table | `0x8000` | `partition-table.bin` |
| OTA data | `0xf000` | `ota_data_initial.bin` |
| Application | `0x20000` | `esp32-csi-node-*-8mb.bin` |

The newest releases ship these together as a single `…-8mb-flash-bundle.zip` —
unzip it and you have all four. Older tags list them as separate assets.

A `SHA256SUMS.txt` is published on some releases but not all; verify against it
when present.

## Flash (esptool ≥ 5.0)

```bash
pip install 'esptool>=5.0' esp-idf-nvs-partition-gen

# ESP32-S3 (8 MB)
python -m esptool --chip esp32s3 --port <PORT> --baud 460800 \
  write-flash --flash-mode dio --flash-size 8MB \
  0x0     bootloader.bin \
  0x8000  partition-table.bin \
  0xf000  ota_data_initial.bin \
  0x20000 esp32-csi-node-<version>-8mb.bin

# ESP32-C6 (4 MB): same offsets, --chip esp32c6, --flash-size 4MB, C6 images.
```

esptool 5 renamed subcommands and flags to hyphenated forms. The old
`write_flash` / `--flash_mode` / `--flash_size` spellings still work but print a
deprecation warning.

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
That state is stored **per machine**, so if you re‑provision the same node from a
different computer there is nothing to merge from — pass every flag again, or you
will silently drop keys the node previously had. `--state` prints what would be
written without flashing.

## Where to point the node (`--target-ip` / `--target-port`)

| Consumer | Port |
|----------|------|
| Firmware default | `5005` |
| Seed | `5006` |
| V0 appliance RVF aggregator | `5008` |

The firmware defaults to **5005**, but a Seed listens on **5006**. If you leave
the default, the Seed never sees the stream. Re‑provision with
`--target-port 5006` (no reflash needed).

> **Do not configure a cog to bind `0.0.0.0:5006` itself.** On a Seed that port
> belongs to `cognitum-csi-relay`, which owns the stream and forwards it to each
> cog on a private loopback port. A cog that binds 5006 directly collides with
> the relay — see [Troubleshooting](#troubleshooting) below.

### What the node actually emits

The node emits **several** packet types on the same UDP port, distinguished by a
4‑byte little‑endian magic. Measured on firmware v0.8.4 at `--edge-tier 2`:

| Type | Magic | Size | Typical rate |
|------|-------|------|--------------|
| Raw CSI | `0xC5110001` | ~276 B | ~34 /s |
| Vitals | `0xC5110002` | 32 B | ~1 /s |
| Feature vector | `0xC5110003` | 48 B | ~1 /s |
| Compressed | `0xC5110005` | varies | occasional |
| Feature state | `0xC5110006` | 60 B | ~1 /s |
| Sync | `0xC511A110` | varies | ~1.7 /s |

Rates vary with `--edge-tier`, motion, and radio conditions. `--edge-tier 0`
disables on‑device DSP entirely (raw passthrough only, no vitals) — the default
is tier 2, which is what you want.

Most health and presence cogs consume the **32‑byte vitals packet**
(`0xC5110002`), not the feature vector. A Seed forwards only the packet types a
given cog subscribes to, so seeing plenty of raw CSI in a packet capture does not
mean your cog is receiving anything.

## Setting up a cog to consume the stream (Seed)

Flashing and provisioning the node is only half the job. The Seed drops CSI
frames unless a cog is installed **and** pointed at the ESP32 source.

```bash
SEED=http://<SEED_IP>

# 1. Install the cog
curl -X POST $SEED/api/v1/apps/install \
  -H 'Content-Type: application/json' -d '{"id":"health-monitor"}'

# 2. Point it at the ESP32 stream
curl -X POST $SEED/api/v1/apps/health-monitor/data-source \
  -H 'Content-Type: application/json' -d '{"source_id":"esp32-udp"}'

# 3. RESTART THE COG — required, see note below
curl -X POST $SEED/api/v1/apps/health-monitor/stop
curl -X POST $SEED/api/v1/apps/health-monitor/start
```

> **Step 3 is not optional.** Installing a cog starts it immediately, and
> changing the data source does **not** restart a running cog — it only takes
> effect the next time the cog launches. Until you restart it, the cog keeps
> running with its previous configuration and quietly serves demo data. Every
> call above returns `200 OK` either way, so there is no error to notice.
>
> A future release will restart the cog for you. Until then, always restart
> after changing a data source.

## Verifying you have real data

```bash
curl $SEED/api/v1/apps/health-monitor/logs
```

Check the **`source`** field in the output — it is the only reliable indicator:

| `source` value | Meaning |
|----------------|---------|
| `auto:esp32-vitals` | Real device‑computed vitals |
| `auto:esp32-udp` | Real CSI features |
| `auto:seed-stream` | **Synthetic demo data** |

A healthy result looks like:

```json
{"breathing_bpm":7.7,"heart_rate_bpm":48.6,"source":"auto:esp32-vitals"}
```

The first reading after a restart may still say `auto:seed-stream` — that is the
startup probe firing before the first packet lands. From the second cycle on it
should be real.

Values that never change — a heart rate pinned at exactly `100.0`, breathing at
exactly `50.0` — are the synthetic feed, regardless of how healthy the
`overall_status` field looks.

## Troubleshooting

**`cannot bind 0.0.0.0:5006 (Address already in use (os error 98))` in cog logs**

The cog is trying to bind the data port directly instead of using the relay's
fan‑out. Almost always this means the cog was not restarted after its data source
changed — do step 3 above. On a Seed, port 5006 belonging to
`cognitum-csi-relay` is correct and expected.

**Cog reports plausible‑looking but constant vitals**

Check `source` (above). If it reads `auto:seed-stream`, you are seeing demo data.

**Frames arrive at the Seed but the cog sees nothing**

Confirm the node is emitting the packet type your cog consumes, not just raw CSI.
Run this on the Seed for a minute:

```bash
sudo tcpdump -i wlan0 -nn 'udp port 5006' -c 300 | awk '{print $NF}' | sort | uniq -c
```

Expect many 276‑byte frames plus roughly one 32‑byte frame per second. If the
32‑byte frames are absent, check that `--edge-tier` is not `0`.

## Which cogs use the CSI stream

The stream feeds any cog calling `cog_sensor_sources::fetch_sensors()`. Whether a
cog "uses ruview" is how it interprets the payload — for the feature packet
(`0xC5110003`) that is 8 little‑endian f32 features
(see `RUVIEW-CAPABILITY-MATRIX`). Documented integration:

- **Required (won't run without CSI):** `package-detect`, `parking-occupancy`, `ppe-compliance` (needs `ruview-densepose` upstream).
- **Dedicated CSI cogs:** `ruview-densepose` (CSI → 17‑keypoint skeleton), `health-monitor` (vitals/presence/apnea).
- **Optional (`--ruview-mode`):** `fall-detect`, `gunshot-detect`, `slip-fall-zone`, `smoke-fire`.
- **Not CSI (audio/other):** cough-detect, baby-cry, snore-monitor, glass-break, water-leak, frost-warning, beehive-monitor, predictive-maintenance.

The node is cog‑agnostic — capability is decided by which cog you deploy and where.

## Known gaps

- **Changing a cog's data source does not restart the cog.** Until this is fixed
  you must stop and start it manually (step 3 above), or the cog silently keeps
  serving demo data while every API call reports success.
- **Cogs cannot yet consume the `0xC5110006` feature‑state packet.** The firmware
  emits it at ~1 Hz as its richest per‑node summary, but Seed‑side cogs currently
  ignore it. Use the vitals packet (`0xC5110002`) meanwhile.
- **Release asset names are inconsistent between tags** (`esp32-csi-node-s3-8mb.bin`
  vs `esp32-csi-node-v0.8.4-8mb.bin`, bundled zip vs loose files), and
  `SHA256SUMS.txt` is not published on every release.
- The V0 `/edge` "Download RuView firmware" link / registration default points at
  an old release below the CSI minimum. Pick the latest `v0.8.x-esp32` instead.
- The mapping between the V0 `/edge` PSK (device‑id‑bound, for
  `/api/v1/v0/swarm/esp32/*`) and `provision.py --seed-token` (a Seed pairing
  bearer token) is undocumented. See issue #30.
