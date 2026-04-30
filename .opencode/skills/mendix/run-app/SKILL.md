---
name: 'run-app'
description: 'Run Mendix app in Docker: first run, rebuild after MDL changes, hot reload workflow, --fresh vs reload decision'
compatibility: opencode
---

<role>Run Mendix app in Docker: first run, rebuild after MDL changes, hot reload workflow, --fresh vs reload decision</role>

<summary>
> Run Mendix app in Docker: first run, rebuild after MDL changes, hot reload workflow, --fresh vs reload decision
</summary>

<triggers>
Load when:
- The user asks to run, start, or launch the app
- The user asks to rebuild and restart after making changes
- The user wants to test their changes in a running environment
</triggers>

<prerequisites_check>
Before running, verify:

1. **mxcli is local** — always use `./mxcli`, not `mxcli`
2. **Docker is available** — check with `docker ps`

Everything else (MxBuild, runtime, Docker stack) is auto-downloaded/initialized by `docker run`.

---
</prerequisites_check>

<quick_start_one_command>
```bash
./mxcli docker run -p MxCliDemoApp2.mpr --wait
```

This single command:
1. Downloads MxBuild (if not cached)
2. Downloads the Mendix runtime (if not cached)
3. Initializes the Docker stack (if not already done)
4. Builds the PAD package
5. Starts the containers in the background
6. Waits for "Runtime successfully started" (with `--wait`)

The app is available at: **http://localhost:8080**
Admin console: **http://localhost:8090** (password: `AdminPassword1!`)

---
</quick_start_one_command>

<after_making_mdl_changes>
**Model/page/security changes** (microflows, entities, pages, navigation):

```bash
./mxcli exec changes.mdl -p MxCliDemoApp2.mpr

./mxcli docker build -p MxCliDemoApp2.mpr --skip-check
./mxcli docker reload -p MxCliDemoApp2.mpr
```

Use `docker reload` for most changes — it reloads the model in ~100ms without restarting.
Use `docker run --fresh` only when schema changes are destructive (dropped entities/attributes):

```bash
./mxcli docker run -p MxCliDemoApp2.mpr --fresh --wait
```

**CSS/theme changes only** (no rebuild needed):

```bash
./mxcli docker reload -p MxCliDemoApp2.mpr --css
```

Or simply hard-refresh the browser (Cmd+Shift+R / Ctrl+Shift+R) — the volume mount means compiled CSS is immediately available on disk after `mxcli docker build`.

---
</after_making_mdl_changes>

<stepbystep_alternative>
If you prefer more control over individual steps:

```bash
./mxcli setup mxbuild -p MxCliDemoApp2.mpr
./mxcli setup mxruntime -p MxCliDemoApp2.mpr
./mxcli docker init -p MxCliDemoApp2.mpr

./mxcli docker build -p MxCliDemoApp2.mpr
./mxcli docker up -p MxCliDemoApp2.mpr --detach --wait
```

---
</stepbystep_alternative>

<query_data_oql>
Once the app is running, test OQL queries against the live runtime:

```bash
./mxcli oql -p MxCliDemoApp2.mpr "select Name from MyModule.Customer"

./mxcli oql -p MxCliDemoApp2.mpr --json "SELECT count(c.ID) FROM MyModule.Order AS c"
```

> **Existing projects**: If you get "Action not found: preview_execute_oql", re-initialize
> the Docker stack to pick up the required JVM flag:
> `./mxcli docker init -p MxCliDemoApp2.mpr --force`

---
</query_data_oql>

<monitoring>
```bash
./mxcli docker logs -p MxCliDemoApp2.mpr --follow

./mxcli docker logs -p MxCliDemoApp2.mpr --tail 50

./mxcli docker shell -p MxCliDemoApp2.mpr
```

Look for this line to confirm successful startup:
```
Mendix runtime successfully started, the application is now available.
```

---
</monitoring>

<stop>
```bash
./mxcli docker down -p MxCliDemoApp2.mpr

./mxcli docker down -p MxCliDemoApp2.mpr --volumes
```

---
</stop>

<running_multiple_projects>
When running multiple Mendix apps simultaneously, each project needs unique ports. Use `--port-offset` to shift all ports:

```bash
./mxcli docker run -p project1/app.mpr --wait

./mxcli docker init -p project2/app.mpr --port-offset 1
./mxcli docker run -p project2/app.mpr --wait
```

| Offset | App | Admin | DB |
|--------|-----|-------|----|
| 0 | 8080 | 8090 | 5432 |
| 1 | 8081 | 8091 | 5433 |
| 2 | 8082 | 8092 | 5434 |

The offset is applied once during `docker init` and written to `.docker/.env`. Subsequent `docker run/up/reload` commands read from that `.env` automatically.

> **Note:** If the Docker stack was already initialized, re-run init with `--force`:
> `./mxcli docker init -p app.mpr --port-offset 1 --force`

---
</running_multiple_projects>

<architecture_note>
The containers run inside a Docker-in-Docker daemon inside the devcontainer — they are **not** visible in Docker Desktop on the host. Port forwarding (8080, 8090) is handled by VS Code automatically.

The build output (`.docker/build/`) is **volume-mounted** into the container — no Docker image rebuild needed. After `mxcli docker build`:
- **CSS/theme changes**: Hard-refresh the browser — files are already on disk via the mount
- **Model changes**: Run `mxcli docker reload` — hot reloads the model in ~100ms, no restart needed
- **Destructive schema changes**: Restart with `mxcli docker up --fresh` to recreate the database
</architecture_note>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
