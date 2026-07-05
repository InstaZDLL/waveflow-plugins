# Publishing a WaveFlow plugin

This guide takes you from an idea to a listed, installable plugin.

## 1. Pick a world

A plugin exports exactly one WaveFlow **WIT world**. Choose the one that matches what you're adding:

| World | For | Key exports |
|-------|-----|-------------|
| `waveflow:source@1.0.0` | New playable content (radio, podcasts, network shares) | `list-entries`, `resolve(query)`, `stream-url(track-id)` |
| `waveflow:metadata@1.0.0` | Enriching existing library rows (bios, similar artists, lyrics, artwork) | `artist-info`, `album-info`, `lyrics` |
| `waveflow:ui@1.0.0` | Custom views (reserved — host binding lands in a later app version) | `render` |

The WIT definitions live in the app repo under [`src-tauri/crates/plugin-sdk/wit/`](https://github.com/InstaZDLL/WaveFlow/tree/main/src-tauri/crates/plugin-sdk/wit). Your host imports are `waveflow:host/{http,log,storage}` — nothing else is reachable.

## 2. Build the component

A plugin is a WebAssembly **Component** built with [`cargo component`](https://github.com/bytecodealliance/cargo-component):

```bash
cargo component build --release
# -> target/wasm32-wasip1/release/<crate>.wasm
```

Rename (or configure) the output to `plugin.wasm`. There is **no per-platform matrix** — one `.wasm` is all platforms.

## 3. Write the manifest

`manifest.toml` sits next to `plugin.wasm`. The `id`, `world`, and `[permissions]` **must** match what you'll declare in the registry entry — the host refuses a mismatch.

```toml
schema_version = 1

[plugin]
id = "apple-artwork"
name = "Apple Motion Artwork"
version = "1.0.0"
author = "InstaZDLL"
world = "waveflow:metadata@1.0.0"
description = "Fetches Apple Music's animated album covers."
homepage = "https://github.com/InstaZDLL/waveflow-plugin-apple-artwork"
license = "MIT"

[permissions]
# Only these hosts are reachable. The sandbox blocks everything else,
# including redirects to non-allowlisted hosts.
http = ["music.apple.com", "amp-api.music.apple.com", "itunes.apple.com", "mvod.itunes.apple.com"]
storage_read = false
storage_state = true
```

Declare the **narrowest** HTTP allowlist that works — it's shown to the user before install and enforced at runtime.

## 4. Release it

Tag a semver release in your plugin repo. Copy [`templates/plugin-release.yml`](templates/plugin-release.yml) into `.github/workflows/` — it is a **single job** (no OS matrix) that:

1. builds the component,
2. packages `manifest.toml` + `plugin.wasm` (+ optional `assets/`) into `<id>-v<version>.zip`,
3. computes the blake3 of `plugin.wasm` and prints it to the job log + a `plugin.wasm.blake3` asset,
4. attaches the zip to the GitHub Release.

## 5. List it here

Open a PR against this repo adding your entry to [`registry.json`](registry.json):

- `version` = the release tag (without `v`).
- `blake3` = the hash the release workflow printed. **This is what the app verifies.** Get it wrong and installs fail closed.
- `permissions` = a byte-for-byte mirror of your `manifest.toml` `[permissions]`.

CI validates your entry against [the schema](schema/registry.schema.json). Once merged, your plugin appears in every user's in-app store within minutes (raw.githubusercontent + jsDelivr are the delivery paths).

## Updating

Bump `version` + `blake3` in a new PR here. The app compares the installed version against the registry and offers the update. Because the registry is the trusted pin, an attacker who compromises a plugin's releases still can't push a payload the app will load — the hash won't match this repo.

## Removing / taking down

Delete the entry in a PR. The plugin disappears from the store and stops receiving updates. Already-installed copies keep working until the user uninstalls (the app can't reach into a machine and delete files), but nothing new installs it.
