---
name: gleam-ffi
description: >-
  Gleam FFI (Foreign Function Interface) — @external syntax, Erlang FFI (module
  naming, file structure, type mapping), JavaScript/TypeScript FFI, multi-target
  functions, opaque handle types, @target attribute, module name mangling,
  gleam_erlang stdlib helpers, atoms, charlists, Dynamic/decode
user-invocable: false
---

# Gleam FFI

Gleam interops with Erlang and JavaScript using the `@external` attribute.

---

## `@external` syntax

```gleam
@external(erlang, "module_name", "function_name")
pub fn my_function(arg: SomeType) -> ReturnType
```

- First argument: target — `erlang` or `javascript`
- Second argument: the native module name (string)
- Third argument: the native function name (string)
- Type annotations are **mandatory** — the compiler cannot infer types from
  non-Gleam code; omitting them is a compile error
- The compiler trusts your annotation; an incorrect type causes **runtime
  crashes**, not compile errors
- The Gleam function has **no body** when it's fully external
- If it **has a body**, the body acts as the fallback for the other target (see
  below)

Since Gleam v1.14 (2025), `@external` is also supported on **external types** to
link them to an Erlang or TypeScript type definition:

```gleam
@external(erlang, "erlang", "map")
@external(javascript, "../dict.d.mts", "Dict")
pub type Dict(key, value)
```

---

## Erlang FFI

### Writing the Erlang module

Create a `.erl` file in `src/` alongside your Gleam source. The Gleam compiler
automatically compiles and includes it — no separate build step needed.

**Convention for FFI helper files:** name them `<package>_ffi.erl` (e.g.
`my_app_ffi.erl`). This is the pattern used in the official `gleam_erlang`
package (`gleam_erlang_ffi.erl`). Any valid Erlang module name works.

```erlang
-module(my_app_ffi).
-export([greet/1, add/2]).

greet(Name) ->
    <<"Hello, ", Name/binary>>.

add(A, B) ->
    A + B.
```

Then reference it from Gleam:

```gleam
@external(erlang, "my_app_ffi", "greet")
pub fn greet(name: String) -> String

@external(erlang, "my_app_ffi", "add")
pub fn add(a: Int, b: Int) -> Int
```

### Calling Erlang stdlib directly

You can reference any Erlang module — no FFI file needed:

```gleam
@external(erlang, "erlang", "unique_integer")
pub fn unique_id() -> Int

@external(erlang, "lists", "reverse")
pub fn reverse_list(list: List(a)) -> List(a)

@external(erlang, "crypto", "strong_rand_bytes")
pub fn random_bytes(n: Int) -> BitArray

@external(erlang, "persistent_term", "get")
pub fn all_terms() -> List(#(Dynamic, Dynamic))
```

### Elixir FFI (variant of Erlang target)

Elixir modules require the `"Elixir."` prefix:

```gleam
@external(erlang, "Elixir.RandomColor", "hex")
pub fn random_color() -> String
```

Custom Elixir files (`src/mylib.ex`) are auto-compiled; reference them as
`"Elixir.Mylib"`.

**Limitation:** Elixir macros are compile-time only and cannot be called from
Gleam.

### Module name mangling — how Gleam maps to Erlang atoms

When the Gleam compiler emits Erlang, the module path separator `/` becomes `@`:

| Gleam module path        | Erlang atom              |
| ------------------------ | ------------------------ |
| `my_project`             | `my_project`             |
| `my_project/application` | `my_project@application` |
| `gleam/erlang/node`      | `gleam@erlang@node`      |

Consequences:

- **Do not name Gleam modules the same as Erlang stdlib modules** (`lists`,
  `erlang`, `maps`, `io`, etc.) — the compiled output would clash.
- When calling Gleam from Erlang, use the `@`-separated atom:
  `'my_project@application':my_function(Args)`.
- Raw FFI helper modules (e.g. `my_app_ffi.erl`) use flat names — they are not
  Gleam modules, so no `@` appears.

**Function name mangling:** Gleam function names in compiled Erlang are
identical to their Gleam names (snake_case). No transformation is applied to
function names.

