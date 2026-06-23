---
name: gleam-lustre
description:
  Description of Lustre, the Gleam frontend framework — MVU architecture,
  HTML/events/effects API, components, server components, SSR, testing,
  deployment
user-invocable: false
---

# Lustre

Lustre is a web framework for Gleam. It can produce static HTML, client-side
SPAs, Web Components, and real-time server components. Install with
`gleam add lustre`.

Docs: https://lustre.hexdocs.pm/

---

## Architecture: Model-View-Update (MVU)

Every Lustre app is built from three pure functions:

```gleam
fn init(args) -> #(Model, Effect(Message))   // or just Model for lustre.simple
fn update(model: Model, msg: Message) -> #(Model, Effect(Message))
fn view(model: Model) -> Element(Message)
```

- State lives entirely in `Model`; UI is a pure function of it
- All external events produce a `Message` value
- `update` is the only place state changes
- `Effect` values describe work to run (HTTP, timers, etc.) — not the work
  itself

---

## Application Types

```gleam
// Static element — no state
lustre.element(html.text("Hello"))

// Simple — no effects
lustre.simple(init, update, view)

// Full — with effects
lustre.application(init, update, view)

// Web Component
lustre.component(init, update, view, options)
```

Start on the client:

```gleam
let assert Ok(_) = lustre.start(app, "#app", flags)
```

---

## HTML Elements (`lustre/element/html`)

All element functions: `tag(attributes, children)`. Void elements take only
attributes.

```gleam
import lustre/element/html
import lustre/element.{text, none, fragment}
import lustre/attribute

html.div([attribute.class("container")], [
  html.h1([], [text("Title")]),
  html.p([], [text("Body")]),
  html.input([attribute.type_("text"), attribute.placeholder("…")]),
])
```

Key elements: `div`, `span`, `p`, `h1`–`h6`, `a`, `button`, `input`, `form`,
`select`, `option`, `textarea`, `label`, `ul`/`ol`/`li`,
`table`/`thead`/`tbody`/`tr`/`th`/`td`, `img`, `video`, `audio`,
`details`/`summary`, `dialog`, `section`, `article`, `nav`, `header`, `footer`,
`main`, `pre`, `code`, `script`, `style`, `svg`, `math`.

Utilities:

```gleam
element.text("hello")        // text node
element.none()               // renders nothing — use for conditionals
element.fragment([…])        // multiple nodes without a wrapper
element.map(el, fn(msg) { … })  // transform message type
element.memo(deps, view_fn)  // skip re-render when deps are reference-equal
```

String output (for SSR):

```gleam
element.to_string(el)
element.to_document_string(el)   // prepends <!doctype html>
element.to_readable_string(el)   // indented, for debugging
```

---

## Attributes (`lustre/attribute`)

```gleam
import lustre/attribute.{attribute, property}

attribute.class("foo")
attribute.classes([#("active", is_active), #("disabled", is_disabled)])
attribute.style("color", "red")
attribute.styles([#("color", "red"), #("font-size", "14px")])
attribute.id("my-id")
attribute.type_("text")
attribute.value("hello")          // controlled input
attribute.default_value("hello")  // uncontrolled input
attribute.disabled(True)
attribute.readonly(True)
attribute.required(True)
attribute.placeholder("…")
attribute.href("/path")
attribute.src("/img.png")
attribute.alt("description")
attribute.checked(True)
attribute.selected(True)
attribute.hidden(True)
attribute.autofocus(True)
attribute.tabindex(0)
attribute.data("test-id", "foo")   // data-test-id="foo"
attribute.aria("label", "close")   // aria-label="close"
attribute.none()                   // no-op; useful in conditionals

// Raw escape hatch
attribute.attribute("data-x", "y")   // serializable (SSR-safe)
attribute.property("innerHTML", json.string("<b>bold</b>"))  // DOM-only, not SSR
```

---

## Events (`lustre/event`)

```gleam
import lustre/event

// Common convenience handlers
event.on_click(MyMessage)
event.on_input(fn(value) { UserTyped(value) })
event.on_change(fn(value) { SelectChanged(value) })
event.on_check(fn(checked) { Toggled(checked) })
event.on_submit(fn(fields) { FormSubmitted(fields) })  // prevents default
event.on_keydown(fn(key) { KeyPressed(key) })
event.on_focus(Focused)
event.on_blur(Blurred)
event.on_mouse_enter(Hovered)

// Custom event with decoder
event.on("custom-event", my_decoder)

// Modifiers (wrap existing attribute)
event.prevent_default(event.on_click(Msg))
event.stop_propagation(event.on_click(Msg))

// Rate limiting (ms) — useful for server components with network latency
event.debounce(event.on_input(UserTyped), 300)
event.throttle(event.on_click(Clicked), 200)

// Emit from a component to the outside world
event.emit("my-event", json.object([#("value", json.string("x"))]))
```

