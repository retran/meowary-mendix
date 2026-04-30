---
name: 'overview-pages'
description: 'Build CRUD page sets in Mendix MDL: navigation snippets, overview pages with DataGrid, NewEdit forms, and circular dependency patterns'
compatibility: opencode
---

<role>
CRUD page pattern expert for Mendix. Covers navigation snippets, overview pages, NewEdit forms, DataGrid syntax, and circular dependency resolution.
</role>

<summary>
Standard pattern for creating CRUD (Create, Read, Update, Delete) pages in Mendix using MDL syntax. Consists of a navigation snippet, overview page with DataGrid, and NewEdit form.
</summary>

<triggers>
- Create an overview page listing entity records
- Create a NewEdit form page for an entity
- Build a navigation snippet
- Set up a standard CRUD page set
- Resolve circular dependencies between snippets and pages
</triggers>

<pattern_summary>
| Component | Type | Purpose | Key Widgets |
|-----------|------|---------|-------------|
| `Entity_Menu` | Snippet | Vertical sidebar navigation | NAVIGATIONLIST with ITEM actions |
| `Entity_Overview` | Page | List all records | SNIPPETCALL (sidebar), DATAGRID, Heading |
| `Entity_NewEdit` | Page | Create/Edit form | DataView, Input widgets, Save/Cancel |
</pattern_summary>

<navigation_menu_snippet>
```sql
create snippet Module.Entity_Menu
{
  navigationlist navMenu {
    item itemCustomers (caption: 'Customers', action: show_page Module.Customer_Overview)
    item itemOrders (caption: 'Orders', action: show_page Module.Order_Overview)
    item itemProducts (caption: 'Products', action: show_page Module.Product_Overview)
  }
}
```

### Snippet Syntax

```sql
create [or replace] snippet Module.SnippetName
[(
  params: { $ParamName: Module.EntityType }
)]
[folder 'path']
{
  -- Widget definitions (same as pages)
}
```

### NAVIGATIONLIST Syntax

```sql
navigationlist widgetName {
  item itemName (caption: 'Caption', action: show_page Module.PageName)
  item itemName (caption: 'Caption', action: microflow Module.MicroflowName)
  item itemName (caption: 'Caption', action: close_page)
}
```
</navigation_menu_snippet>

<overview_page_template>
```sql
create page Module.Entity_Overview
(
  title: 'Entity Overview',
  layout: Atlas_Core.Atlas_Default,
  folder: 'OverviewPages'
)
{
  layoutgrid mainGrid {
    row row1 {
      column colNav (desktopwidth: 2) {
        snippetcall navMenu (snippet: Module.Entity_Menu)
      }
      column colContent (desktopwidth: 10) {
        dynamictext heading (content: 'Entities', rendermode: H2)
        datagrid EntityGrid (datasource: database Module.Entity) {
          column colName (attribute: Name, caption: 'Name')
          column colDescription (attribute: description, caption: 'Description')
        }
      }
    }
  }
}
```

### SNIPPETCALL Syntax

```sql
-- Simple snippet call
snippetcall widgetName (snippet: Module.SnippetName)

-- With parameters (for parameterized snippets):
snippetcall widgetName (snippet: Module.SnippetName, params: {Customer: $Customer})
```

### DATAGRID Syntax

```sql
datagrid GridName (
  datasource: database from Module.Entity where [IsActive = true] sort by Name asc,
  selection: single|multiple|none
) {
  column colName (attribute: attributename, caption: 'Label')
  column colCustom (caption: 'Custom') {
    -- Nested widgets (ACTIONBUTTON, LINKBUTTON, DYNAMICTEXT)
  }
}
```

**Properties:**
- `datasource: database from Module.Entity` - Entity data source (required)
- `where [condition]` - Optional XPath filter
- `sort by attr asc|desc` - Optional sorting
- `selection: single|multiple|none` - Optional selection mode

**Column Properties (non-default only in DESCRIBE output):**

| Property | Values | Default |
|----------|--------|---------|
| `Sortable` | `true`/`false` | `true` (with attribute) |
| `Resizable` | `true`/`false` | `true` |
| `Hidable` | `yes`/`hidden`/`no` | `yes` |
| `ColumnWidth` | `autofill`/`autoFit`/`manual` | `autofill` |
</overview_page_template>

