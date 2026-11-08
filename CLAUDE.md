# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

`bevy_seedling` is a Bevy (0.20) integration of the [Firewheel](https://github.com/BillyDM/firewheel) audio engine (0.12). Firewheel 0.12 only supports Bevy 0.19 upstream, so `Cargo.toml` has a `[patch.crates-io]` pointing at locally patched copies in `../firewheel-bevy-0.20/` (see its README). It's a library crate; there is no binary — use the examples to run things.

## Commands

These mirror `.github/workflows/checks.yaml` (CI runs with `RUSTFLAGS=--deny warnings`). Linux needs `libasound2-dev libpulse-dev libudev-dev` (or distro equivalents) for cpal.

```sh
# Tests (CI uses --all-features; dev-dependencies already enable hrtf/loudness/effects/resample_inputs)
cargo test --all-features --all-targets --lib --tests --benches
cargo test --all-features --doc        # doctests run separately (cargo#6669)

cargo test --all-features <test_name>  # single test, e.g. a fn in src/pool/mod.rs `mod test`

cargo clippy --all-targets --all-features -- --deny warnings
cargo fmt --all -- --check
cargo doc --all-features --document-private-items --no-deps
cargo build --no-default-features      # must also compile
typos src seedling_macros examples benches   # CI spellcheck (crate-ci/typos)

cargo run --example one_shot           # examples load audio from assets/
cargo bench --features profiling       # benches/basic.rs requires `profiling`

# Wasm check
RUSTFLAGS='--cfg getrandom_backend="wasm_js"' cargo build --examples --target wasm32-unknown-unknown --features=ogg,bevy/bevy_asset
```

CI also runs `cargo-semver-checks` — avoid breaking public API outside of a version bump.

## Layout

- `src/` — the main crate. `seedling_macros/` provides `#[derive(NodeLabel)]` / `#[derive(PoolLabel)]`.
- `crates/firewheel-ircam-hrtf` and `crates/firewheel-web-audio` are separate crates (own `Cargo.toml`, own CI workflows triggered by path) pulled in as optional path deps via the `hrtf` and `web_audio` features. They are not part of a Cargo workspace with the root.
- The crate uses `extern crate self as bevy_seedling;` so internal macros can refer to `bevy_seedling::...`.
- Lints: `#![warn(missing_docs)]` and `#![warn(missing_debug_implementations)]` — every public item needs docs and a `Debug` impl (warnings are errors in CI).

## Architecture

**Plugins.** `SeedlingPlugins` (a `plugin_group!` in `src/lib.rs`) = `SeedlingCorePlugin` + feature-gated platform plugins (`cpal`, `rtaudio`, `web_audio`; they don't override each other, priority cpal < rtaudio < web) + `AudioDiagnosticsPlugin`. The core plugin adds the sub-plugins: context, node, edge, pool, nodes, spatial, time, and sample loaders.

**Audio context thread.** `AudioContext` (`src/context/`) wraps the `FirewheelContext`, which lives on a dedicated control thread (`context/os.rs`; `context/web.rs` for wasm). ECS code reaches it via `AudioContext::with(|ctx| ...)`, which ships a closure over a channel and blocks until it returns. Nesting `with` inside itself deadlocks.

**Initialization is two-stage** (`src/context/graph.rs`): in `PreStartup` the `AudioGraphTemplate` (default buses/pools such as `MainBus`, `SoundEffectsBus`, `MusicPool`, `SpatialPool`, `DefaultPool`) is spawned; the audio stream is started afterwards from the `AudioStreamConfig<_>` resource (platform-specific, `src/platform/`), so `Startup` systems can still configure it. Later changes to `AudioStreamConfig` restart the stream.

**Per-frame pipeline.** All runtime systems run in `Last`, ordered by `SeedlingSystems`: `Acquire` → `Connect` → `Pool` → `Queue` → `Flush` → `PollStream`.
- `Acquire`: entities with a registered node component get a `FirewheelNode(NodeID)` from the graph; configuration-component changes recreate the node.
- `Connect`: pending `Connect`/`Disconnect` edges (`src/edge/`) are applied (disconnects first).
- `Pool`: `SamplePlayer`s are assigned to samplers in pools (`src/pool/`).
- `Queue`: ECS→audio parameter sync via diffing (below), bypass, scheduled events.
- `Flush`: events are sent to the audio context.

**Nodes as ECS components.** A Firewheel `AudioNode` type is used directly as a Bevy component and registered via the `RegisterNode` trait (`src/node/mod.rs`): `register_node` (for `Diff + Patch` types) or `register_simple_node`. Registration makes `AudioNode::Configuration` a required component and inserts a hidden *baseline* copy; each frame, `Changed` nodes are diffed against the baseline and resulting patches are sent to the audio processor. `register_node_state` copies processor state (e.g. atomics on `SamplerNode`/`LoudnessNode`) back as `AudioState<S>`. Built-in registrations live in `src/nodes/mod.rs`; seedling's own nodes (send, limiter, itd, loudness) live alongside. **A custom node that isn't registered silently does nothing.**

**Samples and pools.** `SamplePlayer` (`src/sample/`) is a request to play an `AudioSample` asset; it doesn't own an audio node. A *pool* is a set of `SamplerNode`s feeding a local bus. Static pools are `SamplerPool<L: PoolLabel>` — a player tagged with the same label component plays there (otherwise `DefaultPool`). Players with `sample_effects![...]` and no label go to *dynamic pools* (`pool/dynamic.rs`), created per unique effect-chain and routed to `DynamicBus`. Effects on a player are separate entities related via `SampleEffects`/`EffectOf`; their params are diffed onto the assigned sampler's effect nodes. `pool/queue.rs` handles priority/queuing when a pool is full and pools grow within `PoolSize`.

**Time & scheduling.** `src/time.rs` exposes `Time<Audio>` (the audio clock); `AudioEvents` on a node entity schedules parameter changes at audio-clock instants (`node/events.rs`).

## Testing

Unit tests live in `#[cfg(test)] mod test` blocks inside the source files. Use `crate::test::prepare_app(startup_system)` (in `src/lib.rs`), which builds an app with `MinimalPlugins`, `SeedlingCorePlugin`, the `MockBackendPlugin` (`src/platform/mock.rs`, no real audio device), `AudioGraphTemplate::Empty`, and zero `DiffRate`, then runs one update. Drive it with `app.update()` and inspect with `crate::test::run(&mut app, system)`.
