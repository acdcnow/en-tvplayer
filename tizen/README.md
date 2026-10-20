# Tizen OS Build — EN IPTV Player

> See the [main README](../../README.md) for the full project overview and quick start.

Packages the IPTV web app into a `.wgt` package installable on Samsung Tizen TVs (4.0+).

## Prerequisites

1. **Node.js** — Already have it (used for the main project)
2. **Python 3** — Required for the zip ordering helper
3. **OpenSSL** — Used for code signing (bundled with Git for Windows, or install via `winget install OpenSSL.OpenSSL`)
4. **Samsung Developer Mode app** — Install from Samsung Smart Hub on your TV
   - Go to **Apps** → Search **"Developer Mode"** → Install and enable it
   - Note the IP address shown on TV

## Quick Start

From the project root:

```bash
npm run tizen
```

This runs `npm build` and packages/signs the `.wgt` in one step.

## Step-by-Step

### 1. Build the Web App

```bash
npm run build
```

Outputs the production build to `packages/player/dist/`.

### 2. Package the Tizen .wgt

```bash
node packages/tizen/package.mjs
```

This:
1. Auto-generates a self-signed certificate (`author-key.pem` / `author-cert.pem`) if absent
2. Copies `dist/` into a temp directory with `config.xml` and icons
3. Signs it with your certificate
4. Outputs to the repo root: `EN-IPTV_Player_{stable|beta}_{version}_{commit}.wgt`

### 3. Install on TV

```bash
node packages/tizen/install.mjs --ip=<TV_IP_ADDRESS>
```

Make sure Developer Mode is running on your TV.

## Build Output

| Branch | Output Example |
|---|---|
| `main` | `EN-IPTV_Player_stable_0.8.0_abc1234.wgt` |
| Any other | `EN-IPTV_Player_beta_0.8.0_abc1234.wgt` |

### Certificate & Signing Note

**Verified on hardware (Samsung UE40NU7190, Tizen 4.0) — two distinct install errors:**

| Code | Stage | Meaning |
|---|---|---|
| `install failed[118019]` | immediate | the widget's `required_version` is **higher than the TV's `platform_version`**. A package declaring `5.0` on a `4.0` TV fails here before anything is unpacked. |
| `install failed[118012]` | after `installing[23]` | the TV rejected the **signature/certificate** of an otherwise valid package. |

The first one is why this fork declares `required_version="4.0"` (override with
`TIZEN_REQUIRED_VERSION=5.0`). The second one is not solved by the version bump: the
self-signed `signature.xml` produced by `package.mjs` is accepted on newer TVs but
**rejected on Tizen 4.0 sets**. This is the open issue described in `docs/BUGS.md`
BUG-003, and it is not specific to this app — a 2018 Tizen 4.0 TV also fails with
`install failed[118012]` at `installing[23]` for a minimal Hello-World app *and* for a
package signed with a valid Samsung Public certificate containing the device DUID
(Samsung Developer Forum topic 10230, June 2026, unresolved).

What is known to work on other Tizen 4.0 sets is the **Tizen-native signing flow**: an
author certificate issued by the *Tizen Developers CA* (with code-signing usage and the
issuer chain embedded in the `.wgt`, as Jellyfin-Tizen ships it) plus
`author-signature.xml`/`signature1.xml`, instead of a single PKCS#7 `signature.xml`.
See `MrHumanRebel/airtizen` → `docs/TV_INSTALL_TROUBLESHOOTING.md` for that analysis.

### Old note (kept for context)

> **Current packaging uses a single self-signed `signature.xml`.** Samsung's official flow via **Tizen Studio Certificate Manager** creates a Samsung-issued author certificate (tied to your TV's DUID) and produces three signatures (`author-signature.xml`, `signature1.xml`, `signature2.xml`). The current single-signature WGT has not been verified on hardware and **may fail to install on some TVs** — see `docs/BUGS.md` BUG-003. To verify:

1. Build a WGT with `npm run tizen` on `dev` (now at `stable/`).
2. Enable Developer Mode on a Tizen 4.0+ TV and note its DUID (shown in Developer Mode).
3. Install via `node packages/tizen/install.mjs --ip=<TV_IP>` or Tizen Studio's Device Manager. If you see "Invalid certificate" or "Signature verification failed", generate a Samsung certificate via Tizen Studio (Certificate Manager → Samsung → DUID) and replace `author-*.pem` (see `SECURITY.md`), then rebuild.

For local development the self-signed flow is sufficient; for distribution, use the Samsung flow.

## File Structure

```
packages/tizen/
├── README.md                 # This file
├── config.xml                # Tizen web app manifest (reference)
├── package.mjs               # Build & sign the .wgt
├── install.mjs               # Install .wgt to TV via Developer Mode
├── ziphelper.py              # Python helper for Tizen-compliant zip ordering
├── icons/
│   └── icon_128.png          # App icon (128x128)
├── author-key.pem            # Private key (auto-generated, gitignored)
├── author-cert.pem           # Certificate (auto-generated, gitignored)
└── temp-wgt/                 # Temporary build directory (gitignored)
```

## Troubleshooting

| Problem | Fix |
|---|---|
| TV says "Invalid certificate" | Delete `author-key.pem` and `author-cert.pem`, re-run `package.mjs` to regenerate |
| App doesn't appear after install | Restart TV, check **Apps** → **My Apps** again |
| Video won't play | Make sure the CORS proxy (`npm run proxy`) is running on your PC |
| Install fails | Ensure TV and PC are on the same network, Developer Mode is enabled |
| OpenSSL not found | Install via `winget install OpenSSL.OpenSSL` or use Git for Windows which bundles it |
| Python not found | Install Python 3 and ensure it's on your PATH |
