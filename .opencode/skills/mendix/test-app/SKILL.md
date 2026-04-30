---
name: 'test-app'
description: 'End-to-end browser testing with Playwright: UI rendering, form interactions, data persistence verification'
compatibility: opencode
---

<role>End-to-end browser testing with Playwright: UI rendering, form interactions, data persistence verification</role>

<summary>
> End-to-end browser testing with Playwright: UI rendering, form interactions, data persistence verification
</summary>

<triggers>
Load when:
- The user asks to test, verify, or validate a running Mendix app **in the browser**
- The user wants to confirm that generated pages and widgets actually render
- The user asks for end-to-end or integration tests involving the UI
- The user wants to verify that data is persisted correctly after UI interactions
- You have generated MDL that creates pages and want to close the feedback loop
</triggers>

<prerequisites>
The devcontainer created by `mxcli init` includes:
- **Node.js** (LTS) — installed via devcontainer feature
- **playwright-cli** — installed globally (`npm install -g @playwright/cli@latest`)
- **Chromium** — installed via `playwright-cli install --with-deps chromium`
- **Docker-in-Docker** — Mendix + PostgreSQL running via `mxcli docker run`

The app must be running before verification:

```bash
./mxcli docker run -p app.mpr --wait
```

---
</prerequisites>

<quick_start>
```bash
playwright-cli open http://localhost:8080

playwright-cli snapshot

playwright-cli click e12
playwright-cli fill e15 "some text"

playwright-cli run-code "document.querySelector('.mx-name-dgCustomers') !== null"

playwright-cli screenshot

playwright-cli close
```

---
</quick_start>

<widget_name_selectors>
Mendix renders each widget's `name` property as a CSS class on the DOM element:

```html
<div class="mx-name-submitButton form-group">
```

This maps directly to MDL widget names. When you generate a widget in MDL:

```sql
actionbutton submitButton (caption: 'Submit', action: save_changes)
```

The stable CSS selector is `.mx-name-submitButton`. Use this with `run-code` for reliable assertions:

```bash
playwright-cli run-code "document.querySelector('.mx-name-submitButton') !== null"
```

---
</widget_name_selectors>

<verification_patterns>
### Login (Security Enabled)

The Mendix login page uses standard HTML IDs:

```bash
playwright-cli open http://localhost:8080
playwright-cli snapshot
playwright-cli run-code "document.querySelector('#usernameInput').value = 'MxAdmin'"
playwright-cli run-code "document.querySelector('#passwordInput').value = 'AdminPassword1!'"
playwright-cli run-code "document.querySelector('#loginButton').click()"

playwright-cli run-code "await new Promise(r => setTimeout(r, 3000))"
playwright-cli snapshot

playwright-cli state-save mendix-auth
```

To reuse saved auth in a later session:
```bash
playwright-cli open http://localhost:8080
playwright-cli state-load mendix-auth
playwright-cli goto http://localhost:8080/p/Customer_Overview
```

**When security is OFF**: Skip login entirely. Navigate directly to `/`.

### Widget Presence Verification

After navigating to a page, verify that all expected widgets are present:

```bash
playwright-cli goto http://localhost:8080/p/Customer_Overview

playwright-cli run-code "document.querySelector('.mx-name-dgCustomers') !== null"
playwright-cli run-code "document.querySelector('.mx-name-btnNew') !== null"
playwright-cli run-code "document.querySelector('.mx-name-btnEdit') !== null"
playwright-cli run-code "document.querySelector('.mx-name-btnDelete') !== null"
```

### Form Interaction

```bash
playwright-cli goto http://localhost:8080/p/Customer_Edit

playwright-cli snapshot

playwright-cli run-code "document.querySelector('.mx-name-txtName input').value = 'Test Customer'"
playwright-cli run-code "document.querySelector('.mx-name-txtName input').dispatchEvent(new event('input', {bubbles: true}))"
playwright-cli run-code "document.querySelector('.mx-name-txtEmail input').value = 'test@example.com'"
playwright-cli run-code "document.querySelector('.mx-name-txtEmail input').dispatchEvent(new event('input', {bubbles: true}))"

playwright-cli fill e42 "Test Customer"
playwright-cli fill e45 "test@example.com"

playwright-cli run-code "document.querySelector('.mx-name-btnSave').click()"
```

### Page Navigation (Security OFF)

When security is OFF, direct `/p/PageName` URLs **do not work** — Mendix redirects to the home page. Navigate through your own named widgets instead:

```bash
playwright-cli open http://localhost:8080

playwright-cli run-code "await new Promise(r => { const check = () => document.querySelector('.mx-page') ? r() : setTimeout(check, 500); check(); })"

playwright-cli run-code "document.querySelector('.mx-name-btnCustomers').click()"

playwright-cli run-code "await new Promise(r => setTimeout(r, 2000))"
playwright-cli run-code "document.querySelector('.mx-name-dgCustomers') !== null"
```

### Page Navigation (Security ON)

Direct URLs work after login:

```bash
playwright-cli state-load mendix-auth
playwright-cli goto http://localhost:8080/p/Customer_Overview
playwright-cli run-code "document.querySelector('.mx-name-dgCustomers') !== null"
```

### Data Assertions via OQL

After a UI interaction, verify data persistence using `mxcli oql` (no `pg` package needed):

```bash
./mxcli oql -p app.mpr --json "SELECT Name, Email FROM MyModule.Customer WHERE Name = 'Test Customer'"
```

This returns JSON that you can inspect directly. No npm dependencies required.

---
</verification_patterns>

<cicd_test_scripts>
For regression testing in CI/CD, capture playwright-cli commands as shell scripts. These are the same commands used interactively — readable without TypeScript knowledge.

### Script Format

