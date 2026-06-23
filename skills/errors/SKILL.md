---
name: gleam-errors
description: >-
  Error handling — Result and Option types, use-expression chaining, the full
  result/option module APIs, error type design, and when to use panic vs let
  assert vs todo
user-invocable: false
---

# Gleam Error Handling

## Result fundamentals

`Result(value, error)` is the standard return type for any function that can
fail. Constructors `Ok(value)` and `Error(reason)` are available everywhere
without imports.

```gleam
fn divide(a: Int, b: Int) -> Result(Int, String) {
  case b {
    0 -> Error("division by zero")
    _ -> Ok(a / b)
  }
}
```

Use `Result(a, Nil)` when failure carries no useful information:

```gleam
fn find_user(id: Int) -> Result(User, Nil) { ... }
```

## Chaining with `use`

`use` desugars a callback into a flat sequence. It is the primary pattern for
chaining multiple fallible operations without nesting.

```gleam
// Nested — hard to read at scale
fn process(input: String) -> Result(Output, AppError) {
  result.try(parse(input), fn(parsed) {
    result.try(validate(parsed), fn(valid) {
      result.map(transform(valid), fn(out) { out })
    })
  })
}

// Flat with use — equivalent, preferred
fn process(input: String) -> Result(Output, AppError) {
  use parsed <- result.try(parse(input))
  use valid  <- result.try(validate(parsed))
  use out    <- result.map(transform(valid))
  out
}
```

`use` works with any function whose last argument is a callback — not just
`result.try`. The value on the right of `<-` must be a function call whose last
parameter is `fn(a) -> b`.

## `result` module API

```gleam
import gleam/result

// Transform the Ok value; propagate Error unchanged
result.map(Ok(1), fn(x) { x * 2 })        // Ok(2)
result.map(Error("e"), fn(x) { x * 2 })   // Error("e")

// Chain: callback returns a new Result; flattens one level
result.try(Ok(1), fn(x) { Ok(x + 1) })    // Ok(2)
result.try(Error("e"), fn(x) { Ok(x) })   // Error("e")

// Transform the Error value; propagate Ok unchanged
result.map_error(Error("low"), fn(e) { AppError(e) })

// Provide a default for the Error case
result.unwrap(Ok(42), 0)       // 42
result.unwrap(Error("e"), 0)   // 0

// Lazy default — avoids evaluating the default when Ok
result.lazy_unwrap(res, fn() { expensive_default() })

// Inspect without consuming (for logging/side-effects)
result.inspect(res, fn(v) { io.println("ok: " <> v) })           // runs on Ok
result.inspect_error(res, fn(e) { io.println("err: " <> e) })    // runs on Error

// Check shape without unwrapping
result.is_ok(res)     // Bool
result.is_error(res)  // Bool

// Flatten Result(Result(a, e), e) → Result(a, e)
result.flatten(Ok(Ok(1)))      // Ok(1)
result.flatten(Ok(Error("e"))) // Error("e")

// Replace Ok value with a constant
result.replace(Ok(1), "done")          // Ok("done")
result.replace_error(Error("e"), 42)   // Error(42)

// Collect a list of Results into Result(List(a), e)
// Stops at the first Error
result.all([Ok(1), Ok(2), Ok(3)])        // Ok([1, 2, 3])
result.all([Ok(1), Error("e"), Ok(3)])   // Error("e")

// Split a list of Results into two lists
result.partition([Ok(1), Error("e"), Ok(2)])  // #([1, 2], ["e"])

// Extract Ok values from a list, discard Errors
result.values([Ok(1), Error("e"), Ok(2)])  // [1, 2]

// Combine two Results — Ok only when both are Ok
result.and(Ok(1), Ok(2))         // Ok(2)
result.and(Error("e"), Ok(2))    // Error("e")

// Return first Ok, or last Error if all fail
result.or(Error("e1"), Ok(2))     // Ok(2)
result.or(Error("e1"), Error("e2"))  // Error("e2")
```

## `option` module API

`Option(a)` is `Some(a)` or `None`. Import constructors to use them unqualified.

```gleam
import gleam/option.{type Option, Some, None}

option.map(Some(1), fn(x) { x * 2 })   // Some(2)
option.map(None, fn(x) { x * 2 })      // None

// Chain: callback returns a new Option
option.then(Some(1), fn(x) { Some(x + 1) })  // Some(2)
option.then(None, fn(x) { Some(x) })          // None

option.unwrap(Some(42), 0)   // 42
option.unwrap(None, 0)        // 0

option.lazy_unwrap(opt, fn() { expensive_default() })

option.is_some(opt)  // Bool
option.is_none(opt)  // Bool

// Flatten Option(Option(a)) → Option(a)
option.flatten(Some(Some(1)))   // Some(1)
option.flatten(Some(None))      // None

option.or(None, Some(2))       // Some(2)
option.or(Some(1), Some(2))    // Some(1)
```

