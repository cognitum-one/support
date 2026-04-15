# Getting Started

## What you need

- 1x Cognitum Seed
- 1x USB-A or USB-C port on a computer (Windows 10+, macOS, or Linux)
- A web browser

## Step 1 - Plug it in

Plug the Seed into any USB port. The activity LED should blink for 5-10 seconds as it boots.

## Step 2 - Connect

The Seed presents itself as a network adapter with a static address. Open your browser to:

```
http://169.254.42.1/guide
```

This walks you through pairing the Seed with your local account.

## Step 3 - Install a cog

Open `http://169.254.42.1/store` to browse the cog library. Cogs are small self-contained apps (Rust/WASM) that give the Seed new capabilities - sensor readers, ML models, communication bridges.

Click any cog to install. Installation takes a few seconds.

## Step 4 - Connect to other seeds (optional)

If you have more than one Seed, they can form a local WiFi mesh. Open `http://169.254.42.1/mesh` to enable.

## Troubleshooting

- **Guide page does not load** -> see [troubleshooting](troubleshooting.md)
- **Seed not detected on Windows** -> set firewall to Private for the USB adapter
- **Store shows "not installed"** -> your firmware is out of date; [upgrade](../README.md#upgrade-an-existing-seed)
