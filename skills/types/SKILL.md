---
name: gleam-types
description: >-
  Type system patterns — opaque types for encapsulation and smart constructors,
  phantom types for compile-time state tracking, and Dynamic decode as a
  boundary validation strategy
user-invocable: false
---

# Gleam Type System Patterns

## Opaque types — encapsulation and invariants

`pub opaque type` makes the type public but hides its constructors. Only the
defining module can construct or pattern-match on values of that type.

```gleam
// Define an opaque type with a private constructor
pub opaque type Email {
  Email(address: String)
}

// Smart constructor — the only way to create an Email from outside this module
pub fn from_string(s: String) -> Result(Email, String) {
  case string.contains(s, "@") {
    True  -> Ok(Email(s))
    False -> Error("invalid email: " <> s)
  }
}

// Accessor — expose the inner value without exposing the constructor
pub fn to_string(email: Email) -> String {
  email.address
}
```

Callers cannot do `Email("bad")` — they must go through `from_string`, so the
invariant (must contain `@`) holds for every `Email` in existence.

### When to use opaque types

| Use case                                      | Pattern                                                  |
| --------------------------------------------- | -------------------------------------------------------- |
| Validated primitive (email, URL, port number) | Smart constructor returning `Result`                     |
| Non-negative / bounded number                 | Smart constructor clamping or returning `Result`         |
| Opaque handle to external resource            | Empty body: `pub opaque type Connection` — see FFI skill |
| Builder accumulator                           | Opaque type with chained `with_*` functions              |

### Inside-module access

Pattern-matching on opaque types is allowed **only within the defining module**:

```gleam
// Inside the email module — fine
pub fn domain(email: Email) -> String {
  let Email(address) = email
  let assert [_, domain] = string.split(address, "@")
  domain
}

// From outside — compile error: Email constructor not in scope
// let Email(addr) = some_email  ✗
```

### Opaque type vs. type alias

```gleam
pub type Metres = Float   // alias — NOT opaque, constructors are just Float literals
pub opaque type Metres { Metres(Float) }  // opaque — can only be made via module functions
```

---

## Phantom types — type-level state tracking

A **phantom type** is a type parameter that appears in the type definition but
not in the data. It carries compile-time information without any runtime cost.

```gleam
// The `state` parameter is never used in the fields — it's purely a type tag
pub opaque type Connection(state) {
  Connection(host: String, socket: Socket)
}

pub type Open
pub type Closed

// Only returns Connection(Open) — the type encodes that it's open
pub fn connect(host: String) -> Result(Connection(Open), Error) { ... }

// Only accepts Connection(Open) — compiler rejects Connection(Closed)
pub fn query(conn: Connection(Open), sql: String) -> Result(Rows, Error) { ... }

// Transitions from Open to Closed
pub fn close(conn: Connection(Open)) -> Connection(Closed) { ... }

// Using a closed connection is a compile error:
// let conn = connect("localhost") |> result.unwrap(...)
// let _ = close(conn)
// query(conn, "SELECT 1")   ✗  — type mismatch: Connection(Closed) vs Connection(Open)
```

The phantom type is opaque to prevent callers from forging state:
`Connection(Open)` can only be produced by `connect()`, not by arbitrary code.

### Common phantom type patterns

**Validated vs. unvalidated input:**

```gleam
pub opaque type FormData(state) { FormData(fields: Dict(String, String)) }
pub type Raw
pub type Validated

pub fn parse(raw: String) -> FormData(Raw) { ... }

pub fn validate(data: FormData(Raw)) -> Result(FormData(Validated), List(String)) { ... }

// Functions that process data only accept validated form
pub fn save(data: FormData(Validated)) -> Result(Nil, DbError) { ... }
```

**Capability / permission token:**

```gleam
pub opaque type Capability(action) { Capability }
pub type Read
pub type Write

pub fn get_read_cap(auth: Auth) -> Option(Capability(Read)) { ... }
pub fn get_write_cap(auth: Auth) -> Option(Capability(Write)) { ... }

pub fn read_file(path: String, _cap: Capability(Read)) -> Result(String, Error) { ... }
pub fn write_file(path: String, content: String, _cap: Capability(Write)) -> Result(Nil, Error) { ... }
```

**Units of measure (prevent mixing incompatible values):**

