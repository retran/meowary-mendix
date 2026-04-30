---
name: 'alter-page'
description: 'ALTER PAGE / ALTER SNIPPET — SET, INSERT, DROP, REPLACE widget operations on existing pages and snippets'
compatibility: opencode
---

<role>MDL page/snippet editor — targeted in-place modifications to existing pages and snippets without full rebuild.</role>

<summary>
> Syntax reference for ALTER PAGE and ALTER SNIPPET operations. Covers SET (property changes), INSERT (add widgets), DROP (remove widgets), REPLACE (swap widget subtrees), DataGrid column operations, ADD/DROP VARIABLES, and SET LAYOUT.
</summary>

<triggers>
Load when:
- Changing a button caption, label, style, or property on an existing page
- Adding or removing fields from an existing form
- Replacing a footer or section on an existing page
- Modifying a snippet widget tree
- Using DataGrid column operations (add, drop, set caption)
</triggers>

<overview>

ALTER PAGE and ALTER SNIPPET modify an existing page or snippet's widget tree **in-place** without requiring a full `create or replace`. Operations work directly on the raw BSON tree, preserving widget types and properties that MDL doesn't explicitly model.

| Scenario | Use |
|----------|-----|
| Change a button caption, label, or style | `alter page` with `set` |
| Add a field to an existing form | `alter page` with `insert` |
| Remove unused widgets | `alter page` with `drop` |
| Replace a footer or section | `alter page` with `replace` |
| Rebuild entire page from scratch | `create or replace page` |
| Create a new page | `create page` |

**Rule of thumb:** Use `alter page` for targeted edits to a few widgets. Use `create or replace page` when redefining the full page structure.

</overview>

<syntax>

```sql
alter page Module.PageName {
  operation1;
  operation2;
  ...
};

alter snippet Module.SnippetName {
  operation1;
  operation2;
  ...
};
```

Multiple operations can be combined in a single ALTER statement. They are applied sequentially.

</syntax>

<operations>

### SET - Modify Widget Properties

```sql
-- Single property
set caption = 'New Caption' on widgetName

-- Multiple properties
set (caption = 'Save & Close', buttonstyle = success) on btnSave

-- Page-level property (no ON clause)
set title = 'New Page Title'
```

**Supported SET properties:**

| Property | Widget Types | Value Type | Example |
|----------|-------------|------------|---------|
| `caption` | ACTIONBUTTON, LINKBUTTON | String | `set caption = 'Submit' on btnSave` |
| `content` | DYNAMICTEXT | String | `set content = 'New Heading' on txtTitle` |
| `label` | TEXTBOX, TEXTAREA, DATEPICKER, COMBOBOX, CHECKBOX, RADIOBUTTONS | String | `set label = 'full Name' on txtName` |
| `buttonstyle` | ACTIONBUTTON, LINKBUTTON | Primary, Default, Success, Danger, Warning, Info | `set buttonstyle = danger on btnDelete` |
| `class` | Any widget | CSS class string | `set class = 'card mx-2' on container1` |
| `style` | Any widget (see warning below) | Inline CSS string | `set style = 'padding: 16px;' on container1` |
| `editable` | Input widgets | String | `set editable = 'Never' on txtReadOnly` |
| `visible` | Any widget | String or Boolean | `set visible = false on txtHidden` |
| `Name` | Any widget | String | `set Name = 'newName' on oldName` |
| `title` | Page-level only | String | `set title = 'Edit Customer'` |
| `layout` | Page-level only | Qualified name | `set layout = Atlas_Core.Atlas_Default` |
| `'quotedProp'` | Pluggable widgets | String, Boolean, Number | `set 'showLabel' = false on cbStatus` |

**Pluggable widget properties** use quoted names to set values in the widget's `Object.Properties[]`. Boolean values are stored as `"yes"`/`"no"` in BSON.

> **Warning: Style on DYNAMICTEXT** — Setting `style` directly on a DYNAMICTEXT widget crashes MxBuild with a NullReferenceException. Wrap the DYNAMICTEXT in a CONTAINER and apply styling to the container instead:
> ```sql
> -- Wrong: crashes MxBuild
> SET Style = 'color: red;' ON txtHeading
>
> -- Correct: style the container
> REPLACE txtHeading WITH {
>   CONTAINER ctnHeading (Style: 'color: red;') {
>     DYNAMICTEXT txtHeading (Content: 'Heading', RenderMode: H2)
>   }
> }
> ```

### INSERT - Add Widgets

```sql
-- Insert after a widget
insert after txtName {
  textbox txtMiddleName (label: 'Middle Name', attribute: MiddleName)
}

-- Insert before a widget
insert before btnSave {
  actionbutton btnPreview (caption: 'Preview', action: microflow Module.ACT_Preview)
}
```

Inserted widgets use the same syntax as `create page`. Multiple widgets can be inserted in a single block.

### DROP - Remove Widgets

```sql
-- Drop a single widget
drop widget txtUnused

-- Drop multiple widgets
drop widget txtOldField, lblOldLabel, container2
```