```bash
#!/usr/bin/env bash
set -euo pipefail

playwright-cli open http://localhost:8080
playwright-cli run-code "document.querySelector('#usernameInput').value = 'MxAdmin'"
playwright-cli run-code "document.querySelector('#passwordInput').value = 'AdminPassword1!'"
playwright-cli run-code "document.querySelector('#loginButton').click()"
playwright-cli run-code "await new Promise(r => setTimeout(r, 3000))"

playwright-cli goto http://localhost:8080/p/Customer_Overview
playwright-cli run-code "if (!document.querySelector('.mx-name-dgCustomers')) throw new error('dgCustomers not found')"
playwright-cli run-code "if (!document.querySelector('.mx-name-btnNew')) throw new error('btnNew not found')"

playwright-cli run-code "document.querySelector('.mx-name-btnNew').click()"
playwright-cli run-code "await new Promise(r => setTimeout(r, 2000))"
playwright-cli fill txtName "CI Test Customer"
playwright-cli fill txtEmail "ci@test.com"
playwright-cli run-code "document.querySelector('.mx-name-btnSave').click()"
playwright-cli run-code "await new Promise(r => setTimeout(r, 2000))"

./mxcli oql -p app.mpr --json "SELECT Name FROM MyModule.Customer WHERE Name = 'CI Test Customer'" \
  | grep -q "CI Test Customer"

playwright-cli close
echo "PASS: verify-customers"
```

### Running Scripts

```bash
bash tests/verify-customers.sh

for f in tests/verify-*.sh; do bash "$f" || exit 1; done

```

### Assertion Pattern

For `set -e` scripts, use `throw new error()` to trigger non-zero exit:

```bash
playwright-cli run-code "if (!document.querySelector('.mx-name-widgetName')) throw new error('missing widgetName')"
```

---
</cicd_test_scripts>

<session_management>
playwright-cli maintains browser sessions across commands. The devcontainer sets `PLAYWRIGHT_CLI_SESSION=mendix-app` by default.

```bash
playwright-cli list

playwright-cli close

playwright-cli close-all

playwright-cli -s=test2 open http://localhost:8080
```

---
</session_management>

<debugging>
```bash
playwright-cli screenshot

playwright-cli screenshot e42

playwright-cli open http://localhost:8080 --headed

playwright-cli console

playwright-cli network

playwright-cli tracing-start
playwright-cli tracing-stop

playwright-cli show
```

---
</debugging>

<selector_rules>
**Use `.mx-name-*` selectors from your own MDL widgets.** These are reliable and predictable because you control the widget names:

```sql
-- MDL: names you define become test hooks
actionbutton btnDrivers (caption: 'Drivers', action: show_page Module.Drivers_Overview)
datagrid dgOrders (datasource: database Module.Order) { ... }
```

```bash
playwright-cli run-code "document.querySelector('.mx-name-btnDrivers').click()"
playwright-cli run-code "document.querySelector('.mx-name-dgOrders') !== null"
```

**Do NOT guess CSS selectors for Mendix built-in layout widgets.** The top navigation bar, sidebar, header, and other platform UI elements have unpredictable class names.

**NavigationList items need `text_` prefix.** The `<li>` container does NOT get an `mx-name-*` class. The inner `<span>` gets `mx-name-text_<itemName>`:

```bash
playwright-cli run-code "document.querySelector('.mx-name-text_itemDrivers').click()"
```

**DataGrid2 rows**: Both header and data rows share `role="row"`. Filter with `:has([role="gridcell"])`:

```bash
playwright-cli run-code "document.querySelector('.mx-name-dgCustomers [role=\"row\"]:has([role=\"gridcell\"])').textContent"
```

---
</selector_rules>

<known_gotchas>
### Never use `waitForLoadState('networkidle')`
Mendix maintains a permanent long-polling XHR connection. `networkidle` never fires. Use element-based waits via `run-code` instead.

### Top navigation clicks intercepted
Clicking top nav items may fail due to `div.mx-placeholder` overlay. Use `dispatchEvent('click')`:

```bash
playwright-cli run-code "document.querySelector('.mx-name-navigationTree1-1').dispatchEvent(new event('click', {bubbles: true}))"
```

### Login page selectors are stable
The Mendix login page (`/login.html`) uses fixed IDs: `#usernameInput`, `#passwordInput`, `#loginButton`. These are stable across Mendix versions.

---
</known_gotchas>

<feedback_loop_workflow>
The key workflow: generate MDL → build → verify → fix → repeat.

```bash
./mxcli exec changes.mdl -p app.mpr

./mxcli docker run -p app.mpr --fresh --wait

playwright-cli open http://localhost:8080
playwright-cli snapshot

```

### Interpreting Failures

| Failure Type | What It Means | MDL Fix |
|-------------|---------------|---------|
| `.mx-name-X` not found | Widget X missing from DOM | Check widget nesting, container visibility, BSON structure |
| Page returns 500 | Runtime error on page load | Check page layout, datasource, parameter bindings |
| Page returns 404 | Page doesn't exist or wrong URL | Verify page qualified name and navigation |
| OQL returns empty | Microflow didn't commit | Check COMMIT statement, error handling in microflow |
| Console error | JavaScript error in widget | Check widget template, pluggable widget config |

---
</feedback_loop_workflow>

<related_skills>
- [test-microflows.md](./test-microflows.md) - **MDL microflow tests** (business logic, no browser needed)
- [/run-app](./run-app.md) - Build and start the Mendix app in Docker
- [/docker-workflow](./docker-workflow.md) - Full Docker workflow reference
- [/demo-data](./demo-data.md) - Seed test data into PostgreSQL
- [/create-page](./create-page.md) - Page creation patterns (widget names for selectors)
- [/write-microflows](./write-microflows.md) - Microflow patterns (data persistence logic)
</related_skills>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
