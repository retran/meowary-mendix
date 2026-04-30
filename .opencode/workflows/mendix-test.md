---
updated: 2026-04-29
tags: [mendix]
---

<role>
Mendix app tester. Verify the running Docker app behaves correctly using playwright-cli for UI interaction and mxcli OQL for data assertions.
</role>

<summary>
Verify a running Mendix application in Docker using playwright-cli (UI) and `./mxcli oql` (data). Covers: app start, login, page navigation, widget verification, data persistence assertions, and regression test scripts. Always requires the app to be running in Docker before starting.
</summary>

<inputs>
| Input | Source | Required |
|-------|--------|----------|
| What to verify | User invocation | Required |
| Codebase context | `codebases/<project>.md` | Required |
| Running app | Docker (`./mxcli docker run -p <project>.mpr --wait`) | Required |
</inputs>

<steps>

<step n="1" name="Start the app">
Verify app is running. If not:

```bash
# Normal start — waits until "runtime successfully started"
./mxcli docker run -p <project>.mpr --wait

# Fresh start — wipes database volumes (use for clean-state testing only)
./mxcli docker run -p <project>.mpr --fresh --wait
```

App available at: `http://localhost:8080` | Admin: `http://localhost:8090`

**Docker architecture:** `.docker/build/` is volume-mounted into the container at `/mendix`. After executing MDL changes, hot-reload or restart the container (no Docker image rebuild needed):
```bash
# Hot reload — keeps database, ~100ms
./mxcli docker reload -p <project>.mpr --model-only

# Full restart — use when DDL changes (new entities/attributes)
./mxcli docker run -p <project>.mpr --wait
```
<done_when>App running and responding at localhost:8080.</done_when>
</step>

<step n="2" name="Login (if security enabled)">

```bash
playwright-cli open http://localhost:8080
playwright-cli run-code "document.querySelector('#usernameInput').value = 'MxAdmin'"
playwright-cli run-code "document.querySelector('#passwordInput').value = 'AdminPassword1!'"
playwright-cli run-code "document.querySelector('#loginButton').click()"
playwright-cli run-code "await new Promise(r => setTimeout(r, 3000))"

# Save auth state for reuse
playwright-cli state-save mendix-auth

# Restore in a later session
playwright-cli state-load mendix-auth
```
<done_when>Logged in; auth state saved.</done_when>
</step>

<step n="3" name="Verify UI">

```bash
# Take DOM snapshot
playwright-cli open http://localhost:8080
playwright-cli snapshot

# Verify widget exists — .mx-name-* selectors are stable (derived from MDL widget names)
playwright-cli run-code "document.querySelector('.mx-name-widgetName') !== null"

# Assert a value (throw = test failure)
playwright-cli run-code "
  const val = document.querySelector('.mx-name-textBoxName input').value;
  if (val !== 'Expected Value') throw new Error('Got: ' + val);
"

# Screenshot
playwright-cli screenshot

# Click a button
playwright-cli run-code "document.querySelector('.mx-name-buttonName').click()"
playwright-cli run-code "await new Promise(r => setTimeout(r, 1000))"
```

Always use `.mx-name-*` selectors — never positional or CSS class selectors that may change.
<done_when>UI state verified; screenshots taken if needed.</done_when>
</step>

<step n="4" name="Verify data (OQL)">

```bash
# Query entity data
./mxcli oql -p <project>.mpr --json "SELECT Name FROM MyModule.Customer"

# Check record count
./mxcli oql -p <project>.mpr "SELECT COUNT(*) FROM MyModule.Customer"

# Filter
./mxcli oql -p <project>.mpr --json "SELECT Name, Email FROM MyModule.Customer WHERE IsActive = true"
```

Compare results against expected values to assert data persistence.
<done_when>Data assertions pass.</done_when>
</step>

<step n="5" name="Run regression test scripts" condition="scripts exist">

```bash
# Run single script
bash tests/verify-customers.sh

# Run all scripts
for f in tests/verify-*.sh; do bash "$f" || exit 1; done
```

To capture current verification as a reusable script, write steps to `tests/verify-<feature>.sh`.
<done_when>All scripts pass; failures reported.</done_when>
</step>

<step n="6" name="Close browser">

```bash
playwright-cli close
```
<done_when>Browser closed.</done_when>
</step>

<step n="7" name="Report">
Report: what was verified, pass/fail per check, screenshots captured, any unexpected behavior. If failures: describe actual vs expected state and suggest next step.
<done_when>Results reported.</done_when>
</step>

</steps>

<error_handling>
- **App not running:** Start with `./mxcli docker run -p <project>.mpr --wait`.
- **Login fails:** Check security is enabled; check credentials.
- **Widget selector not found:** Verify widget name in MDL; use `playwright-cli snapshot` to inspect DOM.
- **OQL fails:** Verify entity/attribute names via `DESCRIBE ENTITY Module.Name`.
- **App crashes on start:** Check logs: `./mxcli docker logs -p <project>.mpr`
- **playwright-cli not installed:** Included in devcontainer — check devcontainer setup.
</error_handling>

<contracts>
1. NEVER test against a stopped app.
2. Use `.mx-name-*` selectors only.
3. Use `throw new Error()` for assertions — never swallow failures.
4. ALWAYS close browser after testing.
5. `--fresh` wipes the database — confirm with user before using in non-disposable environments.
</contracts>
