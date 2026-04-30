---
name: 'docker-workflow'
description: 'Docker Build & Run — mxcli docker run/build/up/reload/down, hot reload, CSS reload, environment variables, and troubleshooting'
compatibility: opencode
---

<role>Mendix Docker operator — build, run, reload, and manage Mendix apps in Docker or Podman containers.</role>

<summary>
> Full workflow for building, running, and testing a Mendix application using Docker inside a devcontainer. Covers quick start, step-by-step workflow, hot reload, CSS reload, environment variables, and troubleshooting.
</summary>

<triggers>
Load when:
- User wants to run their Mendix app locally in Docker
- User wants to build a deployable container image
- User asks about hot reload or CSS reload
- User needs to set up the Docker workflow for the first time
</triggers>

<prerequisites>

The devcontainer created by `mxcli init` includes:
- **JDK 21** (Adoptium temurin-21-jdk) — required by MxBuild
- **Docker-in-Docker** (or **Podman-in-Podman**) — container runtime inside the devcontainer
- **Port forwarding** — ports 8080 (app) and 8090 (admin) auto-forwarded

To force Podman: `export MXCLI_CONTAINER_CLI=podman`

</prerequisites>

<quick_start>

```bash
# Setup, build, and start in one command
./mxcli docker run -p app.mpr

# With startup confirmation
./mxcli docker run -p app.mpr --wait

# Fresh start (removes database volumes first)
./mxcli docker run -p app.mpr --fresh --wait
```

`docker run` handles everything: downloads MxBuild and runtime (if not cached), initializes the Docker stack, builds the PAD package, starts the containers, and waits for startup.

</quick_start>

<creating_new_app>

```bash
# Recommended: all-in-one
./mxcli new MyApp --version 11.8.0

# Manual approach
./mxcli setup mxbuild --version 11.6.4
mkdir -p /path/to/my-app
~/.mxcli/mxbuild/{version}/modeler/mx create-project --app-name MyApp --output-dir /path/to/my-app
./mxcli init /path/to/my-app
```

Blank projects have no demo users — configure security via MDL or Studio Pro before logging in.

</creating_new_app>

<step_by_step_workflow>

### 1. Setup MxBuild and Runtime (first time only)

```bash
./mxcli setup mxbuild -p app.mpr
./mxcli setup mxruntime -p app.mpr
```

MxBuild is cached at `~/.mxcli/mxbuild/{version}/`, runtime at `~/.mxcli/runtime/{version}/`.

**If PAD build fails with `StudioPro.conf.hbs does not exist`:**

```bash
version=11.6.4
cp -r ~/.mxcli/runtime/$version/runtime/pad ~/.mxcli/mxbuild/$version/runtime/pad
cp -r ~/.mxcli/runtime/$version/runtime/lib ~/.mxcli/mxbuild/$version/runtime/lib
cp -r ~/.mxcli/runtime/$version/runtime/launcher ~/.mxcli/mxbuild/$version/runtime/launcher
cp -r ~/.mxcli/runtime/$version/runtime/agents ~/.mxcli/mxbuild/$version/runtime/agents
```

### 2. Initialize Docker stack (first time only)

```bash
./mxcli docker init -p app.mpr

# Check port conflicts, use offset if needed
ss -tlnp | grep -E '808|809|543'
./mxcli docker init -p app.mpr --port-offset 5
```

### 3. Check project for errors

```bash
./mxcli docker check -p app.mpr
```

### 4. Build the PAD package

```bash
./mxcli docker build -p app.mpr
./mxcli docker build -p app.mpr --skip-check   # skip pre-build check
```

### 5. Start the application

```bash
./mxcli docker up -p app.mpr                           # foreground
./mxcli docker up -p app.mpr --detach                  # background
./mxcli docker up -p app.mpr --detach --wait           # wait for startup
./mxcli docker up -p app.mpr --fresh                   # remove database volumes
./mxcli docker up -p app.mpr --detach --wait --wait-timeout 600
```

App: **http://localhost:8080** | Admin: **http://localhost:8090**

### 6. Monitor and manage

