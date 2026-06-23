---
name: gleam-redraw
description: >-
  Redraw — type-safe React bindings for Gleam: components, hooks (state, effect,
  ref, memo, context), event handling, and interop with plain React
user-invocable: false
---

# Redraw — Gleam React Bindings

Redraw provides type-safe React bindings for Gleam. It maps React's component
model to Gleam's functional paradigm. Requires `redraw` and `redraw_dom`
packages (and optionally `redraw_batteries`).

## Core concepts

- **Element**: a DOM node or the result of executing a component. Elements are
  immutable and have no state.
- **ReactComponent(props)**: an opaque type — a function that accepts props and
  returns Elements. Components hold state and run hooks.
- The central distinction: _components create elements_. Never confuse the two.

## Setup

```sh
gleam add redraw redraw_dom redraw_batteries
```

Bootstrap flow — components must be composed before they can be used:

```gleam
import redraw
import redraw/dom/client
import redraw/dom/html as h

pub fn main() {
  let assert Ok(root) = client.create_root("app")
  client.render_(root, app(), fn(app) { app(Nil) })
}

fn app() {
  use counter <- redraw.compose(counter())
  use _props <- redraw.component_("App")
  h.div([], [counter(CounterProps(initial: 0))])
}

fn counter() {
  use props: CounterProps <- redraw.component_("Counter")
  let #(count, set_count) = redraw.use_state(props.initial)
  h.button([events.on_click(fn(_) { set_count(count + 1) })], [
    h.text(int.to_string(count)),
  ])
}
```

## `redraw` module

### Types

```gleam
pub type Element
pub type ReactComponent(props)
pub type Context(a)
pub type Suspense { Suspense(fallback: Element) }
pub type Error {
  ExistingContext(name: String)
  UnknownContext(name: String)
  DevelopmentOnly
  OwnerStackUnavailable
}
```

### Component creation

```gleam
// Primary way to create a component. Must be the last compose call (the actual component body).
pub fn component_(name: String, render: fn(props) -> Element) -> ReactComponent(props)

// Compose another component into this one — must happen during bootstrap.
pub fn compose(component: ReactComponent(props), return: fn(fn(props) -> Element) -> ReactComponent(p)) -> ReactComponent(p)

// Wrap with React.memo — skips re-render when props are unchanged.
pub fn memoize_(component: ReactComponent(props)) -> ReactComponent(props)

// Standalone component with no props.
pub fn standalone(name: String, render: fn() -> Element) -> fn() -> Element
```

### State hooks

```gleam
pub fn use_state(initial_value: a) -> #(a, fn(a) -> Nil)
pub fn use_state_(initial_value: a) -> #(a, fn(fn(a) -> a) -> Nil)  // updater form
pub fn use_lazy_state(initial_value: fn() -> a) -> #(a, fn(a) -> Nil)
pub fn use_lazy_state_(initial_value: fn() -> a) -> #(a, fn(fn(a) -> a) -> Nil)
pub fn use_reducer(reducer: fn(state, action) -> state, initial_state: state) -> #(state, fn(action) -> Nil)
pub fn use_reducer_(reducer: fn(state, action) -> state, initializer: i, init: fn(i) -> state) -> #(state, fn(action) -> Nil)
```

### Effect hooks

```gleam
// Dependencies are tuples, e.g. #(a, b). Use Nil for run-once, omit trailing fields freely.
pub fn use_effect(value: fn() -> Nil, dependencies: a) -> Nil
pub fn use_effect_(value: fn() -> fn() -> Nil, dependencies: a) -> Nil  // with cleanup
pub fn use_layout_effect(value: fn() -> Nil, dependencies: a) -> Nil
pub fn use_layout_effect_(value: fn() -> fn() -> Nil, dependencies: a) -> Nil
pub fn use_insertion_effect(handler: fn() -> Nil, deps: deps) -> Nil
pub fn use_insertion_effect_(handler: fn() -> fn() -> Nil, deps: deps) -> Nil
```

### Context