<newedit_page_template>
```sql
create page Module.Entity_NewEdit
(
  params: { $entity: Module.Entity },
  title: 'Edit Entity',
  layout: Atlas_Core.PopupLayout,
  folder: 'OverviewPages'
)
{
  layoutgrid mainGrid {
    row row1 {
      column col1 (desktopwidth: autofill) {
        dataview dataView1 (datasource: $entity) {
          -- Input fields for each attribute
          textbox txtName (label: 'Name', attribute: Name)
          textbox txtDescription (label: 'Description', attribute: description)
          datepicker dpDueDate (label: 'Due Date', attribute: DueDate)
          combobox cbStatus (label: 'Status', attribute: status)

          footer footer1 {
            actionbutton btnSave (caption: 'Save', action: save_changes, buttonstyle: success)
            actionbutton btnCancel (caption: 'Cancel', action: cancel_changes)
          }
        }
      }
    }
  }
}
```

### Page Parameter Syntax

```sql
create page Module.PageName
(
  params: { $ParamName: Module.EntityName },
  title: '...',
  layout: ...
)
```

- Parameter name conventionally matches the entity name (e.g., `$store`, `$Customer`)
- DataView binding references this parameter (`datasource: $ParamName`)
</newedit_page_template>

<widget_selection_guide>
| Attribute Type | Widget | Example |
|----------------|--------|---------|
| String | `textbox` | Name, Description |
| String (long) | `textarea` | Comments, Notes |
| Integer, Long, Decimal | `textbox` | Price, Quantity |
| Boolean | `checkbox` or `radiobuttons` | IsActive, IsPublished |
| DateTime | `datepicker` | DueDate, OrderDate |
| Enumeration | `combobox` or `radiobuttons` | Status, Type |
| Association (reference) | `combobox` with DataSource | Category, Owner |

**Note:** `dropdown` is deprecated. Use `combobox` for enumeration attributes.

**ComboBox modes:**
- Enum mode: `combobox cb (label: 'status', attribute: status)`
- Association mode: `combobox cb (label: 'Customer', attribute: Order_Customer, datasource: database MyModule.Customer, CaptionAttribute: Name)`

**Reserved Attribute Names:** Do not use `CreatedDate`, `ChangedDate`, `owner`, `ChangedBy` — these are system attributes automatically added to all entities.
</widget_selection_guide>

<button_styles>
| Style | Use Case | Color |
|-------|----------|-------|
| `success` | Save, Confirm | Green |
| `default` | Cancel, Back | Gray |
| `primary` | Primary action | Blue |
| `danger` | Delete | Red |
| `warning` | Caution actions | Yellow |
</button_styles>

<circular_dependency_pattern>
When a navigation snippet references pages (via `show_page`) and those pages reference the snippet (via `snippetcall`), use the **placeholder pattern**:

### Creation Order

1. **Create placeholder snippet first** (before pages)
2. **Create all pages** (which reference the snippet via SNIPPETCALL)
3. **Replace snippet with full content** (which can now reference existing pages)

### Example Pattern

```sql
-- Step 1: Create placeholder snippet (pages can reference this)
create snippet Module.NavigationMenu
{
  layoutgrid navGrid {
    row row1 {
      column col1 (desktopwidth: 12) {
        dynamictext loading (content: 'Loading...')
      }
    }
  }
}
/

-- Step 2: Create all pages (they reference the snippet via SNIPPETCALL)
create page Module.Customer_NewEdit
(
  params: { $Customer: Module.Customer },
  title: 'Edit Customer',
  layout: Atlas_Core.PopupLayout
)
{
  -- ... page content with SNIPPETCALL navMenu (Snippet: Module.NavigationMenu)
}
/

create page Module.Customer_Overview
(
  title: 'Customer Overview',
  layout: Atlas_Core.Atlas_Default
)
{
  -- ... page content with SNIPPETCALL navMenu (Snippet: Module.NavigationMenu)
}
/

-- Step 3: Replace snippet with full navigation (pages now exist)
create or replace snippet Module.NavigationMenu
{
  layoutgrid navGrid {
    row row1 {
      column col1 (desktopwidth: 12) {
        actionbutton btnCustomers (caption: 'Customers', action: show_page Module.Customer_Overview)
      }
    }
  }
}
/
```
</circular_dependency_pattern>

<snippet_commands_reference>
| Command | Description |
|---------|-------------|
| `show snippets [in module]` | List all snippets |
| `show snippet Module.Name` | Show snippet summary |
| `describe snippet Module.Name` | Show snippet MDL source |
| `create snippet Module.Name { ... }` | Create a new snippet |
| `create or replace snippet Module.Name { ... }` | Create or update snippet |
| `alter snippet Module.Name { ... }` | Modify snippet widgets in-place |
| `drop snippet Module.Name` | Delete a snippet |
</snippet_commands_reference>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