---

## Effects (`lustre/effect`)

Effects are descriptions of work; the runtime executes them after `update`.

```gleam
import lustre/effect

effect.none()                         // no side effect
effect.batch([effect_a, effect_b])    // run both

// Custom effect — dispatch sends a message back
effect.from(fn(dispatch) {
  do_something()
  |> MyMessage
  |> dispatch
})

// DOM timing
effect.before_paint(fn(dispatch, root) { … })  // after render, before paint
effect.after_paint(fn(dispatch, root) { … })   // after browser paint

// Context (Web Components Community Group protocol)
effect.provide("theme", json.string("dark"))
effect.subscribe("theme", decode.string |> decode.map(ThemeChanged))
effect.unsubscribe("theme")

// Transform message type
effect.map(some_effect, fn(inner_msg) { OuterMsg(inner_msg) })
```

Common community packages for effects:

- `rsvp` — HTTP requests
- `modem` — SPA routing / navigation
- `plinth` — Node.js and browser API bindings

---

## Keyed Elements (`lustre/element/keyed`)

Use keyed elements when rendering dynamic lists to help the diffing algorithm
reuse DOM nodes.

```gleam
import lustre/element/keyed

keyed.ul([], list.map(items, fn(item) {
  #(item.id, html.li([], [html.text(item.label)]))
}))

// Also available: keyed.div, keyed.ol, keyed.dl, keyed.tbody
// Generic: keyed.element, keyed.fragment, keyed.namespaced
```

Keys must be unique within a list but can repeat across different lists.

---

## SVG (`lustre/element/svg`)

```gleam
import lustre/element/svg

svg.svg([attribute.attribute("viewBox", "0 0 100 100")], [
  svg.circle([attribute.attribute("cx", "50"), attribute.attribute("cy", "50"), attribute.attribute("r", "40")]),
  svg.rect([…]),
  svg.path([attribute.attribute("d", "M 0 0 L 100 100")]),
  svg.text([…], "label"),
  svg.g([…], [ … ]),  // group
])
```

---

## Web Components (`lustre/component`)

Register a Lustre app as a Custom Element:

```gleam
let assert Ok(_) = lustre.register(app, "my-counter")
// then use <my-counter /> in HTML
```

Component options:

```gleam
import lustre/component

component.on_connect(Connected)
component.on_disconnect(Disconnected)
component.on_attribute_change("count", fn(s) { result.map(int.parse(s), CountSet) })
component.on_property_change("items", items_decoder)
component.open_shadow_root(True)   // default: True
component.adopt_styles(True)       // inherit parent stylesheets
component.delegates_focus(False)
component.form_associated()        // participate in forms
```

Shadow DOM:

```gleam
component.default_slot([], fallback_children)
component.named_slot("header", [], fallback_children)
component.slot("header")   // attribute to assign child to a named slot
component.part("thumb")    // expose for ::part() styling
```

Form-associated effects:

```gleam
component.set_form_value("42")
component.clear_form_value()
component.set_pseudo_state("checked")
component.remove_pseudo_state("checked")
```

---

## Server Components (`lustre/server_component`)

Server components run on the server (Erlang) and stream DOM patches to the
browser.

```gleam
import lustre/server_component

// In your HTML view, mount the component:
server_component.element([
  server_component.route("/ws/my-component"),
  server_component.method(server_component.WebSocket),  // or ServerSentEvents / Polling
  server_component.csrf_token(token),
], [])

// Include the client runtime (dev only; bundle in prod):
server_component.script()
```

Start on the server:

```gleam
lustre.start_server_component(app, args)
// or with OTP supervision:
lustre.supervised(app, args)
lustre.factory(app)  // multiple dynamic instances
```

Send event properties to the server:

```gleam
server_component.include(event.on_input(UserTyped), ["target.value"])
```

---

## State Management Best Practices

**Model design** — use custom types to make impossible states impossible:

```gleam
type Model {
  LoggedIn(LoggedInModel)
  Public(PublicModel)
}
```

**Message naming** — use Subject-Verb-Object:

```gleam
// Good
UserClickedSave
ApiReturnedItems(Result(List(Item), Error))
// Avoid
Save
SetItems
```

**Prefer view functions over components** — components add boilerplate and are
harder to test. Only reach for `lustre.component` when you need encapsulation,
web component interop, or shadow DOM.

---

## Server-Side Rendering

```gleam
// No lustre.start needed — just call your view and convert to string
import lustre/element

let html = html.html([], [
  html.head([], [html.title([], "My App")]),
  html.body([], [view(model)]),
])

element.to_document_string(html)   // → full HTML string with <!doctype html>
```

**Hydration**: embed the initial model as JSON in a `<script id="model">` tag on
the server; decode and use it as `init` args on the client.

---

## Testing (`lustre/dev/query` + `lustre/dev/simulate`)

