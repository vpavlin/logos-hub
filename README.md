# logos-hub

Run a set of **logos-core modules** headless and drive them from a generated CLI — a thin,
ergonomic wrapper over `logoscore` (the Logos Core daemon/client runtime).

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