Removes widgets and their entire subtree from the page.

### REPLACE - Replace Widget Subtree

```sql
-- Replace a single widget with new content
replace footer1 with {
  footer newFooter {
    actionbutton btnSave (caption: 'Save', action: save_changes, buttonstyle: primary)
    actionbutton btnCancel (caption: 'Cancel', action: cancel_changes)
  }
}
```

Replaces the target widget with one or more new widgets. The new widgets use the same syntax as `create page`.

### DataGrid Column Operations

DataGrid2 columns are addressable using dotted notation: `gridName.columnName`. The column name is derived from the attribute short name or caption (same as shown by `describe page`).

```sql
-- SET a column property
set caption = 'Product SKU' on dgProducts.Code

-- DROP a column
drop widget dgProducts.OldColumn

-- INSERT a column after an existing one
insert after dgProducts.Price {
  column Margin (attribute: Margin, caption: 'Margin')
}

-- REPLACE a column
replace dgProducts.Description with {
  column Notes (attribute: Notes, caption: 'Notes')
}
```

To discover column names, run `describe page Module.PageName` and look at the COLUMN names inside the DATAGRID.

### ADD Variables - Add a Page Variable

```sql
add variables $showStockColumn: boolean = 'true'
```

Adds a new page variable (`Forms$LocalVariable`) to the page/snippet. DataType can be `boolean`, `string`, `integer`, `decimal`, `datetime`, or an entity type. Default value is a Mendix expression in single quotes.

### DROP Variables - Remove a Page Variable

```sql
drop variables $showStockColumn
```

Removes a page variable by name.

### SET Layout - Change Page Layout

```sql
-- Auto-map placeholders by name (most common case)
set layout = Atlas_Core.Atlas_Default

-- Explicit mapping when placeholder names differ
set layout = Atlas_Core.Atlas_SideBar map (Main as content, Extra as Sidebar)
```

Changes the page's layout without rebuilding the widget tree. Only rewrites the `FormCall.Form` and `FormCall.Arguments[].Parameter` BSON fields — all widget content is preserved. Not supported for snippets.

When placeholders have the same names in both layouts (e.g., both have `Main`), auto-mapping works. Use `map` when placeholder names differ between the old and new layout.

</operations>

<examples>

### Change button text and style

```sql
alter page MyModule.Customer_Edit {
  set (caption = 'Save & Close', buttonstyle = success) on btnSave
};
```

### Add a field to a form

```sql
alter page MyModule.Customer_Edit {
  insert after txtEmail {
    textbox txtPhone (label: 'Phone', attribute: Phone)
  }
};
```

### Add a page variable for column visibility

```sql
alter page MyModule.ProductOverview {
  add variables $showStockColumn: boolean = 'if (3 < 4) then true else false'
};
```

### Remove unused fields and update title

```sql
alter page MyModule.Customer_Edit {
  set title = 'Edit Customer Details';
  drop widget txtLegacyField, lblOldNote;
  set label = 'Email Address' on txtEmail
};
```

### Replace a footer section

```sql
alter page MyModule.Customer_Edit {
  replace footer1 with {
    footer newFooter {
      actionbutton btnSave (caption: 'Save', action: save_changes, buttonstyle: success)
      actionbutton btnDelete (caption: 'Delete', action: delete, buttonstyle: danger)
      actionbutton btnCancel (caption: 'Cancel', action: cancel_changes)
    }
  }
};
```

### Modify a snippet

```sql
alter snippet MyModule.NavigationMenu {
  set caption = 'Dashboard' on btnHome;
  insert after btnHome {
    actionbutton btnReports (caption: 'Reports', action: show_page MyModule.Reports_Overview)
  }
};
```

### Set pluggable widget properties

```sql
alter page MyModule.Customer_Edit {
  set 'showLabel' = false on cbStatus;
  set 'labelWidth' = 4 on cbCategory
};
```

</examples>

<common_mistakes>

| Mistake | Fix |
|---------|-----|
| Missing `on widgetName` for widget SET | Add `on widgetName` (only page-level Title omits ON) |
| Using unquoted pluggable property names | Quote pluggable props: `set 'showLabel' = false on cb` |
| Wrong widget name | Use `describe page Module.Name` to see widget names |
| SET on non-existent widget | Widget names are case-sensitive; check with DESCRIBE |
| Missing semicolons between operations | Each operation inside `{ }` ends with `;` |

</common_mistakes>

<validation_checklist>

1. **Get widget names first**: Run `describe page Module.PageName` to see all widget names
2. **Check syntax**: `mxcli check script.mdl`
3. **Check references**: `mxcli check script.mdl -p app.mpr --references`
4. **Verify result**: Run `describe page Module.PageName` after ALTER to confirm changes
5. **Validate project**: `~/.mxcli/mxbuild/*/modeler/mx check app.mpr` (or `mxcli docker check -p app.mpr`)

</validation_checklist>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