```gleam
pub fn create_context_(default_value: a) -> Context(a)
pub fn use_context(context: Context(a)) -> a
pub fn provider(context: Context(a), value: a, children: List(Element)) -> Element
```

Context is created at module level (outside components), then threaded through
`provider`:

```gleam
const theme_context = redraw.create_context_(Light)

fn themed_app() {
  use _props <- redraw.component_("ThemedApp")
  redraw.provider(theme_context, Dark, [child_component(ChildProps)])
}

fn child() {
  use _props <- redraw.component_("Child")
  let theme = redraw.use_context(theme_context)
  // use theme...
}
```

### Ref hooks

```gleam
pub fn use_ref() -> ref.Ref(option.Option(a))         // for optional DOM refs
pub fn use_ref_(initial_value: a) -> ref.Ref(a)        // for typed persistent data
pub fn use_imperative_handle(ref: ref.Ref(option.Option(a)), handler: fn() -> a, dependencies: b) -> Nil
pub fn use_imperative_handle_(ref: ref.Ref(a), handler: fn() -> a, dependencies: b) -> Nil
```

### Performance hooks

```gleam
pub fn use_memo(calculate_value: fn() -> a, dependencies: b) -> a
pub fn use_callback(fun: function, dependencies: dependencies) -> function
pub fn use_deferred_value(value: a) -> a
```

### Transition hooks

```gleam
pub fn use_transition() -> #(Bool, fn() -> Nil)
pub fn use_async_transition() -> #(Bool, fn() -> promise.Promise(Nil))
pub fn start_transition(scope: fn() -> Nil) -> Nil
pub fn use_promise(promise: promise.Promise(state)) -> state  // suspends until resolved
```

### Form / action hooks

```gleam
pub fn use_action_state(action: fn(state, payload) -> Nil, initial_state: state) -> #(state, fn(payload) -> nil, Bool)
pub fn use_optimistic(state: state) -> #(state, fn(state) -> Nil)
pub fn use_optimistic_(state: state) -> #(state, fn(fn(state) -> state) -> Nil)
pub fn use_optimistic_action(state: state, update: fn(state, action) -> state) -> #(state, fn(action) -> Nil)
pub fn use_sync_external_store(subscribe: fn(fn() -> Nil) -> fn() -> Nil, get_snapshot: fn() -> snapshot) -> snapshot
```

### Utility

```gleam
pub fn fragment(children: List(Element)) -> Element
pub fn keyed(element: fn(List(Element)) -> Element, content: List(#(String, Element))) -> Element
pub fn strict_mode(children: List(Element)) -> Element
pub fn suspense(props: Suspense, children: List(Element)) -> Element
pub fn profiler(children: List(Element)) -> Element
pub fn use_id() -> String
pub fn use_debug_value(value: a) -> Nil
pub fn use_debug_value_(value: a, formatter: fn(a) -> String) -> Nil
pub fn capture_owner_stack() -> Result(String, Error)  // dev only
pub fn act(act_fn: fn() -> promise.Promise(Nil)) -> promise.Promise(Nil)  // testing
```

---

## `redraw/ref` module

```gleam
pub type Ref(a)  // mutable reference, persists across renders

pub fn current(from ref: Ref(a)) -> a
pub fn assign(of ref: Ref(a), with value: a) -> Nil
```

---

## `redraw/dom/client` module

```gleam
pub type Root

pub fn create_root(root: String) -> Result(Root, dom.Error)
pub fn hydrate_root(root: String, node: redraw.Element) -> Result(Root, dom.Error)
pub fn render(root: Root, child: redraw.Element) -> Nil           // deprecated
pub fn render_(root: Root, child: redraw.ReactComponent(Nil), return: fn(fn(Nil) -> redraw.Element) -> redraw.Element) -> Nil
pub fn unmount(root: Root) -> Nil
pub fn virtual_root() -> #(dynamic.Dynamic, Root)
```

---

## `redraw/dom/html` module

All container elements follow: `fn(List(Attribute), List(Element)) -> Element`

All self-closing elements follow: `fn(List(Attribute)) -> Element`

