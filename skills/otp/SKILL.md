---
name: gleam-otp
description: >-
  Gleam OTP — actors (gen_server equivalent), supervisors, process subjects,
  selectors, monitoring, named processes, timers, and factory supervisors
user-invocable: false
---

# Gleam OTP

`gleam_otp` provides OTP abstractions for Gleam on the Erlang target. Install
with:

```sh
gleam add gleam_otp gleam_erlang
```

Modules:

- `gleam/otp/actor` — main concurrency primitive (gen_server equivalent)
- `gleam/otp/static_supervisor` — fixed set of children
- `gleam/otp/factory_supervisor` — dynamic children from a template
- `gleam/otp/supervision` — child spec helpers
- `gleam/erlang/process` — subjects, selectors, monitors, timers

---

## Actors

An actor is a process that receives typed messages sequentially, maintains
state, and is OTP-compliant (responds to system messages for tracing/debugging).

### Minimal actor

```gleam
import gleam/erlang/process
import gleam/otp/actor

pub type Msg {
  Increment
  Get(reply_with: process.Subject(Int))
  Shutdown
}

fn handle(state: Int, msg: Msg) -> actor.Next(Int, Msg) {
  case msg {
    Increment -> actor.continue(state + 1)
    Get(client) -> {
      process.send(client, state)
      actor.continue(state)
    }
    Shutdown -> actor.stop()
  }
}

pub fn main() {
  let assert Ok(started) =
    actor.new(0)
    |> actor.on_message(handle)
    |> actor.start

  let subject = started.data   // process.Subject(Msg)
  process.send(subject, Increment)
  let count = process.call(subject, 100, Get)
  process.send(subject, Shutdown)
}
```

### `actor.Next` return values

| Expression                                           | Effect                                              |
| ---------------------------------------------------- | --------------------------------------------------- |
| `actor.continue(state)`                              | Stay in loop with new state                         |
| `actor.stop()`                                       | Graceful shutdown (reason: Normal)                  |
| `actor.stop_abnormal("reason")`                      | Abnormal exit — linked processes receive the reason |
| `actor.continue(state) \|> actor.with_selector(sel)` | Continue and swap to a new selector                 |

### Custom initialiser

Use `actor.new_with_initialiser` when the actor needs to do work before
returning to the parent (e.g. open a connection, register a name):

```gleam
pub fn start() -> Result(actor.Started(process.Subject(Msg)), actor.StartError) {
  actor.new_with_initialiser(5000, fn(subject) {
    // setup that can fail — return Error(reason_string) to abort
    use conn <- result.try(connect() |> result.map_error(string.inspect))
    let state = State(subject:, conn:)
    actor.initialised(state)
    |> actor.returning(subject)   // what the parent receives as started.data
    |> Ok
  })
  |> actor.on_message(handle)
  |> actor.start
}
```

- `actor.initialised(state)` — wraps initial state
- `actor.returning(value)` — sets `started.data` returned to parent; defaults to
  `process.Subject(Msg)`
- Timeout (ms) is how long the parent waits for init to complete before
  returning `InitTimeout`

### `actor.StartError`

```gleam
pub type StartError {
  InitTimeout
  InitFailed(String)   // init returned Error(msg)
  InitExited(process.ExitReason)
}
```

Wrap it in your own error type:

```gleam
pub type MyError {
  ActorFailed(actor.StartError)
  // ...
}

actor.start(builder)
|> result.map_error(ActorFailed)
```

---

## Subjects and messaging

`process.Subject(message)` is the typed message channel for an actor. A subject
is owned by the process that created it — only that process receives messages
sent to it.

```gleam
let subject: process.Subject(Msg) = process.new_subject()

// Async send (never blocks)
process.send(subject, SomeMessage)
actor.send(subject, SomeMessage)   // alias in actor module

// Sync request-reply — crashes caller on timeout
let reply = process.call(subject, 100, fn(reply_subject) { Get(reply_subject) })
// or shorthand:
let reply = actor.call(subject, 100, Get)

// Low-level receive (use selectors instead in most cases)
let result: Result(Msg, Nil) = process.receive(subject, timeout_ms: 500)
```

### Timers

```gleam
import gleam/erlang/process

// Schedule a message after delay_ms milliseconds
let timer: process.Timer = process.send_after(subject, delay_ms, MyMessage)

// Cancel a pending timer
let result: process.Cancelled = process.cancel_timer(timer)
```