```gleam
pub opaque type Quantity(unit) { Quantity(value: Float) }
pub type Metres
pub type Seconds
pub type MetresPerSecond

pub fn metres(v: Float) -> Quantity(Metres) { Quantity(v) }
pub fn seconds(v: Float) -> Quantity(Seconds) { Quantity(v) }

pub fn velocity(dist: Quantity(Metres), time: Quantity(Seconds)) -> Quantity(MetresPerSecond) {
  let Quantity(d) = dist
  let Quantity(t) = time
  Quantity(d /. t)
}
// metres(10.0) |> velocity(seconds(2.0))  ✓
// seconds(10.0) |> velocity(metres(2.0))  ✗  type mismatch
```

### Phantom types with multiple parameters

```gleam
pub opaque type Query(input, output) { Query(sql: String) }
pub type UserRow
pub type PostRow

pub fn user_by_id() -> Query(Int, UserRow) { Query("SELECT * FROM users WHERE id = $1") }
pub fn run(db: Db, query: Query(a, b), param: a) -> Result(List(b), Error) { ... }
```

---

## `Dynamic` decode — boundary validation

`Dynamic` is the type of values whose shape is unknown at compile time — JSON
parsed at runtime, Erlang terms, HTTP request bodies. The decode API turns
`Dynamic` into typed Gleam at the boundary.

For the full decode API surface, see the **gleam-ffi** skill. This section
covers design strategy.

### The boundary principle

Decode at the entry point; never pass `Dynamic` into your domain logic.

```gleam
// ✓ — decode at the HTTP handler, domain functions get typed values
pub fn handle_create_user(req: Request) -> Response {
  use body   <- result.try(request.body_json(req))
  use params <- result.map_error(decode.run(body, user_params_decoder()), decode_error_response)
  create_user(params)  // pure domain function, fully typed
}

// ✗ — leaking Dynamic into business logic
pub fn create_user(data: Dynamic) -> Result(User, Error) { ... }
```

### Defining reusable decoders

Decoders are first-class values — define them as functions, compose them freely.
See **gleam-ffi** for the full API (`decode.field`, `decode.optional`,
`decode.at`, etc.).

The key pattern not obvious from the API: use `decode.then` + `decode.failure`
to decode a string into a custom type:

```gleam
pub type Role { Admin Member }

fn role_decoder() -> decode.Decoder(Role) {
  decode.string
  |> decode.then(fn(s) {
    case s {
      "admin"  -> decode.success(Admin)
      "member" -> decode.success(Member)
      other    -> decode.failure(Member, "expected admin|member, got: " <> other)
    }
  })
}
```

### Recursive / self-referential decoders

Use `decode.recursive` to decode tree structures:

```gleam
pub type Tree {
  Leaf(value: Int)
  Node(left: Tree, right: Tree)
}

fn tree_decoder() -> decode.Decoder(Tree) {
  decode.one_of(
    decode.recursive(fn() {
      use left  <- decode.field("left",  tree_decoder())
      use right <- decode.field("right", tree_decoder())
      decode.success(Node(left:, right:))
    }),
    or: [
      decode.field("value", decode.int, fn(v) { decode.success(Leaf(v)) }),
    ],
  )
}
```

### Handling decode errors

`decode.run` returns `Result(a, List(DecodeError))`. Collect all field errors
and surface them together — don't just take the first:

```gleam
import gleam/dynamic/decode.{DecodeError}
import gleam/list

fn format_errors(errs: List(DecodeError)) -> String {
  errs
  |> list.map(fn(e) {
    "at " <> string.join(e.path, ".") <> ": expected " <> e.expected <> ", got " <> e.found
  })
  |> string.join(", ")
}
```

### `decode.one_of` for tagged unions / sum types

```gleam
pub type Shape { Circle(Float) Rectangle(Float, Float) }

fn shape_decoder() -> decode.Decoder(Shape) {
  use tag <- decode.field("type", decode.string)
  case tag {
    "circle" -> {
      use r <- decode.field("radius", decode.float)
      decode.success(Circle(r))
    }
    "rectangle" -> {
      use w <- decode.field("width",  decode.float)
      use h <- decode.field("height", decode.float)
      decode.success(Rectangle(w, h))
    }
    other -> decode.failure(Circle(0.0), "unknown shape type: " <> other)
  }
}
```

### Avoid `Dynamic` for known types

If an `@external` function returns a type you can map directly, declare the
return type in Gleam — do not decode it:

```gleam
// ✗ — unnecessary Dynamic round-trip
@external(erlang, "my_lib", "get_count")
fn get_count_raw() -> Dynamic

pub fn get_count() -> Result(Int, List(DecodeError)) {
  decode.run(get_count_raw(), decode.int)
}

// ✓ — declare the real type
@external(erlang, "my_lib", "get_count")
pub fn get_count() -> Int
```

Use `Dynamic` decode only when the shape is genuinely unknown at compile time.