**Container elements:** `a`, `abbr`, `address`, `article`, `aside`, `audio`,
`b`, `bdi`, `bdo`, `blockquote`, `body`, `button`, `canvas`, `caption`, `cite`,
`code`, `colgroup`, `data`, `datalist`, `dd`, `del`, `details`, `dfn`, `dialog`,
`div`, `dl`, `dt`, `em`, `fieldset`, `figcaption`, `figure`, `footer`, `form`,
`h1`–`h6`, `head`, `header`, `hgroup`, `i`, `ins`, `kbd`, `label`, `legend`,
`li`, `main`, `map`, `mark`, `menu`, `meter`, `nav`, `noscript`, `object`, `ol`,
`optgroup`, `option`, `output`, `p`, `picture`, `pre`, `progress`, `q`, `rp`,
`rt`, `ruby`, `s`, `samp`, `section`, `select`, `slot`, `small`, `span`,
`strong`, `sub`, `summary`, `sup`, `table`, `tbody`, `td`, `template`,
`textarea`, `tfoot`, `th`, `thead`, `time`, `tr`, `u`, `ul`, `var`, `video`

**Self-closing elements:** `area`, `base`, `br`, `col`, `embed`, `hr`, `iframe`,
`img`, `input`, `link`, `meta`, `source`, `track`, `wbr`

**Special:**

```gleam
pub fn element(tag: String, attrs: List(Attribute), children: List(Element)) -> Element
pub fn text(content: String) -> Element
pub fn none() -> Element   // renders nothing
pub fn script(attrs: List(Attribute), script: String) -> Element
pub fn style(attrs: List(Attribute), content: String) -> Element
pub fn title(attrs: List(Attribute), content: String) -> Element
pub fn html(attrs: List(Attribute), children: List(Element)) -> Element
pub fn to_props(attrs: List(Attribute)) -> props
```

---

## `redraw/dom/attribute` module

```gleam
pub type Attribute
pub type Dir { Ltr | Rtl }
pub type Translate { Yes | No }
pub type InnerHTML

// Key attributes
pub fn id(String) -> Attribute
pub fn class(String) -> Attribute           // additive
pub fn class_name(String) -> Attribute
pub fn style(List(#(String, String))) -> Attribute
pub fn key(String) -> Attribute             // React list key
pub fn ref(Ref(Option(a))) -> Attribute
pub fn ref_(fn(a) -> Nil) -> Attribute      // callback ref

// Common HTML attributes
pub fn href(String) -> Attribute
pub fn src(String) -> Attribute
pub fn type_(String) -> Attribute
pub fn value(String) -> Attribute
pub fn checked(Bool) -> Attribute
pub fn disabled(Bool) -> Attribute
pub fn placeholder(String) -> Attribute
pub fn name(String) -> Attribute
pub fn for(String) -> Attribute             // label for
pub fn html_for(String) -> Attribute
pub fn alt(String) -> Attribute
pub fn action(String) -> Attribute
pub fn action_(fn(FormData) -> Nil) -> Attribute
pub fn method(String) -> Attribute
pub fn target(String) -> Attribute
pub fn rel(String) -> Attribute
pub fn autocomplete(String) -> Attribute
pub fn autofocus(Bool) -> Attribute
pub fn autoplay(Bool) -> Attribute
pub fn controls(Bool) -> Attribute
pub fn loop(Bool) -> Attribute
pub fn multiple(Bool) -> Attribute
pub fn readonly(Bool) -> Attribute
pub fn required(Bool) -> Attribute
pub fn selected(Bool) -> Attribute
pub fn open(Bool) -> Attribute
pub fn hidden(Bool) -> Attribute
pub fn draggable(Bool) -> Attribute
pub fn tab_index(Int) -> Attribute
pub fn width(Int) -> Attribute
pub fn height(Int) -> Attribute
pub fn rows(Int) -> Attribute
pub fn cols(Int) -> Attribute
pub fn max(String) -> Attribute
pub fn min(String) -> Attribute
pub fn step(String) -> Attribute
pub fn pattern(String) -> Attribute
pub fn max_length(String) -> Attribute
pub fn min_length(String) -> Attribute
pub fn col_span(Int) -> Attribute
pub fn row_span(Int) -> Attribute
pub fn role(String) -> Attribute
pub fn aria(String, String) -> Attribute
pub fn data(String, String) -> Attribute    // data-* attributes
pub fn attribute(String, a) -> Attribute    // generic key-value
pub fn none() -> Attribute                 // no-op, for conditionals
pub fn lang(String) -> Attribute
pub fn dir(Dir) -> Attribute
pub fn translate(Translate) -> Attribute
pub fn content_editable(Bool) -> Attribute
pub fn inert(Bool) -> Attribute
pub fn input_mode(String) -> Attribute
pub fn popover(String) -> Attribute
pub fn slot(String) -> Attribute
pub fn access_key(String) -> Attribute
pub fn download(String) -> Attribute
pub fn cross_origin(String) -> Attribute
pub fn nonce(String) -> Attribute
pub fn title(String) -> Attribute
pub fn spell_check(Bool) -> Attribute
pub fn suppress_hydration_warning(Bool) -> Attribute
pub fn suppress_content_editable_warning(Bool) -> Attribute
pub fn dangerously_set_inner_html(InnerHTML) -> Attribute
pub fn inner_html(String) -> InnerHTML
```

