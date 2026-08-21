# logos-hub

Run a set of **logos-core modules** headless and drive them from a generated CLI — a thin,
ergonomic wrapper over `logoscore` (the Logos Core daemon/client runtime).

**📖 Docs:** <https://vpavlin.github.io/logos-hub/>

Nothing here is app-specific: it drives **any** logos-core module — `kym_core`, `qaku_core`,
`scala`, `loam_core`, `delivery_module`, … Loam apps are just some of the tenants. The value it
adds over raw `logoscore`:

- **Profiles** — a declarative set of modules + env + module-dir to bring up together, in an
  **isolated config dir** (so one hub never reads another's registry).
- **One-command bring-up** — `up` starts the daemon and `load-module`s the whole set (logoscore
  resolves each module's dependency chain automatically).
- **A generated view of each module's API** — `describe` surfaces the module's own methods from
  `logoscore module-info`, so you can see what's callable without reading source.
- **Ergonomics** — pretty-printed JSON (unwraps the `{"result": "<json>"}` envelope), `watch`
  for events, `logs`, `status`.

The heavy lifting (load-module / call / module-info / watch / access-policy) is logoscore's — this
just makes it pleasant and repeatable. A strong candidate to upstream into `logos-co`.

## Usage

```sh
logos-hub profiles                       # list profiles
logos-hub up   kym                       # start daemon + load kym_core (pulls loam_core, delivery)
logos-hub ls   kym                       # loaded/available modules
logos-hub describe kym kym_core          # kym_core's API (methods) from module-info
logos-hub call kym kym_core snapshot     # call a method → pretty JSON
logos-hub call kym loam_core metricsJson # drive the shared node's facade directly
logos-hub watch kym kym_core             # stream events
logos-hub logs kym                       # tail the daemon log
logos-hub down kym                       # stop the daemon
```

Drive the shared node on its own:

```sh
logos-hub up loam
logos-hub call loam loam_core start '{"mode":"Core","preset":"logos.test"}'
logos-hub call loam loam_core metricsJson
```

## Profiles

A profile is a JSON file under `./profiles/` (bundled) or `~/.logos-hub/profiles/`:

```json
{
  "name": "kym",
  "modulesDir": "/path/to/modules",
  "load": ["kym_core"],
  "env": { "KYM_HUB": "1", "KYM_CORE_DATA": "{data}" }
}
```

- `modulesDir` — a logoscore modules dir (portable `linux-amd64` bundles).
- `load` — modules to `load-module` on `up` (deps auto-resolve).
- `env` — environment for the daemon's modules; `{data}` expands to the profile's data dir.

## Config

- `LOGOSCORE` — path to the logoscore binary (default `~/logoscore-new/result/bin/logoscore`).
- `LOGOS_HUB_STATE` — where per-profile config/data/logs live (default `~/.logos-hub`).

Each profile runs in `~/.logos-hub/run/<name>/{cfg,data,daemon.log}` — isolated per profile.

## Typed CLI + completion (driven by the module's own API)

`call` accepts **named `--flags`**, coerced to JSON per each parameter's Qt type read live from
`logoscore module-info` — no hand-written JSON, and a bad flag is a clear error, not a silent
no-op:

```sh
logos-hub describe kym kym_core            # typed signatures: createBudget --name <QString>, …
logos-hub describe kym kym_core --json     # full machine-readable schema (params+returnType+events)
logos-hub call kym kym_core createBudget --name Groceries
logos-hub call kym kym_core addAccount --name Checking --type asset --balance 100
logos-hub call kym kym_core createBudget --name X --dry-run   # show the JSON, don't invoke
```

A raw JSON positional (`call … createBudget '{"name":"X"}'`) still works. Because it all reflects
`module-info` at runtime, adding a method to a core surfaces it automatically — nothing here changes.

**Bash completion** (dynamic: subcommands → profiles → modules → methods → `--param` flags):

```sh
source <(logos-hub completions)                                     # this shell
logos-hub completions | sudo tee /etc/bash_completion.d/logos-hub   # persistent
```

## MCP bridge — drive the hub from any agent

`logos-hub-mcp <profile>` is an MCP (JSON-RPC over stdio) server that turns **every invokable
method into an MCP tool** (`<module>__<method>`, with a JSON-schema `inputSchema` derived from the
Qt parameter types). Any MCP client — Claude, an agent runtime — then drives kym/qaku/scala/loam
with first-class **typed tool-calls**, no CLI string-mangling. Add a method to a core and it appears
on the next `tools/list` with zero code change.

```sh
logos-hub up kym                 # bring the daemon up first
logos-hub-mcp kym                # speaks MCP over stdio
```

MCP client config: command `logos-hub-mcp`, args `["kym"]`. The tool list is generated from the
loaded modules' live schema; `tools/call` runs the method and returns its JSON result.

## Relationship to `logos-logoscore-py`

logos-co ships [`logos-logoscore-py`](https://github.com/logos-co/logos-logoscore-py) — a Python
client over the same `logoscore` CLI (launch / load / call / subscribe). The generally-useful
layers here — the **typed CLI + completion** and the **module-method MCP**, both generated from
`module-info` — belong upstream on that client rather than in this fork. (Profiles stay local: a
real long-running hub is managed by systemd, not brought up per-invocation, so the reflective
CLI/MCP just **attach** to the running daemon.)

Attaching works today with no upstream change: a `LogoscoreClient(binary, config_dir, token)` built
against a running daemon's `<config_dir>/client/config.json` (+ token from `client/auto.json`) is a
live client — no `connect(endpoints=...)` needed. And `pip install git+https://…` installs it fine;
PyPI would just be polish. So the CLI + MCP are being contributed as a PR to `logos-logoscore-py`.
