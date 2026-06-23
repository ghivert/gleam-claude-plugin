---
name: gleam-testing
description: >-
  Gleam testing — gleeunit test runner, assert keyword, should.* assertions,
  birdie snapshot testing, qcheck property-based testing, test organisation, and
  fixture patterns
user-invocable: false
---

# Gleam Testing

## Setup

```sh
gleam add --dev gleeunit
```

Create `test/<project_name>_test.gleam` — the entry point gleeunit looks for:

```gleam
import gleeunit

pub fn main() {
  gleeunit.main()
}
```

Run all tests:

```sh
gleam test
```

---

## Test discovery

Any **public function ending in `_test`** in the `test/` directory is
automatically discovered and run. No registration needed.

```gleam
// test/math_test.gleam
import gleeunit
import myapp/math

pub fn main() {
  gleeunit.main()
}

pub fn add_test() {
  assert math.add(1, 2) == 3
}

pub fn divide_by_zero_test() {
  assert math.divide(10, 0) == Error(Nil)
}
```

A test fails if it panics (any unhandled panic, failed assertion, or
`let assert` mismatch).

---

## Assertions

### `assert` keyword (preferred)

The `assert` keyword evaluates a boolean expression and panics with a diff if
it's `False`:

```gleam
assert value == expected
assert list.length(items) > 0
assert result == Ok("hello")
```

### `let assert` for pattern matching

`let assert` destructures and panics if the pattern doesn't match — useful for
unwrapping `Ok`/`Some` in tests:

```gleam
let assert Ok(user) = db.find_user(id)
let assert Some(token) = cache.get("session")
let assert [first, ..] = results
```

### `gleeunit/should` (legacy, still common)

The `should` module is an older pipeline-friendly assertion style. The gleeunit
docs now recommend `assert` instead, but you will encounter `should` in existing
codebases:

```gleam
import gleeunit/should

value |> should.equal(expected)
value |> should.not_equal(other)

result |> should.be_ok          // returns the inner Ok value
result |> should.be_error       // returns the inner Error value

option |> should.be_some        // returns inner Some value
option |> should.be_none

bool_value |> should.be_true
bool_value |> should.be_false

should.fail()                   // unconditionally fail the test
```

---

## Test organisation

**One entry point per project.** The file `test/<project_name>_test.gleam`
contains only `main()`. Tests live in separate modules:

```
test/
  myapp_test.gleam        ← entry point: only contains main()
  math_test.gleam
  user_test.gleam
  helpers/
    db_helper.gleam       ← shared setup code (no _test suffix = not auto-run)
```

Each test module imports `gleeunit` only if it defines `main` — otherwise just
import what you need:

```gleam
// test/math_test.gleam
import myapp/math

pub fn add_test() {
  assert math.add(2, 3) == 5
}
```

---

## Test helpers and fixtures

Extract reusable setup into helper modules. Use `use` expressions for resource
setup/teardown:

```gleam
// test/helpers/db_helper.gleam
import myapp/db

pub fn with_connection(callback: fn(db.Connection) -> a) -> a {
  let assert Ok(conn) = db.connect_test()
  let result = callback(conn)
  db.disconnect(conn)
  result
}

pub fn with_seeded_db(callback: fn(db.Connection) -> a) -> a {
  use conn <- with_connection
  let assert Ok(_) = db.run_migrations(conn)
  let assert Ok(_) = db.seed_test_data(conn)
  callback(conn)
}
```

Using the helper in tests:

```gleam
// test/user_test.gleam
import test/helpers/db_helper

pub fn find_user_test() {
  use conn <- db_helper.with_seeded_db
  let assert Ok(user) = user_repo.find(conn, id: "123")
  assert user.email == "test@example.com"
}
```

---

## Snapshot testing with Birdie

Birdie captures a string value as a snapshot and fails if it changes between
runs. Good for testing serialisation, codegen output, or complex formatted
results.

```sh
gleam add --dev birdie
```

```gleam
import birdie
import gleeunit

pub fn main() {
  gleeunit.main()
}

pub fn render_invoice_test() {
  invoice.render(sample_invoice())
  |> birdie.snap(title: "render_invoice_test")
}

pub fn encode_user_test() {
  user.to_json(sample_user())
  |> birdie.snap(title: "encode_user_test")
}
```

**Workflow:**

1. `gleam test` — new/changed snapshots are flagged as failures
2. `gleam run -m birdie` — interactive review: accept or reject each change
3. Commit `birdie_snapshots/` to version control

Snapshots are stored in `birdie_snapshots/<title>.accepted`. The title must be
unique across the whole project — duplicate titles would compare unrelated
things.

**Tips:**

- `birdie.snap` only accepts `String` — convert your type first
  (`string.inspect`, custom `to_string`, etc.)
- Keep each snapshot small and focused on one concept
- Use descriptive titles that double as documentation

---

## Property-based testing with qcheck

qcheck generates thousands of random test cases from a description of
invariants. It also shrinks failing cases to the simplest example.

```sh
gleam add --dev qcheck
```

```gleam
import qcheck

pub fn reverse_twice_is_identity_test() {
  use xs <- qcheck.given(qcheck.list(qcheck.small_non_negative_int()))
  assert list.reverse(list.reverse(xs)) == xs
}

pub fn add_commutative_test() {
  use #(a, b) <- qcheck.given(qcheck.tuple2(
    qcheck.bounded_int(-100, 100),
    qcheck.bounded_int(-100, 100),
  ))
  assert math.add(a, b) == math.add(b, a)
}
```

**Core API:**

```gleam
qcheck.given(generator, fn(value) { ... }) -> Nil
```

**Built-in generators:**

| Generator                         | Produces                    |
| --------------------------------- | --------------------------- |
| `qcheck.small_non_negative_int()` | Small non-negative integers |
| `qcheck.uniform_int()`            | Full-range integers         |
| `qcheck.bounded_int(min, max)`    | Integers within bounds      |
| `qcheck.list(gen)`                | Lists of generated values   |
| `qcheck.tuple2(gen_a, gen_b)`     | Pairs                       |

**Combining generators:**

```gleam
// Use tuple2/map2 — never nest qcheck.given() calls
qcheck.return(MyRecord)
|> qcheck.apply(qcheck.bounded_int(0, 100))
|> qcheck.apply(qcheck.uniform_int())
```

On failure, qcheck reports both the original failing value and the shrunken
minimal example.

---

## Common patterns

### Testing `Result` and `Option`

```gleam
// assert + let assert
assert my_fn() == Ok(42)
let assert Ok(value) = my_fn()
assert value > 0

// should style
my_fn() |> should.be_ok |> should.equal(42)
```

### Testing that something errors

```gleam
assert my_fn(bad_input) == Error(InvalidInput)

// or extract and inspect the error:
let assert Error(err) = my_fn(bad_input)
assert err == InvalidInput
```

### Multi-case table tests

Gleam has no built-in parameterised tests — use a list and `list.each`:

```gleam
pub fn parse_test() {
  let cases = [
    #("1", Ok(1)),
    #("0", Ok(0)),
    #("abc", Error(Nil)),
    #("", Error(Nil)),
  ]
  list.each(cases, fn(case_) {
    let #(input, expected) = case_
    assert parser.parse(input) == expected
  })
}
```

### Labelled assertions with `as`

Add context to a `let assert` failure:

```gleam
let assert Ok(_) = db.migrate(conn) as "migration must succeed before tests run"
```
