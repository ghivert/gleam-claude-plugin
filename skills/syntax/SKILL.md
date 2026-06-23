---
name: gleam-syntax
description: >-
  Full Gleam language syntax reference — types, custom types, pattern matching,
  functions, pipelines, generics, use expressions, and standard constructs
user-invocable: false
---

# Gleam Language Reference

Source: https://tour.gleam.run/everything/

## Basics

```gleam
import gleam/io
import gleam/io.{println}           // unqualified import
import gleam/bytes_tree.{type BytesTree}  // type import

io.println("Hello")
echo 42                             // debug print any type

// Ints: +  -  *  /  %  ==  !=  <  >
// Floats: +.  -.  *.  /.  (division by 0.0 → 0.0)
// Number formats: 1_000  0xFF  0b1010  0o17  7.0e7
// Strings: double-quoted, multiline, \n \t \u{1F600}, <> concat
// Bools: True False  &&  ||  !  (short-circuiting)

let x = "value"                     // immutable binding
let x = "rebind"                    // shadowing is fine
let _unused = 42                    // _ prefix silences warning
let name: String = "typed"         // optional annotation

pub type Number = Int               // type alias (not a new type)
const MAX: Int = 100                // module-level constant

// Blocks: { ... } — last expression is the value
let result = {
  let x = 10
  x * 2
}

// Lists: singly-linked, homogeneous, O(1) prepend
let nums = [1, 2, 3]
let more = [-1, 0, ..nums]
```

## Functions

```gleam
fn double(a: Int) -> Int { a * 2 }
pub fn main() { echo double(5) }

// Higher-order & anonymous
fn twice(arg: Int, f: fn(Int) -> Int) -> Int { f(f(arg)) }
let add_one = fn(x) { x + 1 }

// Function capture — _ is the argument slot
let add_one = add(1, _)

// Generic functions
fn twice(arg: a, f: fn(a) -> a) -> a { f(f(arg)) }

// Pipelines
"Hello" |> string.reverse |> io.println

// Labelled arguments (order-independent when labelled)
fn calc(value: Int, add x: Int) { value + x }
calc(1, add: 2)
calc(add: 2, value: 1)

// Label shorthand when variable name == label
calc(value:, add: delta)

/// Doc comment for functions/types
//// Doc comment for modules
@deprecated("Use new_func instead")
fn old_func() { Nil }
```

## Flow Control

```gleam
// Case — exhaustive, compiler-enforced
case x {
  0 -> "Zero"
  n -> int.to_string(n)     // variable pattern
}

// String prefix pattern
case greeting {
  "Hello, " <> name -> name
  _ -> "Unknown"
}

// List patterns
case list {
  []           -> "empty"
  [x]          -> "one"
  [h, ..rest]  -> "many"
}

// Multiple subjects
case x, y {
  0, 0 -> "both zero"
  _, _ -> "other"
}

// Alternative patterns
case n {
  2 | 4 | 6 -> "even"
  _          -> "other"
}

// Pattern alias
case lists {
  [[_, ..] as first, ..] -> first
  _ -> []
}

// Guards
case numbers {
  [first, ..] if first > limit -> first
  _ -> 0
}

// Recursion & tail calls
fn sum(list: List(Int), acc: Int) -> Int {
  case list {
    [h, ..t] -> sum(t, acc + h)   // tail call — reuses stack frame
    []       -> acc
  }
}
```

## Data Types

```gleam
// Tuples — fixed-size, mixed types
let t = #(1, 2.2, "three")
t.0          // index access
let #(a, _, c) = t

// Custom types with variants
pub type Season { Spring Summer Autumn Winter }

// Records (variants with fields)
pub type Person { Person(name: String, age: Int) }
let p = Person("Alice", 30)
p.name                          // field access
let Person(name, age) = p       // pattern match

// Record update
let p2 = Person(..p, age: 31)

// Generic custom types
pub type Option(a) { Some(a) None }

// Nil — unit type, not nullable
let x = Nil

// Result — built-in success/failure
fn divide(a: Int, b: Int) -> Result(Int, String) {
  case b {
    0 -> Error("division by zero")
    _ -> Ok(a / b)
  }
}

// Bit arrays
<<3>>                       // 8-bit int
<<"Hello":utf8>>            // UTF-8 string
<<1:size(4), 2:size(4)>>    // 4-bit segments
```

## Standard Library Highlights

```gleam
import gleam/list
list.map([1,2,3], fn(x) { x * 2 })
list.filter([1,2,3], fn(x) { x > 1 })
list.fold([1,2,3], 0, fn(acc, e) { acc + e })

import gleam/result
result.map(Ok(1), fn(x) { x * 2 })
result.try(Ok("1"), int.parse)
result.unwrap(Ok("v"), "default")

import gleam/dict
dict.from_list([#("a", 1)]) |> dict.insert("b", 2) |> dict.delete("a")

import gleam/option.{type Option, Some, None}
```

## Advanced

```gleam
// Opaque types — public type, private constructors
pub opaque type PositiveInt { PositiveInt(inner: Int) }
pub fn new(i: Int) -> PositiveInt {
  case i >= 0 {
    True -> PositiveInt(i)
    False -> PositiveInt(0)
  }
}

// use — desugars to a callback, reduces nesting
use username <- result.try(get_username())
use password <- result.try(get_password())
username <> password
// equivalent to:
//   result.try(get_username(), fn(username) {
//     result.try(get_password(), fn (password) {
//       username <> password
//     })
//   })

// todo / panic / let assert / assert
todo as "not implemented"
panic as "unreachable"
let assert [first, ..] = items      // crashes if pattern fails
assert add(1, 2) == 3               // crashes if false

// Externals (FFI)
// Partials
@external(javascript, "./ffi.mjs", "now")
pub fn now() -> DateTime

// Total
@external(javascript, "./ffi.mjs", "now")
@external(erlang, "ffi", "now")
pub fn now() -> DateTime

// Multi-target + Gleam fallback
@external(erlang, "lists", "reverse")
pub fn reverse(items: List(a)) -> List(a) {
  tail_recursive_reverse(items, [])  // used on JS target
}
```
