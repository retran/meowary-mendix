---
name: 'verify-with-oql'
description: 'Verify deployed changes via OQL: confirm microflow side effects, data assertions post-deploy, fast feedback loop'
compatibility: opencode
---

<role>Verify deployed changes via OQL: confirm microflow side effects, data assertions post-deploy, fast feedback loop</role>

<summary>
> Verify deployed changes via OQL: confirm microflow side effects, data assertions post-deploy, fast feedback loop
</summary>

<triggers>
Load when:
- You've deployed changes and want to verify data was created/updated/deleted correctly
- You need to confirm a microflow produced the expected side effects
- You're combining Playwright UI tests with backend data verification
- You want a fast feedback loop: deploy, trigger, verify
</triggers>

<the_pattern>
The core verification workflow is:

1. **Deploy** — apply MDL changes and rebuild
2. **Trigger** — execute the action (via UI, microflow call, or API)
3. **Verify** — query the database with OQL to confirm the result

```bash
./mxcli exec changes.mdl -p app.mpr
./mxcli docker build -p app.mpr --skip-check
./mxcli docker reload -p app.mpr   # or: ./mxcli docker up -p app.mpr --fresh --wait


./mxcli oql -p app.mpr "select Name, Email from MyModule.Customer where Name = 'Jane Doe'"
```
</the_pattern>

<oql_verification_examples>
### Check data was created

```bash
./mxcli oql -p app.mpr "select Name, Email from Sales.Customer where Name = 'Test Customer'"

./mxcli oql -p app.mpr "select count(*) as Total from Sales.Order"
```

### Check data was updated

```bash
./mxcli oql -p app.mpr "select OrderNumber, status from Sales.Order where OrderNumber = 'ORD-001'"
```

### Check data was deleted

```bash
./mxcli oql -p app.mpr "select count(*) as Total from Sales.Customer where Name = 'Deleted Customer'"
```

### Check associations

```bash
./mxcli oql -p app.mpr \
  "select o.OrderNumber, c.Name from Sales.Order o join o/Sales.Order_Customer/Sales.Customer c where o.OrderNumber = 'ORD-001'"
```

### JSON output for assertions

Use `--json` for structured output that's easy to parse in scripts:

```bash
./mxcli oql -p app.mpr --json "SELECT Name FROM Sales.Customer" | jq '.[].Name'

count=$(mxcli oql -p app.mpr --json "SELECT count(*) AS Total FROM Sales.Order" | jq -r '.[0].Total')
if [ "$count" -gt 0 ]; then
  echo "Orders exist: $count"
fi
```
</oql_verification_examples>

<combining_with_playwright>
The most powerful pattern: trigger actions through the UI with Playwright, then verify side effects with OQL.

### Example: Create via UI, verify via OQL

```typescript
// tests/verify-create.spec.ts
import { test, expect } from '@playwright/test';
import { execSync } from 'child_process';
import { login } from './utils/login';

function oql(query: string): any[] {
  const result = execSync(
    `mxcli oql -p app.mpr --json "${query}"`,
    { encoding: 'utf-8' }
  );
  return JSON.parse(result);
}

test('creating a customer via UI persists correctly', async ({ page }) => {
  await login(page);
  await page.goto('/p/Customer_Edit');

  // Fill the form
  await page.locator('.mx-name-txtName input').fill('OQL Test Customer');
  await page.locator('.mx-name-txtEmail input').fill('oql@test.com');
  await page.locator('.mx-name-btnSave').click();

  // wait for save to complete
  await page.waitForTimeout(2000);

  // Verify via OQL
  const rows = oql("select Name, Email from Sales.Customer where Name = 'OQL Test Customer'");
  expect(rows).toHaveLength(1);
  expect(rows[0].Email).toBe('oql@test.com');
});
```

### Example: Verify microflow side effects

```typescript
test('approving an order updates status and creates audit log', async ({ page }) => {
  await login(page);
  await page.goto('/p/Order_Overview');

  // Click approve on first order
  await page.locator('.mx-name-btnApprove').first().click();
  await page.waitForTimeout(2000);

  // Verify order status changed
  const orders = oql("select status from Sales.Order where OrderNumber = 'ORD-001'");
  expect(orders[0].Status).toBe('Approved');

  // Verify audit log was created
  const logs = oql("select action from Sales.AuditLog where action = 'Order Approved'");
  expect(logs.length).toBeGreaterThan(0);
});
```
</combining_with_playwright>

<combining_with_hot_reload>
For the fastest iteration loop when developing microflow logic:

```bash
./mxcli exec fix-logic.mdl -p app.mpr

./mxcli docker build -p app.mpr --skip-check

./mxcli docker reload -p app.mpr


./mxcli oql -p app.mpr "select status from Sales.Order where OrderNumber = 'ORD-001'"

```

This loop avoids container restarts and database resets, making each iteration take seconds instead of minutes.
</combining_with_hot_reload>

<tips>
- **Use `--json` for scripted assertions** — structured output is easier to parse than table format
- **OQL is read-only** — `mxcli oql` uses the `preview_execute_oql` action which cannot modify data
- **Check before and after** — query the state before triggering an action to establish a baseline
- **Common OQL patterns for testing:**
  - `count(*)` to verify record counts
  - `where` clauses to find specific records
  - `join` to verify associations were set
  - `ORDER by ... limit 1` to check the most recent record
- **`--direct` mode** is faster when the admin port is reachable (after `admin.addresses` build patch)
</tips>

<related_skills>
- [/test-app](./test-app.md) — Full Playwright test patterns and setup
- [/runtime-admin-api](./runtime-admin-api.md) — Admin API details and curl examples
- [/docker-workflow](./docker-workflow.md) — Build, run, and hot reload workflow
- [/write-oql-queries](./write-oql-queries.md) — OQL syntax reference
</related_skills>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
