# Troubleshooting

## Guide page blank

**Symptom:** `http://169.254.42.1/guide` loads but shows nothing.

**Fix:**
1. Wait 60 seconds after plugging in - first-boot takes time.
2. If still blank, unplug and replug.
3. Check that your computer assigned a `169.254.42.x` address to the Cognitum USB adapter.

## Cog Store says "not installed"

**Symptom:** `/store` shows "Cog Store not installed".

**Fix:** your firmware predates the embedded store. Upgrade to the latest release via `upgrade-seed.sh`.

## HTTPS certificate warning

The Seed uses a self-signed CA the first time you visit. Accept the warning (or install the CA cert from `http://169.254.42.1/ca.crt`).

## Still stuck?

Open a [bug report](https://github.com/cognitum-one/support/issues/new?template=bug_report.yml) with your firmware version and host OS.