### Converting between Option and Result

```gleam
// Option → Result
option.to_result(Some(1), "missing")   // Ok(1)
option.to_result(None, "missing")      // Error("missing")

// Result → Option (discards the error)
option.from_result(Ok(1))      // Some(1)
option.from_result(Error("e")) // None
```

## Error type design

Define domain-specific error types as custom types. Each variant describes a
business-domain failure; nested fields carry context.

```gleam
pub type AuthError {
  InvalidCredentials
  AccountLocked(reason: String)
  TokenExpired(expired_at: Int)
  InternalError(cause: String)
}
```

**Wrap lower-level errors** to preserve debugging context while keeping the
domain type clean:

```gleam
pub type RegisterError {
  DuplicateEmail(email: String)
  WeakPassword
  DatabaseError(cause: String)   // wraps db error string, not the db type
}

fn register(email: String, password: String) -> Result(User, RegisterError) {
  use _  <- result.map_error(check_password_strength(password), fn(_) { WeakPassword })
  use _  <- result.map_error(check_email_unique(email), fn(_) { DuplicateEmail(email) })
  use u  <- result.map_error(db.insert_user(email), fn(e) { DatabaseError(db.error_message(e)) })
  Ok(u)
}
```

**Use `result.map_error` to translate error types** when crossing module
boundaries — each layer should expose its own error type, not leak internals.

```gleam
fn handle_login(req: Request) -> Result(Response, HttpError) {
  use user <- result.map_error(auth.login(req), fn(e) {
    case e {
      auth.InvalidCredentials -> HttpError(401, "invalid credentials")
      auth.AccountLocked(r)   -> HttpError(403, r)
      auth.InternalError(e)   -> HttpError(500, e)
    }
  })
  Ok(respond_with_token(user))
}
```

## When to use panic, todo, let assert, assert

These constructs crash the process. Use them only when a crash is the correct
response.

| Construct    | Use case                                                                                                                                        |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `todo`       | Placeholder during development — marks code paths not yet written. Compile warning, runtime crash if reached.                                   |
| `panic`      | Genuinely unreachable branch — use when you have already proven the case cannot happen. Prefer `let assert` if you have a value to destructure. |
| `let assert` | Unwrap a value you know succeeds — crash with a clear message if the pattern fails. Use in tests and for program invariants.                    |
| `assert`     | Assert a boolean invariant — crashes if `False`. Use in tests or for checking preconditions that must hold.                                     |

```gleam
// todo — development placeholder
fn upcoming_feature() -> Result(Nil, Error) {
  todo as "implement after API design is finalised"
}

// panic — unreachable branch after prior validation
fn day_name(n: Int) -> String {
  case n {
    1 -> "Monday"   2 -> "Tuesday"   3 -> "Wednesday"
    4 -> "Thursday" 5 -> "Friday"    6 -> "Saturday"
    7 -> "Sunday"
    _ -> panic as "day_name called with out-of-range value"
  }
}

// let assert — known-good unwrap
pub fn main() {
  let assert Ok(config) = load_config()  // crash on startup if config is broken
  run(config)
}

// assert — boolean invariant check
let total = list.fold(items, 0, fn(acc, x) { acc + x })
assert total >= 0
```

**Never use `let assert` or `panic` in library code** — libraries must return
`Result` so callers can handle failures. These constructs are for application
entry points, tests, and genuinely impossible states.

## Common patterns

### Early return on first error

```gleam
fn process(input: String) -> Result(Output, AppError) {
  use parsed  <- result.try(parse(input))
  use valid   <- result.try(validate(parsed))
  use saved   <- result.try(db.save(valid))
  Ok(build_response(saved))
}
```

### Accumulating all errors (not stopping at first)

`result.all` stops at the first error. To collect all errors, fold manually:

```gleam
fn validate_all(inputs: List(String)) -> Result(List(Parsed), List(String)) {
  let #(oks, errs) =
    inputs
    |> list.map(parse)
    |> result.partition

  case errs {
    [] -> Ok(oks)
    _  -> Error(errs)
  }
}
```

### Providing a fallback without failing

```gleam
// Try the fast path, fall back to slow path, never propagate Error
// result.lazy_or avoids calling db.get when the cache already has a value
let value =
  cache.get(key)
  |> result.lazy_or(fn() { db.get(key) })
  |> result.unwrap(default_value)
```

### Logging errors without changing the type

```gleam
result.inspect_error(op(), fn(e) {
  io.println("operation failed: " <> string.inspect(e))
})
```
