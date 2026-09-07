# Porting Proxy-Based Dependency Injection to Rust's Type System

When I set out to port [Cordis](https://github.com/cordisjs/cordis) — the plugin framework behind the [Koishi](https://koishi.chat) chatbot ecosystem — to Rust, I expected the hard parts to be async runtime integration or file watching. I was wrong. The hardest part was something more fundamental: **Cordis is built on JavaScript `Proxy`, and Rust has nothing like it.**

This is the story of that translation — what a Proxy actually does for a DI framework, why none of the obvious Rust equivalents work, and what I ended up building instead: [cordis-di](https://github.com/Ricardo-M-L/cordis-di), a typed plugin and dependency-injection framework where every runtime lookup becomes a compile-time guarantee.

## What the Proxy buys you

In Cordis, a plugin receives a `Context`. That context is the service locator, the event bus, the config source, and the lifecycle owner, all at once. It works because JS engines let a `Proxy` intercept property access:

```js
// Somewhere inside cordis
ctx.database.query(...)   // resolved at runtime, lazily
ctx.timer.setInterval(...) // same
```

No trait bounds. No generic parameters. The property *is* the lookup. When a plugin accesses `ctx.database`, the Proxy checks the runtime registry, finds the service registered by another plugin, and returns it. Misspelled? Undefined until it explodes at runtime. Circular? Detected at runtime. Not yet started? Also runtime.

This is not a criticism — in a dynamic language, this is a *great* deal. You get zero-ceremony wiring, lazy resolution, and dynamic service replacement, all in one primitive. Koishi's entire plugin ecosystem is built on it.

But porting it means deciding what each of those runtime behaviors should become when the compiler is willing to help.

## Attempt 1: trait objects (the obvious translation)

The first idea everyone reaches for: services are `Box<dyn Service>`, the context holds `HashMap<TypeId, Box<dyn Any>>`, lookup is `downcast`. This *works*, and it preserves the dynamic flavor.

But look at what you paid. Every property access that JS resolved implicitly now needs:

1. A `HashMap` lookup,
2. A `downcast` that can fail,
3. An error path for "service not found" — which in the JS version was just `undefined`.

You have rebuilt `Proxy` by hand, with more syntax and the same runtime failure modes. The compiler learned nothing. If you're going to write Rust and push every failure to runtime anyway, you've paid the borrow-checker tax and bought nothing with it.

## Attempt 2: generics everywhere (the pendulum swing)

The opposite extreme: `Context<D: Database>`, `Context<D: Database, T: Timer>`... The types now encode exactly which services a plugin needs, and a plugin that misspells a dependency doesn't compile.

This fails for a different reason: **service availability in a plugin framework is inherently a property of the runtime, not the type.** A plugin may load before its dependency and start after it. The user may disable a plugin and its dependents in a tree. No static type can express "this will be present *by the time this callback fires*", because that depends on load order, config, and user choices at 2 a.m.

Pushed to this extreme, generics give you a type-level reimplementation of the load order — usually as a tuple of every service in the app, infested through every function signature.

## What actually survived: make lifecycles typed, keep resolution explicit

The design that worked inverts the question. Instead of asking *"how do I make service lookup type-safe?"*, ask *"what does the framework actually guarantee, and can the type system enforce **that**?"*

Cordis' real guarantees are about **lifecycle**: effects registered on a context are cleaned up when the plugin goes away; services registered by a plugin disappear with it; reloads don't corrupt state. Those are exactly the things Rust is best at expressing — through ownership.

So in cordis-di:

- **Fiber** owns every effect. A plugin runs inside a fiber-scoped lifecycle with validated states and RAII handles; when the fiber drops, disposers run in LIFO order. A timer can't outlive its plugin, because the handle *is* the lifetime.
- **Service lookup is explicit and name-scoped**, not inferred from property syntax. Plugins and services live in isolated registry slots with named dependency checks and duplicate protection — the things Cordis checks at runtime, checked at *registration* time instead, which is the earliest point the information exists.
- **What stays dynamic, stays honest.** If a service isn't registered, you get a typed error with the dependency's name in it — at the boundary, once, instead of an `undefined` deref deep inside a callback.

The result reads like Rust, not like translated JavaScript:

```rust
use cordis_di_core::{disposer, Fiber};
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;

let fiber = Fiber::new();
let cleaned = Arc::new(AtomicBool::new(false));
let flag = Arc::clone(&cleaned);
let _effect = fiber.effect(move || disposer(move || {
    flag.store(true, Ordering::SeqCst);
}));

fiber.dispose(); // disposers run, LIFO
assert!(cleaned.load(Ordering::SeqCst));
```

## The second Proxy: dynamic `import()` and hot reload

The same dilemma recurs at module granularity. Cordis reloads a plugin by re-importing its module and re-running its entry function. In Rust, statically linked code cannot be unloaded — period. Any port that pretends otherwise is lying with `dlopen`.

The honest translation separates *what the framework can do* (observe files, compute the reload graph, stage the swap) from *what only the application can do* (provide the new code):

- `cordis-di-hmr` watches the filesystem and emits `Changed`, `Removed`, and transitive `Reload` events — with a bounded queue, because a slow consumer must never stall the watcher thread.
- The application registers module factories with `cordis-di-loader` and maps a reload event to `Loader::reload()`.
- `Loader::reload()` **stages a new fiber first**. If anything in the new plugin's startup fails, every effect the attempt registered is rolled back, and the old runtime keeps running. A failed reload cannot corrupt the app — a property even the JavaScript version doesn't give you for free.

## A post-mortem worth keeping: the counter that wrapped around

One bug from this project is a nice illustration of the domain. The HMR worker consumes events from a bounded channel and decrements a shared queue-depth counter on `recv`. The producer side originally did:

```rust
match sender.try_send(event) {
    Ok(()) => {
        let depth = queue_len.fetch_add(1, AcqRel) + 1;  // after the send
        ...
    }
    ...
}
```

On a fast consumer, the worker could `recv()` and `fetch_sub` *between* the `try_send` and the `fetch_add`. The counter wrapped below zero to `usize::MAX`, and the next `+1` overflowed — a panic in debug builds, silent corruption in release. It only fired under CI load on shared macOS runners; it passed every local run.

The fix was ordering, not locking: reserve the slot (`fetch_add`) *before* `try_send`, roll back on the `Full`/`Disconnected` paths. The invariant becomes "the counter only ever describes slots that were reserved before their event existed," which the consumer cannot race against.

Backpressure systems fail exactly this way in production: not loudly, but in the gap between two operations you assumed were atomic. If you're building a bounded queue, draw the timeline before you trust the test.

## What I'd tell the next person porting a dynamic-language framework

1. **Don't port the mechanism; port the guarantees.** `Proxy` isn't the product — lazy, safe wiring is. Ownership gives you stronger versions of the safety guarantees if you stop chasing the syntax.
2. **Find the earliest point the information exists.** Cordis checks service wiring at property access (the latest possible point). Registration time is earlier; type time is earliest — but not everything can move all the way left, and forcing it produces generic-soup.
3. **Separate observation from action in hot-reload designs.** A Rust framework can watch, debounce, and stage; it cannot swap code. APIs that pretend otherwise push `unsafe` into user hands.
4. **Load-test your CI on someone else's slow machine.** Every timing bug I shipped passed locally. The macOS CI runner found in three runs what my laptop couldn't in a hundred.

---

[cordis-di](https://github.com/Ricardo-M-L/cordis-di) is MIT-licensed, runs CI on three operating systems, and is early — I'm actively interested in holes being poked in the lifecycle model. If you've ported a dynamic-language framework to Rust, I'd love to compare scars.
