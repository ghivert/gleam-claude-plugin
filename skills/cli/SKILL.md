---
name: gleam-cli
description: >-
  Run Gleam CLI commands — build, test, run, add/remove dependencies, publish to
  Hex, format code, and scaffold new projects
user-invocable: true
---

# Gleam CLI Skill

Call the `gleam` command in bash to perform Gleam tasks. If `gleam` is not
found, warn the user and point them to https://gleam.run/install/.

When the user asks for a specific task via `$ARGUMENTS`, map their request to
the appropriate subcommand below.

## Commands

| Command                                 | What it does                                        |
| --------------------------------------- | --------------------------------------------------- |
| `gleam new <name>`                      | Scaffold a new project                              |
| `gleam build`                           | Compile the project                                 |
| `gleam check`                           | Type-check only (no output artifacts)               |
| `gleam run`                             | Compile and run `main`                              |
| `gleam dev`                             | Run the development entrypoint                      |
| `gleam test`                            | Run tests                                           |
| `gleam format`                          | Format source files in-place                        |
| `gleam fix`                             | Rewrite deprecated Gleam code                       |
| `gleam add <pkg>`                       | Add a runtime dependency                            |
| `gleam add --dev <pkg>`                 | Add a dev-only dependency                           |
| `gleam remove <pkg>`                    | Remove a dependency                                 |
| `gleam deps download`                   | Download all dependencies                           |
| `gleam update`                          | Update deps to latest compatible versions           |
| `gleam docs build`                      | Render HTML docs locally                            |
| `gleam publish`                         | Publish to Hex                                      |
| `gleam export erlang-shipment`          | Export a self-contained Erlang release              |
| `gleam export javascript-prelude`       | Export the JS prelude file                          |
| `gleam export hex-tarball`              | Build the tarball that `gleam publish` would upload |
| `gleam export package-interface`        | Machine-readable JSON of the public API             |
| `gleam hex retire <pkg> <ver> <reason>` | Retire a published Hex version                      |
| `gleam hex unretire <pkg> <ver>`        | Un-retire a published Hex version                   |
| `gleam lsp`                             | Start the language server (editors call this)       |
| `gleam shell`                           | Start an Erlang REPL with project code loaded       |
| `gleam clean`                           | Delete build artifacts                              |
| `gleam --version`                       | Print the installed Gleam version                   |

## Common flags

```sh
gleam run --target javascript    # compile and run on JS target
gleam run --target erlang        # compile and run on Erlang target (default)
gleam run -m some_module         # run a specific module's main/0
gleam build --target javascript
```

## Tips

- `gleam check` is faster than `gleam build` when you only need type feedback.
- `gleam fix` is safe to run repeatedly; it only rewrites deprecated syntax.
- Use `gleam <command> --help` for flags not listed here.