**Custom type variants:** PascalCase Gleam variants compile to snake_case Erlang
atoms/tuples:

- `SuperUser(id: 11)` → `{super_user, 11}`
- `AtomVariant` (no fields) → atom `atom_variant`

### Type mapping: Gleam ↔ Erlang

| Gleam type              | Erlang representation             |
| ----------------------- | --------------------------------- |
| `String`                | UTF-8 binary (`<<"hello"/utf8>>`) |
| `Int`                   | integer                           |
| `Float`                 | float                             |
| `Bool` (`True`/`False`) | atom `true` / `false`             |
| `Nil`                   | atom `nil`                        |
| `BitArray`              | bitstring (`<<1, 2, 3>>`)         |
| `List(a)`               | proper list                       |
| `#(a, b)` (tuple)       | tuple `{A, B}`                    |
| `Result(a, b)`          | `{ok, A}` / `{error, B}`          |
| `Option(a)`             | `{some, A}` / `none`              |
| Custom record type      | tuple `{variant_name, field1, …}` |
| Custom atom variant     | atom `variant_name`               |
| `Dict(k, v)`            | Erlang map                        |
| Opaque external type    | any Erlang term                   |

**Critical:** Gleam strings are **binary strings**, NOT charlists. Old Erlang
APIs that expect or return charlists require explicit conversion via
`gleam/erlang/charlist`.

Gleam automatically converts `{ok, V}` / `{error, E}` to/from `Result`. Your
Erlang FFI functions should return these tuples.

### Erlang records and headers

For Erlang libraries that use records (`#amqp_params_network{}`), include the
header in the FFI file:

```erlang
-module(my_ffi).
-include_lib("some_lib/include/some_lib.hrl").
-export([connect/1]).

connect(Config) ->
    amqp_connection:start(#amqp_params_network{
        host = Config#config.host,
        port = Config#config.port
    }).
```

### Error handling in Erlang FFI

Wrap Erlang exceptions so Gleam sees `Result`:

```erlang
safe_call(Arg) ->
    try some_lib:call(Arg) of
        {ok, Result} -> {ok, Result};
        _ -> {error, nil}
    catch
        throw:Term -> {error, Term};
        exit:Reason -> {error, Reason};
        error:Reason:_ -> {error, Reason}
    end.
```

---

## JavaScript / TypeScript FFI

### Writing the JS/TS module

Create a `.mjs` (or `.ts`) file next to your Gleam source. **Always follow the
project's existing naming convention** — common patterns include
`module.ffi.mjs`, `module_ffi.mjs`, `module.ffi.ts`, `module_ffi.ts`, etc. There
is no enforced standard.

```javascript
// my_module.ffi.mjs  (or my_module_ffi.mjs — match your project)
export function greet(name) {
  return "Hello, " + name;
}
```

Reference it from Gleam using a **relative path from the Gleam source file**
(not the project root):

```gleam
@external(javascript, "./my_module.ffi.mjs", "greet")
pub fn greet(name: String) -> String
```

Node modules are referenced by package name (no path):

```gleam
@external(javascript, "has-flag", "hasFlag")
pub fn argv_has_flag(name: String) -> Bool
```

Compiled Gleam JS output lives in `build/dev/javascript/<package>/`.

### Gleam-generated export naming (TypeScript consumers)

When consuming Gleam-compiled output from TypeScript, Gleam generates exports
using `$` as a separator:

| Pattern                             | Usage                                        |
| ----------------------------------- | -------------------------------------------- |
| `TypeName$VariantName()`            | Constructor                                  |
| `TypeName$isVariantName(v)`         | Type guard (returns `boolean`)               |
| `TypeName$VariantName$fieldName(v)` | Named field accessor                         |
| `TypeName$VariantName$0(v)`         | Positional accessor (unnamed field)          |
| `TypeName$fieldName(v)`             | Shared accessor across variants (same label) |

Direct properties (`instance.field`) also exist but are marked
`/** @deprecated */` — use the accessor functions.

### Option, List, Dict in JS FFI