---

## `redraw/dom/events` module

All event handlers return `Attribute`. Each also has a `_capture` variant for
capture-phase handling.

```gleam
// Mouse
pub fn on_click(fn(mouse.MouseEvent) -> Nil) -> Attribute
pub fn on_double_click(fn(mouse.MouseEvent) -> Nil) -> Attribute
pub fn on_context_menu(fn(mouse.MouseEvent) -> Nil) -> Attribute
pub fn on_mouse_down(fn(mouse.MouseEvent) -> Nil) -> Attribute
pub fn on_mouse_up(fn(mouse.MouseEvent) -> Nil) -> Attribute
pub fn on_mouse_move(fn(mouse.MouseEvent) -> Nil) -> Attribute
pub fn on_mouse_enter(fn(mouse.MouseEvent) -> Nil) -> Attribute
pub fn on_mouse_leave(fn(mouse.MouseEvent) -> Nil) -> Attribute
pub fn on_mouse_out(fn(mouse.MouseEvent) -> Nil) -> Attribute
pub fn on_aux_click(fn(mouse.MouseEvent) -> Nil) -> Attribute

// Input / change
pub fn on_change(fn(input.InputEvent) -> Nil) -> Attribute
pub fn on_input(fn(input.InputEvent) -> Nil) -> Attribute
pub fn on_before_input(fn(input.InputEvent) -> Nil) -> Attribute

// Keyboard
pub fn on_key_down(fn(keyboard.KeyboardEvent) -> Nil) -> Attribute
pub fn on_key_up(fn(keyboard.KeyboardEvent) -> Nil) -> Attribute
pub fn on_key_press(fn(keyboard.KeyboardEvent) -> Nil) -> Attribute

// Focus
pub fn on_focus(fn(focus.FocusEvent) -> Nil) -> Attribute
pub fn on_blur(fn(focus.FocusEvent) -> Nil) -> Attribute

// Form
pub fn on_submit(fn(event.Event) -> Nil) -> Attribute
pub fn on_reset(fn(event.Event) -> Nil) -> Attribute

// Pointer
pub fn on_pointer_down(fn(pointer.PointerEvent) -> Nil) -> Attribute
pub fn on_pointer_up(fn(pointer.PointerEvent) -> Nil) -> Attribute
pub fn on_pointer_move(fn(pointer.PointerEvent) -> Nil) -> Attribute
pub fn on_pointer_enter(fn(pointer.PointerEvent) -> Nil) -> Attribute
pub fn on_pointer_leave(fn(pointer.PointerEvent) -> Nil) -> Attribute
pub fn on_pointer_cancel(fn(pointer.PointerEvent) -> Nil) -> Attribute
pub fn on_pointer_out(fn(pointer.PointerEvent) -> Nil) -> Attribute
pub fn on_got_pointer_capture(fn(pointer.PointerEvent) -> Nil) -> Attribute
pub fn on_lost_pointer_capture(fn(pointer.PointerEvent) -> Nil) -> Attribute

// Touch
pub fn on_touch_start(fn(touch.TouchEvent) -> Nil) -> Attribute
pub fn on_touch_end(fn(touch.TouchEvent) -> Nil) -> Attribute
pub fn on_touch_move(fn(touch.TouchEvent) -> Nil) -> Attribute
pub fn on_touch_cancel(fn(touch.TouchEvent) -> Nil) -> Attribute

// Drag
pub fn on_drag(fn(drag.DragEvent) -> Nil) -> Attribute
pub fn on_drag_start(fn(drag.DragEvent) -> Nil) -> Attribute
pub fn on_drag_end(fn(drag.DragEvent) -> Nil) -> Attribute
pub fn on_drag_enter(fn(drag.DragEvent) -> Nil) -> Attribute
pub fn on_drag_leave(fn(drag.DragEvent) -> Nil) -> Attribute
pub fn on_drag_over(fn(drag.DragEvent) -> Nil) -> Attribute
pub fn on_drop(fn(drag.DragEvent) -> Nil) -> Attribute

// Clipboard
pub fn on_copy(fn(clipboard.ClipboardEvent) -> Nil) -> Attribute
pub fn on_cut(fn(clipboard.ClipboardEvent) -> Nil) -> Attribute
pub fn on_paste(fn(clipboard.ClipboardEvent) -> Nil) -> Attribute

// Animation
pub fn on_animation_start(fn(animation.AnimationEvent) -> Nil) -> Attribute
pub fn on_animation_end(fn(animation.AnimationEvent) -> Nil) -> Attribute
pub fn on_animation_iteration(fn(animation.AnimationEvent) -> Nil) -> Attribute

// Transition
pub fn on_transition_end(fn(transition.TransitionEvent) -> Nil) -> Attribute

// Wheel / scroll
pub fn on_wheel(fn(wheel.WheelEvent) -> Nil) -> Attribute
pub fn on_scroll(fn(event.Event) -> Nil) -> Attribute

// Composition
pub fn on_composition_start(fn(composition.CompositionEvent) -> Nil) -> Attribute
pub fn on_composition_update(fn(composition.CompositionEvent) -> Nil) -> Attribute
pub fn on_composition_end(fn(composition.CompositionEvent) -> Nil) -> Attribute

// Media events (all fn(event.Event) -> Nil)
// on_abort, on_can_play, on_can_play_through, on_cancel, on_close,
// on_duration_change, on_emptied, on_encrypted, on_ended, on_error,
// on_load, on_load_start, on_loaded_data, on_loaded_metadata,
// on_pause, on_play, on_playing, on_progress, on_rate_change,
// on_resize, on_seeked, on_seeking, on_select, on_stalled,
// on_suspend, on_time_update, on_toggle, on_volume_change, on_waiting

pub fn none() -> Attribute  // conditional no-op
```

