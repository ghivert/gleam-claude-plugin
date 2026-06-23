---
name: gleam-project
description: >-
  Gleam project structure — directory layout, modules, import paths, testing,
  and full gleam.toml reference (config, dependencies, targets, publishing)
user-invocable: false
---

# Gleam Projects

## Directory layout

Running `gleam new <project_name>` scaffolds:

```
.
├── gleam.toml          # project config & dependencies
├── manifest.toml       # locked dependency versions (auto-generated, commit this)
├── src/<project_name>.gleam    # entry point with main()
└── test/<project_name>_test.gleam
```

Additional directories:

- `src/` — all application source files
- `test/` — test files (dev-only, not published)
- `dev/` — development helpers (not published)

## Modules

- Source files in `src/` map directly to module names: `src/foo/bar.gleam` →
  module `foo/bar`
- Functions are **private by default**; use `pub` to export
- Internal modules: place under `<package>/internal` or `<package>/internal/*`
  and declare them in `gleam.toml`'s `internal_modules` — they're accessible
  within the package but have no public API stability guarantee

## Testing

Uses `gleeunit`. Any public function ending in `_test` is auto-discovered.

```sh
gleam test
```

Test files live in `test/`. Dev dependencies (e.g. `gleeunit`) go in
`[dev_dependencies]` in `gleam.toml` and are excluded from published packages.

---

## gleam.toml Reference

### Required fields

```toml
name = "my_project"   # snake_case
version = "1.0.0"     # semver
```

### Publishing to Hex (also required when publishing)

```toml
description = "A parser for the PureData file format"
licences = ["Apache-2.0"]   # SPDX identifiers

[repository]
type = "github"   # bitbucket | codeberg | github | gitlab | sourcehut | tangled | forgejo | gitea | custom
user = "username"
repo = "repo_name"
# For gitea/forgejo: also set host = "..."
# For custom: set url = "..."
# Monorepo fields: path = "...", tag_prefix = "..."

[[links]]
title = "Website"
href  = "https://example.com"
```

### Dependencies

```toml
[dependencies]
gleam_stdlib = ">= 1.0.0 and < 2.0.0"
my_lib       = { path = "../my_lib" }      # local path dep
my_lib       = { git = "git@github.com:user/repo", ref = "<commit-hash>" }  # git dep

[dev_dependencies]
gleeunit = ">= 1.0.0"
# dev_dependencies are excluded from Hex; a package cannot appear in both sections
```

CLI shortcuts:

```sh
gleam add envoy argv        # adds to [dependencies]
gleam add --dev gleeunit    # adds to [dev_dependencies]
gleam remove <pkg>
```

### Compilation

```toml
gleam  = ">= 1.15.0"   # minimum required compiler version
target = "erlang"      # default target: "erlang" | "javascript" (overridable via CLI --target)

internal_modules = ["my_project/internal", "my_project/internal/*"]
```

### Erlang target

```toml
[erlang]
application_start_module = "my_project@application"  # OTP start module
extra_applications = ["inets", "ssl"]                 # additional OTP apps to start
```

### JavaScript target

```toml
[javascript]
source_maps             = false   # emit .map files
typescript_declarations = false   # emit .d.ts files (source of truth for TS FFI)
runtime                 = "node"  # "node" | "deno" | "bun"

[javascript.deno]
allow_all   = false
allow_env   = ["HOME"]   # can also be boolean; same pattern for:
# allow_ffi, allow_hrtime, allow_net, allow_read, allow_run, allow_sys, allow_write
```

### Documentation pages

```toml
[[documentation.pages]]
title  = "Guide"
path   = "/guide.html"
source = "docs/guide.md"
```

### External tools config

```toml
[tools.my_tool]
# arbitrary config consumed by external tools — keep tool config here, not in separate files
```

---

## References

- `gleam.toml` spec: https://gleam.run/writing-gleam/gleam-toml
- Package index: https://packages.gleam.run/
- Language tour: https://tour.gleam.run/