`process.send_after` is the standard way to implement periodic actors:

```gleam
fn handle(state: State, msg: Msg) -> actor.Next(State, Msg) {
  case msg {
    Tick -> {
      do_work(state)
      process.send_after(state.self, 5000, Tick)
      actor.continue(state)
    }
  }
}
```

---

## Selectors

A `Selector` lets an actor receive from multiple subjects (and monitors) at
once, mapping them to a single message type.

```gleam
import gleam/erlang/process

pub opaque type Msg {
  FromWorker(WorkerMsg)
  WorkerDied(process.Down)
}

// Build a selector that merges two sources
let selector =
  process.new_selector()
  |> process.select_map(_, for: worker_subject, mapping: FromWorker)
  |> process.select_monitors(_, mapping: WorkerDied)
```

Set the selector during initialisation:

```gleam
actor.initialised(state)
|> actor.selecting(selector)
|> actor.returning(subject)
|> Ok
```

Or swap selector mid-loop:

```gleam
actor.continue(state) |> actor.with_selector(new_selector)
```

| Function                                   | Purpose                                                         |
| ------------------------------------------ | --------------------------------------------------------------- |
| `process.new_selector()`                   | Empty selector                                                  |
| `process.select(sel, subject)`             | Receive from subject (message type must match selector payload) |
| `process.select_map(sel, subject, mapper)` | Receive from subject and map to payload type                    |
| `process.merge_selector(a, b)`             | Combine two selectors of same payload type                      |
| `process.select_monitors(sel, mapper)`     | Receive `process.Down` from monitored processes                 |
| `process.deselect(sel, subject)`           | Remove subject from selector                                    |

---

## Process monitoring

```gleam
import gleam/erlang/process

// Monitor a pid — get notified when it exits
let monitor: process.Monitor = process.monitor(some_pid)

// In selector: receive Down messages
process.new_selector()
|> process.select_monitors(_, fn(down: process.Down) { WorkerDied(down) })

// Down carries:
// ProcessDown(monitor: Monitor, pid: Pid, reason: ExitReason)
```

`process.ExitReason` is `Normal | Killed | Abnormal(dynamic.Dynamic)`.

---

## Named processes

`process.Name(message)` wraps an Erlang atom for global registration.

```gleam
// Create at startup only — each call allocates a new atom
let name: process.Name(Msg) = process.new_name("my_actor")

// Register an actor by name
actor.new(state)
|> actor.on_message(handle)
|> actor.named(name)
|> actor.start

// Look up a named process later
let subject: process.Subject(Msg) = process.named_subject(name)
```

**Warning:** `process.new_name` creates an Erlang atom on every call. Never call
it inside a loop or inside a supervised process that restarts — atoms are never
garbage-collected and the VM crashes if too many are created.

The typical pattern is to create names once in `main` and pass them down:

```gleam
pub fn main() {
  let registry_name = process.new_name("registry")
  let assert Ok(_) = registry.start(registry_name)
  // ...
}
```

---

## Static supervisor

Manages a fixed set of child processes, restarting them on failure.

```gleam
import gleam/otp/static_supervisor as supervisor
import gleam/otp/supervision

pub fn start_supervisor() {
  supervisor.new(supervisor.OneForOne)
  |> supervisor.add(database.supervised())   // supervision.ChildSpecification
  |> supervisor.add(http_server.supervised())
  |> supervisor.start
}
```

**Strategies:**

| Strategy     | Behaviour                                             |
| ------------ | ----------------------------------------------------- |
| `OneForOne`  | Restart only the crashed child                        |
| `OneForAll`  | Restart all children when any one crashes             |
| `RestForOne` | Restart crashed child + all children started after it |

**Restart tolerance** (defaults: intensity=2, period=5 seconds):

```gleam
supervisor.restart_tolerance(builder, intensity: 5, period: 10)
```

If more than `intensity` restarts occur within `period` seconds, the supervisor
terminates all children and itself.

**Auto-shutdown:**

```gleam
supervisor.auto_shutdown(builder, supervisor.AnySignificant)
// Options: Never (default), AnySignificant, AllSignificant
```

**Nesting supervisors** — convert a supervisor to a child spec with
`supervised()`:

```gleam
fn worker_supervisor() {
  supervisor.new(supervisor.OneForOne)
  |> supervisor.add(worker_a.supervised())
  |> supervisor.add(worker_b.supervised())
  |> supervisor.supervised()   // returns ChildSpecification, not Started
}

pub fn main() {
  supervisor.new(supervisor.OneForOne)
  |> supervisor.add(worker_supervisor())
  |> supervisor.start
}
```

---

## Child specifications (`supervision`)

`supervision.worker` and `supervision.supervisor` create `ChildSpecification`
values.

```gleam
import gleam/otp/supervision

// Most actors expose a `supervised()` function like this:
pub fn supervised() -> supervision.ChildSpecification(process.Subject(Msg)) {
  use <- supervision.worker()    // `use` expression, returns ChildSpec
  start()                        // your actor.start call
}

// Or with options:
supervision.worker(start)
|> supervision.restart(supervision.Transient)   // Permanent | Transient | Temporary
|> supervision.timeout(ms: 10_000)              // shutdown timeout (default 5000ms)
|> supervision.significant(True)                // triggers auto_shutdown when it exits
```

| Restart     | Behaviour                       |
| ----------- | ------------------------------- |
| `Permanent` | Always restarted (default)      |
| `Transient` | Restarted only on abnormal exit |
| `Temporary` | Never restarted                 |

---

## Factory supervisor

Manages a dynamic set of children created from a single template. Useful for
spinning up per-request or per-connection workers.

```gleam
import gleam/otp/factory_supervisor as factory

// Build the factory (does not start yet)
let factory_spec =
  factory.worker_child(my_worker.start)  // fn(arg) -> StartResult(data)
  |> factory.named(workers_name)
  |> factory.supervised()                // ChildSpecification for the parent supervisor

// Start children dynamically at runtime
let factory_supervisor = factory.get_by_name(workers_name)
let assert Ok(started) = factory.start_child(factory_supervisor, my_arg)
```

Children default to `Transient` restart (restarted on abnormal exit only).

---

## Common patterns

### Request-reply (call)

The standard OTP synchronous call pattern — pass a `Subject` in the message:

```gleam
pub type Msg {
  GetState(reply_with: process.Subject(State))
}

// Caller:
let state = actor.call(subject, 500, GetState)
// Equivalent to:
let state = process.call(subject, 500, fn(reply_subject) { GetState(reply_subject) })
```

`actor.call` crashes the caller if no reply arrives within the timeout. For
fallible calls, use `process.receive` manually.

### Self-scheduling periodic actor

```gleam
pub opaque type Msg { Tick }

type State { State(self: process.Subject(Msg), interval_ms: Int) }

fn handle(state: State, msg: Msg) -> actor.Next(State, Msg) {
  case msg {
    Tick -> {
      do_periodic_work()
      process.send_after(state.self, state.interval_ms, Tick)
      actor.continue(state)
    }
  }
}

pub fn start(interval_ms: Int) -> actor.StartResult(process.Subject(Msg)) {
  actor.new_with_initialiser(1000, fn(subject) {
    let state = State(self: subject, interval_ms:)
    process.send(subject, Tick)   // kick off first tick immediately
    actor.initialised(state) |> actor.returning(subject) |> Ok
  })
  |> actor.on_message(handle)
  |> actor.start
}
```

### Storing subjects for broadcasting

Subjects can be stored in ETS tables or passed around to enable multi-process
fan-out:

```gleam
// Store subject in a table
let subject: process.Subject(Msg) = process.new_subject()
table.insert(sockets_table, user_id, subject)

// Broadcast to all stored subjects
list.each(all_subjects, fn(subject) {
  process.send(subject, Broadcast(payload))
})
```

### Abnormal exit triggers supervisor restart

Return `actor.stop_abnormal("reason")` to signal the supervisor the actor
crashed and should be restarted. Return `actor.stop()` when the actor is done
and should not restart.

### Startup order with `let assert Ok`

It is idiomatic in `main` to assert that supervisors and actors start
successfully. Failures at startup are programming errors, not runtime errors:

```gleam
pub fn main() {
  let assert Ok(_postgres) = postgres.supervised() |> supervision.worker |> actor.start
  let assert Ok(_) =
    supervisor.new(supervisor.OneForOne)
    |> supervisor.add(http_server.supervised())
    |> supervisor.start
  process.sleep_forever()
}
```