---

## `redraw/event` module

```gleam
pub type Event  // synthetic React event (cross-browser normalised)

pub fn bubbles(event: Event) -> Bool
pub fn cancelable(event: Event) -> Bool
pub fn current_target(event: Event) -> Dynamic
pub fn default_prevented(event: Event) -> Bool
pub fn event_phase(event: Event) -> Int
pub fn is_trusted(event: Event) -> Bool
pub fn target(event: Event) -> Dynamic
pub fn time_stamp(event: Event) -> Int
pub fn native_event(event: Event) -> Dynamic

pub fn prevent_default(event: Event) -> Event
pub fn stop_propagation(event: Event) -> Event
pub fn persist(event: Event) -> Event             // React Native only
pub fn is_propagation_stopped(event: Event) -> Bool
pub fn is_default_prevented(event: Event) -> Bool
pub fn is_persistent(event: Event) -> Bool
```

---

## `redraw/dom/svg` module

All functions: `fn(List(Attribute), List(Element)) -> Element`

**Shapes:** `circle`, `ellipse`, `line`, `path`, `polygon`, `polyline`, `rect`

**Text:** `text`, `text_path`, `tspan`

**Containers:** `g`, `svg`, `symbol`, `defs`, `marker`, `pattern`, `clip_path`,
`mask`

