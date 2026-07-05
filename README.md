# WaveFlow plugins

The curated plugin registry for [WaveFlow](https://github.com/InstaZDLL/WaveFlow) — a local music player built on Tauri 2 + a Rust audio engine.

WaveFlow plugins are **sandboxed WebAssembly components**. Unlike native-library plugin models, a WaveFlow plugin is:

- **One portable `plugin.wasm`** — the same file runs on Windows, macOS, Linux, and the server. No per-platform build matrix.
- **Sandboxed** — each plugin runs in a wasmtime sandbox with a fuel/epoch/memory budget and a manifest-declared HTTP allowlist that the host **enforces** (a plugin can only reach the hosts it asked for).
- **Permissioned** — the store shows exactly what a plugin can do (which hosts it talks to, whether it persists state) *before* you install it.

## How this repo works

[`registry.json`](registry.json) is the catalogue the WaveFlow app fetches. Each entry pins a plugin's `version` **and** the `blake3` of its `plugin.wasm`. The app downloads the pinned release from the plugin's own repo, verifies the hash against this registry, and only then loads it.

That makes **this repo the trusted source of truth**, not the individual releases:

- Publishing an update = one PR here bumping `version` + `blake3` (the curation gate).
- Taking a plugin down = one commit removing its entry. It vanishes from the store and stops updating for everyone.

The app reads the registry from:

```
https://raw.githubusercontent.com/InstaZDLL/waveflow-plugins/main/registry.json
# fallback:
https://cdn.jsdelivr.net/gh/InstaZDLL/waveflow-plugins@main/registry.json
```

## Publishing a plugin

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full walkthrough. In short:

1. Build your plugin as a WIT component (`cargo component build --release`) targeting one of the WaveFlow worlds (`waveflow:source`, `waveflow:metadata`, `waveflow:ui`).
2. Tag a release in **your** repo; the [CI template](templates/plugin-release.yml) zips `manifest.toml` + `plugin.wasm` into `<id>-v<version>.zip`, computes the blake3, and attaches both to the GitHub Release.
3. Open a PR here adding your entry to `registry.json` (validated against [the schema](schema/registry.schema.json) in CI).

## The registry schema

Every entry is validated against [`schema/registry.schema.json`](schema/registry.schema.json). Example:

```json
{
  "id": "apple-artwork",
  "name": "Apple Motion Artwork",
  "description": "Fetches Apple Music's animated album covers and plays them behind the now-playing view.",
  "author": "InstaZDLL",
  "repo": "InstaZDLL/waveflow-plugin-apple-artwork",
  "homepage": "https://github.com/InstaZDLL/waveflow-plugin-apple-artwork",
  "world": "waveflow:metadata/v1",
  "version": "1.0.0",
  "blake3": "0000000000000000000000000000000000000000000000000000000000000000",
  "permissions": {
    "http": ["music.apple.com", "amp-api.music.apple.com", "itunes.apple.com", "mvod.itunes.apple.com"],
    "storage_read": false,
    "storage_state": true
  },
  "tags": ["artwork", "apple-music", "metadata"],
  "official": true,
  "min_app_version": "1.7.0"
}
```

## Trust & safety

- **Sandbox first.** A plugin can never touch the filesystem or reach a host it didn't declare. That's what makes a community store safe here where a native-plugin store wouldn't be.
- **Hash-pinned.** The app verifies `blake3` before loading. A tampered or corrupt download is refused.
- **Curated.** Entries land by PR review. `official` plugins are maintained by the WaveFlow team; everything else is community-contributed and badged as such.

## License

Registry tooling and schema: [MIT](LICENSE). Each listed plugin carries its own license in its own repo.