**Option** — `None` is a class instance, NOT `null`/`undefined`:

```typescript
$option.Option$isSome(opt); // → boolean
$option.Option$isNone(opt); // → boolean
$option.Option$Some$0(opt); // → extracts the inner value
```

**List** — iterable, use spread rather than `.toArray()` (deprecated):

```typescript
[...list].map(f);
Array.from(list).map(f);
```

**Dict** — opaque (`any`). Convert via `$dict.to_list(dict)`:

```typescript
Object.fromEntries([...$dict.to_list(dict)].map(([k, v]) => [k, fv(v)]));
```

### Import style in TypeScript

Always `import * as` — never named imports.

**With `gleam-loaders`** (vite/node loader configured, private project):

```typescript
import * as _ from "gleam:prelude";
import * as $option from "gleam:gleam/option";
import * as _my_module from "gleam:components/my_module";
```

Naming: stdlib → `$` prefix, app modules → `_` prefix, prelude → `_`.

**Without `gleam-loaders`** (or published packages):

```typescript
import * as $option from "../../gleam/gleam_stdlib/option.mjs";
import * as _my_module from "../components/my_module.mjs";
```

Enable TypeScript declarations in `gleam.toml` for `.d.mts` files:

```toml
[javascript]
typescript_declarations = true
```

---

## Multi-target functions

Stack multiple `@external` attributes — one per target:

```gleam
@external(erlang, "lists", "reverse")
@external(javascript, "./project_ffi.mjs", "reverse_list")
pub fn reverse_list(list: List(element)) -> List(element)
```

### Erlang-only with JS fallback body

If a function only makes sense on Erlang, provide a body as the JS fallback:

```gleam
@external(erlang, "persistent_term", "erase")
pub fn erase(_key: String) -> Bool {
  False  // JS fallback — never called on Erlang target
}
```

### JS-only with Erlang fallback body

```gleam
@external(javascript, "./bright.ffi.mjs", "areDependenciesEqual")
fn are_dependencies_equal(a: a, b: b) -> Bool {
  coerce(a) == coerce(b)  // Erlang fallback using structural equality
}
```

### Multi-target gotcha — concurrent IO mismatch

Erlang concurrent IO is transparent/synchronous from Gleam's perspective.
JavaScript requires Promises/callbacks. These models are incompatible. Libraries
that do concurrent IO must typically **choose one target** and document it in
their README.

### Compiler enforcement

If a function has an `@external` for only one target and **no Gleam fallback
body**, compiling for the other target is a **compile error**.

---

## Opaque handle types

Declare a type with no variants to represent an external handle (Erlang PID,
socket, connection, etc.) that Gleam never inspects:

```gleam
pub type Connection  // no body — Gleam treats it as opaque
pub type Channel

@external(erlang, "my_ffi", "connect")
pub fn connect(config: Config) -> Result(Connection, Nil)

@external(erlang, "my_ffi", "send")
pub fn send(channel: Channel, message: String) -> Result(Nil, Nil)
```

Erlang receives and passes back these values as-is — any Erlang term works.

---

## `@target` attribute — deprecated

`@target` marks a file or function as target-specific — it is excluded from
compilation on other targets. It is deprecated and scheduled for removal.

The core problem: `@target` lets you define the same function name twice with
**different signatures** on different targets, and the compiler does not catch
the mismatch. Code that calls the function type-checks on one target but
silently breaks on the other.

```gleam
@target(erlang)   // file-level: whole module excluded on JS target
```

```gleam
@target(erlang)   // function-level
@external(erlang, "persistent_term_ffi", "info")
pub fn info() -> Info
```

**For new code**, prefer an `@external` with a fallback body — it achieves the
same result and will keep working after `@target` is removed:

```gleam
@external(erlang, "persistent_term_ffi", "info")
pub fn info() -> Info {
  Info(count: 0, memory: 0)  // JS fallback
}
```

**If `@target` is already used in the project**, match the existing style — do
not rewrite working code unprompted.

---

## `gleam_erlang` package

Install: `gleam add gleam_erlang@1` (requires OTP 27.0+, current version 1.3.0)