```bash
./mxcli docker status -p app.mpr
./mxcli docker logs -p app.mpr --follow
./mxcli docker shell -p app.mpr
./mxcli docker down -p app.mpr              # stop
./mxcli docker down -p app.mpr --volumes   # stop + remove DB volumes
```

</step_by_step_workflow>

<common_workflow>

After making MDL changes:

```bash
# Apply MDL changes
./mxcli exec changes.mdl -p app.mpr

# Rebuild and restart (one command)
./mxcli docker run -p app.mpr --fresh --wait
```

</common_workflow>

<hot_reload>

Because mxcli's Docker setup uses a **bind mount** (`.docker/build/` → `/mendix/`), rebuilt PAD output is immediately visible to the running runtime.

### Model Reload

```bash
./mxcli exec changes.mdl -p app.mpr                   # ~1s  — update model
./mxcli docker build -p app.mpr                        # ~55s — compile PAD
./mxcli docker reload -p app.mpr --model-only          # ~100ms — reload only
```

Or combined:

```bash
./mxcli exec changes.mdl -p app.mpr
./mxcli docker reload -p app.mpr --skip-check
```

### CSS-Only Reload

```bash
./mxcli docker build -p app.mpr            # compile SCSS into PAD (~55s)
./mxcli docker reload -p app.mpr --css     # push compiled CSS to browsers (~instant)
```

> **Note:** `--css` does NOT compile SCSS. If you skip the build step, the browser won't reflect SCSS changes.

### When to Use `reload` vs `run`

| Scenario | Command | Why |
|----------|---------|-----|
| Microflow/nanoflow/page logic | `docker reload` | No schema change, keeps data |
| CSS/theme changes only | `docker reload --css` | Instant, no MxBuild needed |
| New entity or attribute (additive) | `docker reload` | Runtime applies DDL on reload |
| Destructive schema change | `docker up --fresh` | Runtime can't apply destructive DDL |
| First-time setup | `docker run` | Need containers + database |
| Database corruption or reset | `docker run --fresh` | Recreates volumes |

### When Reload is NOT Sufficient

`reload_model` cannot handle destructive schema changes:
- Removed an entity or attribute
- Changed an attribute type (e.g., String → Integer)

Use `mxcli docker up -p app.mpr --fresh --detach --wait` in these cases.

</hot_reload>

<environment_variables>

| Variable | Default | Description |
|----------|---------|-------------|
| `ADMIN_ADMINPASSWORD` | `AdminPassword1!` | Admin console password |
| `RUNTIME_DEBUGGER_PASSWORD` | `AdminPassword1!` | Debugger password |
| `RUNTIME_PARAMS_DATABASETYPE` | `POSTGRESQL` | Database type |
| `RUNTIME_PARAMS_DATABASEHOST` | `db:5432` | Hostname and port |
| `RUNTIME_PARAMS_DATABASENAME` | `mendix` | Database name |
| `RUNTIME_PARAMS_DATABASEUSERNAME` | `mendix` | Database user |
| `RUNTIME_PARAMS_DATABASEPASSWORD` | `mendix` | Database password |
| `MX_LOG_LEVEL` | `info` | Log level |

All defaults can be overridden in `.docker/.env`.

</environment_variables>

<troubleshooting>

| Problem | Solution |
|---------|----------|
| `docker: command not found` | Rebuild devcontainer — docker-in-docker feature needs rebuild |
| `mxbuild not found` | Run `mxcli setup mxbuild -p app.mpr` |
| `StudioPro.conf.hbs does not exist` | Copy runtime into mxbuild — see step 1 above |
| `ClassNotFoundException: EventProcessor` | PAD has partial runtime — copy full runtime into `.docker/build/lib/runtime/` |
| Port already allocated | Use `mxcli docker init -p app.mpr --port-offset N --force` |
| `DatabasePassword has no value` | Re-run `mxcli docker init --force` |
| `password should not be empty (debugger)` | Re-run `mxcli docker init --force` |
| Database errors on startup | Try `mxcli docker up -p app.mpr --fresh` |
| OQL: "Action not found: preview_execute_oql" | Re-run `mxcli docker init --force` for updated docker-compose.yml |

</troubleshooting>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
