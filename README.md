# Cognitum Support

Community home for **Cognitum** users. Source code lives in private repos; here you'll find releases, documentation, and a place to report issues.

Website: [cognitum.one](https://cognitum.one) · Order: [cognitum.one/order](https://cognitum.one/order) · Twitter: [@Cognitum](https://twitter.com/Cognitum) · LinkedIn: [Cognitum](https://linkedin.com/company/cognitum)

## Downloads

### Cognitum Seed firmware

The latest gold-image release is always on the [Releases page](https://github.com/cognitum-one/support/releases). Each release includes:

- `cognitum-seed-vX.Y.Z.img.gz` — full SD card image (~1.3 GB)
- `upgrade-seed.sh` — one-command upgrade script for existing seeds
- SHA256 checksum

**Quick install on a fresh SD card:**

```bash
# macOS / Linux
gunzip -c cognitum-seed-vX.Y.Z.img.gz | sudo dd of=/dev/rdiskN bs=4m
```

Or use [Raspberry Pi Imager](https://www.raspberrypi.com/software/) with a custom image.

**Upgrade an existing seed (v0.8.1+):**

```bash
curl -sLO https://github.com/cognitum-one/support/releases/latest/download/upgrade-seed.sh
bash upgrade-seed.sh
```

### Drivers & utilities

- **Windows**: USB gadget support is native on Windows 10+; no driver needed.
- **macOS**: native USB ethernet; no driver needed.
- **Linux**: native; you may need to enable shared connection on the USB interface.

## Getting Started

1. **Plug** a Cognitum Seed into any USB port.
2. **Wait** 10 seconds for the device to boot.
3. **Open** `http://169.254.42.1/guide` in your browser.

See [Getting Started Guide](docs/getting-started.md) for details.

## Report an Issue

- **Firmware bug?** [Open a bug report](https://github.com/cognitum-one/support/issues/new?template=bug_report.yml)
- **Feature request?** [Suggest a feature](https://github.com/cognitum-one/support/issues/new?template=feature_request.yml)
- **Documentation typo?** [Report a doc issue](https://github.com/cognitum-one/support/issues/new?template=docs.yml)

Before opening, please search [existing issues](https://github.com/cognitum-one/support/issues?q=is%3Aissue) first.

## Documentation

- [Getting Started](docs/getting-started.md)
- [API Reference](docs/api.md)
- [Troubleshooting](docs/troubleshooting.md)
- [FAQ](docs/faq.md)

## Learn more

- [Seed product deck](https://cognitum.one/seed)
- [V0 Appliance deck](https://cognitum.one/v0)
- [Technical deck](https://cognitum.one/deck/technical)
- [Security deck](https://cognitum.one/deck/security)
- [Privacy policy](https://cognitum.one/privacy)

## License

Cognitum firmware is distributed under the terms of its EULA (see each release). Documentation in this repo is CC-BY-4.0 unless noted otherwise.