### Query

```gleam
import lustre/dev/query

// Build a query
let q = query.element(query.tag("button"))
let q = query.child(of: query.element(query.tag("form")), matching: query.tag("input"))
let q = query.descendant(of: query.element(query.id("root")), matching: query.class("item"))

// Selectors
query.tag("div")
query.id("my-id")
query.class("active")
query.attribute("type", "submit")
query.data("test-id", "submit-btn")   // data-test-id
query.aria("label", "close")
query.test_id("submit-btn")           // shorthand for data("test-id", …)
query.text("Click me")
query.and(query.tag("button"), query.class("primary"))

// Run query against an Element tree
query.find(in: root_element, matching: q)         // → Result(Element, Nil)
query.find_all(in: root_element, matching: q)     // → List(Element)
query.matches(target: el, selector: query.tag("div"))
query.has(in: el, matching: query.class("foo"))
```

### Simulate

```gleam
import lustre/dev/simulate

let sim =
  simulate.simple(init, update, view)
  |> simulate.start(Nil)
  |> simulate.click(on: query.element(query.test_id("increment")))
  |> simulate.input(on: query.element(query.tag("input")), value: "hello")
  |> simulate.submit(on: query.element(query.tag("form")), fields: [#("name", "Alice")])
  |> simulate.message(SomeDirectMessage)
  |> simulate.expect(fn(model) { assert model.count == 1 })

let model = simulate.model(sim)
let view  = simulate.view(sim)
let log   = simulate.history(sim)   // List(Event(message))
```

Note: effects are discarded in simulations — test effect logic separately.

---

## Deployment

### SPA (static hosting)

Build: `gleam run -m lustre/dev build --minify --outdir=dist`

**GitHub Pages** — GitHub Actions workflow: install Gleam, build to `dist/`,
upload artifact, deploy pages.

**Cloudflare Pages** — same workflow + Wrangler action:
`pages deploy dist --project-name <name>`.

Client-side routing 404 fix: add a `404.html` that redirects to
`/?path=<original>`, then recover the path in `index.html` with
`window.history.replaceState`.

### Full-stack (Docker + Fly.io)

Monorepo structure: `/client` (JS target) + `/server` (Erlang target) +
`/shared` (common types).

Build client into server's static dir:

```sh
gleam run -m lustre/dev build --minify --outdir=../server/priv/static
```

Export Erlang release:

```sh
gleam export erlang-shipment
```

Two-stage Dockerfile: build stage compiles both; runtime stage uses Alpine + the
Erlang shipment.

Fly.io: `fly launch --no-deploy`, adjust `internal_port` in `fly.toml`, then
`fly deploy`.

---

## Quick API Index

| Module                    | Key exports                                                                                                                                                                             |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `lustre`                  | `element`, `simple`, `application`, `component`, `start`, `register`, `start_server_component`, `dispatch`, `send`, `shutdown`, `is_browser`                                            |
| `lustre/element`          | `text`, `none`, `fragment`, `element`, `namespaced`, `map`, `memo`, `unsafe_raw_html`, `to_string`, `to_document_string`                                                                |
| `lustre/element/html`     | All HTML tags as functions                                                                                                                                                              |
| `lustre/element/svg`      | All SVG tags as functions                                                                                                                                                               |
| `lustre/element/keyed`    | `element`, `fragment`, `div`, `ul`, `ol`, `tbody`                                                                                                                                       |
| `lustre/attribute`        | `attribute`, `property`, `none`, `class`, `classes`, `style`, `styles`, `id`, `type_`, `value`, `href`, `src`, `disabled`, `checked`, `aria`, `data` + all HTML attrs                   |
| `lustre/event`            | `on`, `on_click`, `on_input`, `on_change`, `on_check`, `on_submit`, `on_keydown/up/press`, `on_focus`, `on_blur`, `prevent_default`, `stop_propagation`, `debounce`, `throttle`, `emit` |
| `lustre/effect`           | `none`, `from`, `batch`, `before_paint`, `after_paint`, `map`, `provide`, `subscribe`, `unsubscribe`                                                                                    |
| `lustre/component`        | `on_connect`, `on_disconnect`, `on_attribute_change`, `on_property_change`, `default_slot`, `named_slot`, `slot`, `part`, `set_form_value`, `set_pseudo_state`, `prerender`             |
| `lustre/server_component` | `element`, `script`, `route`, `method`, `csrf_token`, `include`, `emit`                                                                                                                 |
| `lustre/dev/query`        | `element`, `child`, `descendant`, `tag`, `id`, `class`, `attribute`, `data`, `aria`, `test_id`, `text`, `and`, `find`, `find_all`, `matches`                                            |
| `lustre/dev/simulate`     | `simple`, `application`, `start`, `message`, `event`, `click`, `input`, `submit`, `expect`, `model`, `view`, `history`                                                                  |
