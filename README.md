# cordis-di

A typed plugin & dependency-injection framework for Rust: fiber-scoped lifecycles, hierarchical contexts, an event bus with multiple dispatch modes, and file-backed reload signals.

Inspired by [Cordis](https://github.com/cordisjs/cordis), the application framework behind the [Koishi](https://koishi.chat) ecosystem — rebuilt around Rust's type system instead of JavaScript proxies.

[![CI](https://github.com/Ricardo-M-L/cordis-di/actions/workflows/ci.yml/badge.svg)](https://github.com/Ricardo-M-L/cordis-di/actions/workflows/ci.yml)
[![crates.io](https://img.shields.io/crates/v/cordis-di-core.svg)](https://crates.io/crates/cordis-di-core)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Rust 1.85+](https://img.shields.io/badge/Rust-1.85%2B-orange.svg)](https://www.rust-lang.org)

> **On the name** — the `cordis` / `cordis-*` crate names on crates.io belong to the upstream Cordis project, so this framework publishes as `cordis-di-*`. An independent Rust port also exists ([dshbox/cordis-rs](https://github.com/dshbox/cordis-rs)); the two projects are unrelated.

## Why

Rust has plenty of DI containers and plenty of plugin systems, but few frameworks that treat **plugin lifecycle as a first-class, deterministic contract**:

- **Fiber** — every plugin runs inside a validated lifecycle scope with RAII handles and LIFO cleanup. When a scope drops, every effect it registered is disposed — no leaks, no orphaned timers.
- **Typed contexts** — hierarchical scopes with per-service isolation replace JS proxy magic. Service lookup is explicit and checked at compile time.
- **Staged reload** — `Loader::reload()` stages a new fiber, and on failure rolls back all of its effects before retaining the old runtime. A failed reload cannot corrupt the running app.
- **Production hardening** — bounded queues with backpressure accounting, panic-safe callbacks, hardened config loading (size/depth limits, strict paths).

## Quick start

```bash
cargo add cordis-di-core
```

```rust
use cordis_di_core::{disposer, Fiber};
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;

let fiber = Fiber::new();
let cleaned = Arc::new(AtomicBool::new(false));
let cleanup_flag = Arc::clone(&cleaned);
let _effect = fiber.effect(move || disposer(move || {
    cleanup_flag.store(true, Ordering::SeqCst);
}));

fiber.dispose(); // disposers run in LIFO order
assert!(cleaned.load(Ordering::SeqCst));
```

Scaffold a full project:

```bash
cargo install cordis-di-create
cordis-di-create my-app --git
```

## How it compares to Cordis (TypeScript)

| | cordis (TS) | cordis-di |
|---|---|---|
| Service lookup | runtime `Proxy` | explicit types + registered factories |
| Module loading | dynamic `import()` | statically registered module factories |
| Plugin lifecycle | ctx effects | Fiber: validated states, RAII, LIFO cleanup |
| Reload | process/HMR-dependent | staged fiber reload with rollback on failure |
| Config | JS objects | JSON/YAML/TOML with safe patches and bounds |
| Hot reload | JS module swap | fs watcher + reload events (app maps them to factories) |

Not a drop-in API-compatible port — see the boundary notes below for exactly what differs and why.

## Implemented behavior

- **Fiber**: validated states, typed effect results, synchronous/asynchronous factories, RAII handles, and LIFO cleanup.
- **Context**: typed hierarchical scopes, per-service isolation, layered configuration intercepts, and explicit runtime binding.
- **Events**: scoped synchronous/asynchronous listeners, context filtering, Fiber-owned cleanup, serial/bail/waterfall dispatch, and concurrent parallel dispatch.
- **Registry**: isolated plugin/service slots, Service init/check integration, named dependency checks, duplicate protection, and staged replacement.
- **Logger**: `%s`, `%d`, `%o` formatting, Unicode-safe truncation, bounded message history, and reentrant exporters.
- **Timer**: stoppable timeout/interval streams and reusable debounce/throttle handles.
- **Include**: JSON, YAML, and TOML file loading plus safe object/array patches, bounded file/patch limits, and optional strict path behavior.
- **Loader**: entry-tree activation through statically registered Rust module factories, resolved Context intercepts, scoped contexts, disabled-tree propagation, unload, staged reload, and side-effect rollback.
- **HMR**: recursive operating-system file watching, debounce, ignored paths, callbacks, bounded event queue, and transitive dependency reload events with panic-safe callback execution. The application remains responsible for mapping reload events to its module factories.
- **Create**: executable project generator with overwrite protection, built-in or Git templates, optional Git initialization, and release-profile generation.

## Workspace

| Crate | Purpose |
|---|---|
| [`cordis-di-core`](cordis-core/) | Core runtime: Fiber, Context, Events, Registry, Logger |
| [`cordis-di-timer`](cordis-timer/) | Timeout, interval, debounce, throttle |
| [`cordis-di-logger-console`](cordis-logger-console/) | ANSI console exporter |
| [`cordis-di-utils`](cordis-utils/) | Shared collections and configuration helpers |
| [`cordis-di-group`](cordis-group/) | Entry grouping |
| [`cordis-di-include`](cordis-include/) | JSON/YAML/TOML loading and patching |
| [`cordis-di-loader`](cordis-loader/) | Entry tree and registered module factories |
| [`cordis-di-hmr`](cordis-hmr/) | File watcher and reload dependency graph |
| [`cordis-di-create`](cordis-create/) | Project scaffolding library and CLI |

## Build and test

Rust 1.85 or newer is required.

```bash
cargo fmt --all -- --check
cargo check --workspace --all-targets --locked
cargo test --workspace --all-targets --locked
cargo clippy --workspace --all-targets --locked -- -D warnings
```

## Runtime integration boundary

`Loader::with_runtime()` binds a root `CordisContext` to one `RegistryService` and shared event bus. Each plugin is prepared in a Loading `Fiber`; services and listeners registered during `Plugin::apply()` remain hidden until activation and are removed with that Fiber. Per-name isolation labels key both plugins and services, while Context intercepts are resolved into the configuration passed to module factories. Reload stages a new Fiber and cleans all of its effects on failure before retaining the old runtime.

This explicit lifecycle replaces Cordis' JavaScript Proxy-based service lookup. It does not implement dynamic JavaScript module linking or offer drop-in API compatibility.

## HMR boundary

`cordis-di-hmr` performs real filesystem observation and emits `Changed`, `Removed`, and transitive `Reload` events. Rust cannot safely unload arbitrary statically linked code. Applications should register module factories with `cordis-di-loader` and call `Loader::reload()` in response to an accepted reload event.

## HMR backpressure and observability

Since callbacks can be slow, `cordis-di-hmr` uses a bounded queue (`queue_capacity`, default `1024`) between the watcher callback and worker thread. Slow consumers therefore drop events instead of blocking file-system processing.

```rust
use cordis_di_hmr::{Hmr, HmrConfig};

let hmr = Hmr::new(
    "app",
    HmrConfig {
        root: Some("./src".into()),
        base: Some("./src".into()),
        debounce: 100,
        ignored: vec![".DS_Store".into()],
        queue_capacity: 1024,
    },
);

hmr.watch().unwrap();
hmr.on_event(|event| println!("{event}"));

let stats = hmr.stats();
assert!(stats.queue_depth <= stats.queue_capacity);
```

Use `stats()` for SRE diagnostics (`total_received`, `total_emitted`, `total_dropped`, `total_errors`, `callback_panics`, and queue depth/peak).

## Include hardening

`cordis-di-include` adds security-oriented options for configuration loading:

- `max_file_bytes` bounds input size (default 1MB)
- `max_patch_depth` bounds path depth (default 64)
- `strict` mode for scalar-segment safety in intermediate path traversal

```rust
use cordis_di_include::{IncludePlugin, Patch};
use serde_json::json;
use std::collections::HashMap;

let plugin = IncludePlugin::with_options(
    "app-config",
    vec![Patch::new("server.port", json!(3000))],
    2 * 1024 * 1024,
    32,
    true,
);

let mut config: HashMap<String, serde_json::Value> = [
    ("server".to_string(), json!({"port": 80})),
].into_iter().collect();

plugin.apply_patches(&mut config).expect("apply patches");
```

## Project generator options

```bash
cordis-di-create my-cordis-app --target /tmp/my-cordis-app --git
cordis-di-create my-cordis-app --core-path ../cordis-core
cordis-di-create my-cordis-app --core-version 0.1.0
```

Without these flags, `cordis-di-create` keeps the existing behavior of using the repository git source. Existing non-empty directories are preserved unless `--force` is explicitly supplied.

## License

MIT — see [LICENSE](LICENSE).
