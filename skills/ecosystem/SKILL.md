---
name: gleam-ecosystem
description:
  Gleam ecosystem knowledge — notable stdlib and third-party packages, and
  community resources (Discord, docs, package index)
user-invocable: false
---

# Gleam Ecosystem

## LSP / editor setup

The language server ships with Gleam — no separate install needed (`gleam lsp`).
Supported editors: VS Code (Gleam extension), Neovim (`nvim-lspconfig gleam`),
Zed (built-in), Helix (built-in).

Search and browse packages at https://packages.gleam.run/.

## Notable stdlib & ecosystem packages

| Package            | Purpose                                                                |
| ------------------ | ---------------------------------------------------------------------- |
| `gleam_stdlib`     | Core — `list`, `dict`, `result`, `option`, `string`, `int`, `float`, … |
| `gleeunit`         | Test runner (wraps EUnit / Node test)                                  |
| `gleam_http`       | Platform-agnostic HTTP request/response types                          |
| `gleam_erlang`     | Erlang-target helpers (processes, atoms, file I/O)                     |
| `gleam_javascript` | JS-target helpers — Promise, Array, Symbol (see below)                 |
| `gleam_otp`        | Actors, supervisors, tasks on Erlang/OTP                               |
| `argv`             | CLI argument access                                                    |
| `envoy`            | Environment variable access                                            |
| `gleescript`       | Produce self-contained escript binaries                                |

## gleam_javascript

Install: `gleam add gleam_javascript@1`. Provides JS-specific types not in
stdlib (which must stay cross-platform).

### `gleam/javascript/promise`

```gleam
import gleam/javascript/promise.{type Promise}

promise.resolve(value)                     // -> Promise(a)
promise.new(fn(resolve) { resolve(x) })   // -> Promise(a)

promise.await(p, fn(v) { ... })           // -> Promise(b)  (like .then)
promise.map(p, fn(v) { ... })             // -> Promise(b)  (sync transform)
promise.tap(p, fn(v) { ... })             // -> Promise(a)  (side-effect)

promise.map_try(p, fn(v) { ... })         // Promise(Result(a,e)) -> Promise(Result(b,e))
promise.try_await(p, fn(v) { ... })       // Promise(Result(a,e)) -> Promise(Result(b,e))

promise.rescue(p, fn(err: Dynamic) { v }) // -> Promise(a)  (error recovery)

promise.await_list(list_of_promises)      // -> Promise(List(a))  (like Promise.all)
promise.race_list(list_of_promises)       // -> Promise(a)        (first to resolve)
promise.wait(delay_ms: Int)              // -> Promise(Nil)
```

`Promise(a)` is not generic over errors — use `Promise(Result(a, e))` for
fallible async.

Typical pattern with `use`:

```gleam
pub fn load() -> Promise(Result(String, Err)) {
  use response <- promise.try_await(fetch_something())
  use text <- promise.map_try(parse(response))
  Ok(text)
}
```

### `gleam/javascript/array`

```gleam
import gleam/javascript/array.{type Array}

array.from_list([1, 2, 3])         // -> Array(a)
array.to_list(arr)                  // -> List(a)
array.size(arr)                     // -> Int           O(1)
array.get(arr, index)               // -> Result(a, Nil)
array.map(arr, fn(x) { x * 2 })    // -> Array(b)
array.fold(arr, 0, fn(acc, x) { acc + x })  // -> b
```

Prefer `Array` over `List` when interfacing with JS APIs — convert at the
boundary.

### `gleam/javascript/symbol`

```gleam
import gleam/javascript/symbol.{type Symbol}

symbol.new("tag")                   // local symbol
symbol.get_or_create_global("key") // global registry (like Symbol.for)
symbol.description(sym)             // -> Result(String, Nil)
```

---

## SBoM

To generate a Source Bill of Materials:
https://gleam.run/documentation/source-bill-of-materials/

## Community & resources

- Language tour: https://tour.gleam.run/
- Package index: https://packages.gleam.run/
- Stdlib docs: https://hexdocs.pm/gleam_stdlib/
- Discord: https://discord.gg/Fm8Pwmy
- GitHub: https://github.com/gleam-lang/gleam