Provides Erlang-specific types and helpers not in `gleam_stdlib`:

| Module                     | Purpose                                             |
| -------------------------- | --------------------------------------------------- |
| `gleam/erlang/atom`        | Atom type: create, get, to_string, decoder          |
| `gleam/erlang/charlist`    | Charlist ↔ String conversion for legacy Erlang APIs |
| `gleam/erlang/process`     | Spawn, link, monitor, message passing, selectors    |
| `gleam/erlang/node`        | Distributed Erlang node management                  |
| `gleam/erlang/application` | OTP application lifecycle                           |
| `gleam/erlang/port`        | Port communication                                  |
| `gleam/erlang/reference`   | Erlang reference type                               |

### `gleam/erlang/atom`

Atom is a string-like type used primarily for BEAM interop. Atoms are **never
garbage collected** — the VM has a hard limit on the atom table.

```gleam
import gleam/erlang/atom.{type Atom}

// Create (adds to atom table — NEVER use with user input!)
atom.create("ok")          // -> Atom

// Look up existing atom (safe — returns Error if not found)
atom.get("ok")             // -> Result(Atom, Nil)

// Convert back to string
atom.to_string(my_atom)    // -> String

// Convert to Dynamic
atom.to_dynamic(my_atom)   // -> Dynamic

// Decoder for use with gleam/dynamic/decode
atom.decoder()             // -> Decoder(Atom)
```

### `gleam/erlang/charlist`

Use when an Erlang API parameter or return is a charlist (list of codepoint
integers) rather than a binary string:

```gleam
import gleam/erlang/charlist.{type Charlist}

charlist.from_string("hello")  // String -> Charlist
charlist.to_string(my_list)    // Charlist -> String
```

---

## `Dynamic` and decoding untyped values

`Dynamic` represents data whose type is unknown at compile time — typically from
Erlang interop or external IO.

### `gleam/dynamic` — creating Dynamic values

```gleam
import gleam/dynamic

dynamic.string("hello")    // -> Dynamic
dynamic.int(42)            // -> Dynamic
dynamic.bool(True)         // -> Dynamic
dynamic.nil()              // -> Dynamic (atom nil on Erlang, undefined on JS)
dynamic.list([...])        // -> Dynamic
dynamic.classify(value)    // -> String describing the runtime type
```

### `gleam/dynamic/decode` — turning Dynamic into typed Gleam

```gleam
import gleam/dynamic/decode

// Primitive decoders (constants):
decode.string    // Decoder(String)
decode.int       // Decoder(Int)
decode.float     // Decoder(Float)
decode.bool      // Decoder(Bool)
decode.dynamic   // Decoder(Dynamic) — always succeeds, no conversion

// Composite decoders:
decode.list(decode.int)                       // Decoder(List(Int))
decode.optional(decode.string)                // Decoder(Option(String))
decode.dict(decode.string, decode.int)        // Decoder(Dict(String, Int))

// Nested field access:
decode.at(["user", "age"], decode.int)        // path navigation
decode.field("name", decode.string, fn(name) { decode.success(name) })

// Run a decoder:
decode.run(my_dynamic_value, decode.string)
// -> Result(String, List(DecodeError))
```

**`DecodeError` type:**

```gleam
type DecodeError {
  DecodeError(expected: String, found: String, path: List(String))
}
```

**Composing decoders with `use`:**

```gleam
let user_decoder = {
  use name <- decode.field("name", decode.string)
  use age  <- decode.field("age",  decode.int)
  decode.success(User(name:, age:))
}

case decode.run(data, user_decoder) {
  Ok(user)   -> // ...
  Error(errs) -> // ...
}
```

**Other decoder combinators:**

```gleam
decode.one_of(decode.int, or: [decode.float])  // try decoders in order
decode.map(decode.string, string.uppercase)     // transform success value
decode.then(decoder, fn(val) -> Decoder(b))    // chain decoders
decode.recursive(fn() { ... })                 // self-referential / nested
```