**Gradients:** `linear_gradient`, `radial_gradient`, `stop`

**Filters:** `filter`, `fe_blend`, `fe_color_matrix`, `fe_component_transfer`,
`fe_composite`, `fe_convolve_matrix`, `fe_diffuse_lighting`,
`fe_displacement_map`, `fe_distant_light`, `fe_drop_shadow`, `fe_flood`,
`fe_func_a`, `fe_func_b`, `fe_func_g`, `fe_func_r`, `fe_gaussian_blur`,
`fe_image`, `fe_merge`, `fe_merge_node`, `fe_morphology`, `fe_offset`,
`fe_point_light`, `fe_specular_lighting`, `fe_spot_light`, `fe_tile`,
`fe_turbulence`

**Animation:** `animate`, `animate_motion`, `animate_transform`, `set`, `mpath`

**Other:** `a`, `foreign_object`, `image`, `switch`, `use_`, `view`, `desc`,
`title`, `metadata`, `style`, `script`, `discard`, `hatch`, `hatchpath`

---

## `redraw_batteries` package

### `redraw/batteries/compose`

Ergonomic helpers for the bootstrap composition phase.

```gleam
// Equivalent to redraw.compose — use in place of it.
pub const compose: fn(
  redraw.ReactComponent(b),
  fn(fn(b) -> redraw.Element) -> redraw.ReactComponent(c),
) -> redraw.ReactComponent(c)

// For components that take Nil props — injects Nil automatically.
pub fn static(
  component: redraw.ReactComponent(Nil),
  next: fn(fn() -> redraw.Element) -> redraw.ReactComponent(a),
) -> redraw.ReactComponent(a)

// For components whose props are #(props, value) — unpacks the tuple.
pub fn children(
  component: redraw.ReactComponent(#(props, value)),
  next: fn(fn(props, value) -> redraw.Element) -> redraw.ReactComponent(a),
) -> redraw.ReactComponent(a)
```

Example:

```gleam
fn my_component() {
  use static_child <- compose.static(static_child())
  use list_child <- compose.children(list_child())
  use _props <- redraw.component_("MyComponent")
  h.div([], [
    static_child(),
    list_child(ItemProps(label: "hello"), [h.text("world")]),
  ])
}
```

### `redraw/batteries/error_boundary`

```gleam
pub type Error {
  Error(
    name: String,
    message: String,
    cause: option.Option(dynamic.Dynamic),
    stack: option.Option(String),
  )
}

pub type ErrorInfo {
  ErrorInfo(
    component_stack: option.Option(String),
    owner_stack: option.Option(String),
  )
}

pub type Props

pub fn children(redraw.Element) -> Props
pub fn fallback(Props, redraw.Element) -> Props
pub fn on_error(Props, fn(Error, ErrorInfo) -> Nil) -> Props
pub fn on_raw_error(Props, fn(dynamic.Dynamic, dynamic.Dynamic) -> Nil) -> Props
pub fn render(Props) -> redraw.Element
```

Example:

```gleam
error_boundary.children(my_app())
|> error_boundary.fallback(error_screen())
|> error_boundary.on_error(fn(err, _info) { log_error(err.message) })
|> error_boundary.render
```

---

## Rules

- **Never call hooks outside components or custom hooks.** React's rules of
  hooks apply.
- **Dependencies are tuples.** Pass `#(a, b)` for multiple deps, `Nil` for
  run-once, `#()` for always-run.
- **`compose` calls happen during bootstrap**, before `component_`. The
  `use x <- compose(...)` chain builds the component closure; `component_` is
  always the innermost call.
- **`create_context_` is called at module level**, not inside components.
- Props can be any Gleam type: records, tuples, or `Nil`.