**Guidance:** Do NOT use `Dynamic` decoders to work with external functions that
return known types. Declare the `@external` with the correct return type
directly. If an Erlang function returns a type you can't directly map, write a
thin Erlang wrapper that converts it first.

---

## JavaScript target — bundler integration

Gleam compiles to ES modules (`.mjs`) in `build/dev/javascript/<package>/`. Run
a build before bundling:

```sh
gleam build --target javascript
```

**Vite** — install the community plugin:

```sh
npm install --save-dev vite-gleam
```

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import gleam from "vite-gleam";

export default defineConfig({ plugins: [gleam()] });
```

Then import `.gleam` files directly — the plugin runs `gleam build` and resolves
them to the compiled `.mjs` output:

```typescript
import { main } from "./src/myapp.gleam";
```

**esbuild / other bundlers** — point at the compiled output after `gleam build`:

```sh
gleam build --target javascript
esbuild build/dev/javascript/myapp/myapp.mjs --bundle --outfile=dist/bundle.js
```

**Node.js** — run compiled output directly, no bundler needed:

```sh
node build/dev/javascript/myapp/myapp.mjs
```

### Gleam ↔ JavaScript type mapping

| Gleam type                | JavaScript representation                               |
| ------------------------- | ------------------------------------------------------- |
| `Bool` (`True`/`False`)   | `true` / `false`                                        |
| `Int`                     | number (whole only; safe range ±2^53−1)                 |
| `Float`                   | number                                                  |
| `String`                  | string                                                  |
| `Nil`                     | `undefined` (**not** `null`)                            |
| `BitArray`                | `BitArray$BitArray(new Uint8Array([...]))` from prelude |
| `List(a)`                 | linked list (`List$NonEmpty`/`List$Empty` from prelude) |
| `#(a, b)` (tuple)         | JS array `[a, b]` — immutable, never mutate             |
| `Result(a, b)` `Ok(v)`    | `Result$Ok(v)` from prelude                             |
| `Result(a, b)` `Error(e)` | `Result$Error(e)` from prelude                          |
| Custom type variant       | `TypeName$VariantName(fields…)` from generated module   |
| `Dict(k, v)`              | opaque — access via imported Gleam functions            |

---

## Gotchas and caveats

1. **No type checking across the boundary.** Wrong annotations = silent runtime
   crash.
2. **Module name collisions.** Don't name Gleam modules the same as Erlang
   stdlib modules (`lists`, `maps`, `erlang`, `io`, etc.).
3. **Charlist vs. binary strings.** Old Erlang APIs return charlists; Gleam
   expects binaries. Use `gleam/erlang/charlist` to convert.
4. **Atom table exhaustion.** Never call `atom.create` with user-supplied
   strings — can crash the VM.
5. **Improper lists.** Gleam lists must be proper lists (`[H|T]` where T is
   always a list). Erlang APIs that return improper lists cause errors.
6. **Multi-target async mismatch.** Erlang concurrent IO is incompatible with
   JavaScript Promises. IO-heavy libraries must usually pick one target.
7. **JavaScript path is relative to the Gleam source file**, not the project
   root.
8. **`.erl` files go in `src/`** — the Gleam compiler auto-compiles them. No
   special build step needed.
9. **Elixir macros** cannot be called from Gleam (they are compile-time only).
10. **Rust NIFs** crash the entire BEAM if they segfault — use with care.
11. **Gleam module `→` Erlang atom uses `@`**, not `/`: `gleam/erlang/node` →
    `gleam@erlang@node`.
12. **JS integers must be within ±2^53−1.** The compiler warns for out-of-range
    literals but not runtime values.
13. **No `Infinity` or `NaN` in Gleam.** Passing these to Gleam code causes
    unexpected behaviour.
14. **`null` is a foreign value.** `Nil` compiles to `undefined`. If a JS API
    returns `null`, wrap it in FFI before passing to Gleam.
15. **`List` is a linked list, not a JS array.** Prefer `gleam/javascript/array`
    at JS boundaries; convert with `from_list`/`to_list`.
16. **ES modules only.** All output uses `import`/`export`. If your setup
    requires CJS, add a bundler step.
