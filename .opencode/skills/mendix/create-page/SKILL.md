---
name: 'create-page'
description: 'CREATE PAGE — full MDL syntax guide for all widgets: DYNAMICTEXT, ACTIONBUTTON, LAYOUTGRID, DATAGRID, DATAVIEW, GALLERY, filters, CONTAINER, FOOTER, SNIPPETCALL, IMAGE, and pluggable widget escape hatch'
compatibility: opencode
---

<role>MDL page author — create and describe pages using the full widget vocabulary.</role>

<summary>
> Comprehensive syntax reference for CREATE PAGE in MDL. Covers page-level properties, all supported widgets with their properties and binding syntax, complete examples, conditional visibility/editability, known limitations, and the PLUGGABLEWIDGET escape hatch.
</summary>

<triggers>
Load when:
- Creating a new page or snippet in MDL
- Looking up the syntax for a specific widget type
- Using DATAGRID, GALLERY, DATAVIEW, COMBOBOX, or filter widgets
- Needing the PLUGGABLEWIDGET escape hatch for advanced properties
</triggers>

<syntax>

```sql
create [or replace] page Module.PageName
(
  [params: { $ParamName: Module.EntityType | PrimitiveType, ... },]
  [variables: { $varName: DataType = 'defaultExpression', ... },]
  title: 'Page Title',
  layout: Module.LayoutName,
  [url: 'page-url',]
  [folder: 'FolderPath']
)
{
  -- Widget definitions using explicit properties
}
```

**Page Variables**: Local variables at the page level for use in expressions (e.g., column visibility).
- DataType: `boolean`, `string`, `integer`, `decimal`, `datetime`
- Default value: Mendix expression in single quotes
- Referenced in expressions as `$varName`
- Use for DataGrid2 column `visible:` (which hides/shows entire column, NOT per-row)

### Key Syntax Elements

| Element | Syntax | Example |
|---------|--------|---------|
| Properties | `(key: value, ...)` | `(title: 'Edit', layout: Atlas_Core.Atlas_Default)` |
| Widget name | Required after type | `textbox txtName (...)` |
| Attribute binding | `attribute: AttrName` | `textbox txt (label: 'Name', attribute: Name)` |
| Variable binding | `datasource: $Var` | `dataview dv (datasource: $Product) { ... }` |
| Action binding | `action: type` | `actionbutton btn (caption: 'Save', action: save_changes)` |
| Database source | `datasource: database entity` | `datagrid dg (datasource: database Module.Entity)` |
| Selection binding | `datasource: selection widget` | `dataview dv (datasource: selection galleryList)` |
| CSS class | `class: 'classes'` | `container c (class: 'card mx-spacing-top-large')` |
| Inline style | `style: 'css'` | `container c (style: 'padding: 16px;')` |
| Design properties | `designproperties: [...]` | `container c (designproperties: ['Spacing top': 'Large', 'Full width': on])` |

### FOLDER Option

```sql
create page MyModule.CustomerEdit
(
  title: 'Edit Customer',
  layout: Atlas_Core.PopupLayout,
  folder: 'Customers'
)
{ -- widgets }

-- Nested folders (created automatically if they don't exist)
create page MyModule.OrderDetail
(
  title: 'Order Details',
  layout: Atlas_Core.Atlas_Default,
  folder: 'Orders/Details'
)
{ -- widgets }
```

### Styling: Class, Style, and DesignProperties

```sql
-- CSS Class
container c (class: 'card mx-spacing-top-large') { ... }

-- Inline Style
container c (style: 'background-color: #f8f9fa; padding: 16px;') { ... }

-- Design Properties
container c (designproperties: ['Spacing top': 'Large', 'Background color': 'Brand Primary']) { ... }
container c (designproperties: ['Full width': on]) { ... }

-- Combined
container ctnHero (
  class: 'card',
  style: 'border-left: 4px solid #264AE5;',
  designproperties: ['Spacing top': 'Large', 'Full width': on]
) {
  dynamictext txtTitle (content: 'Styled Container', rendermode: H3)
}
```

> **Warning:** Do NOT use `style` directly on DYNAMICTEXT widgets — it crashes MxBuild with a NullReferenceException. Wrap the DYNAMICTEXT in a styled CONTAINER instead.

</syntax>

<widgets>

### DYNAMICTEXT

```sql
-- Simple text
dynamictext heading (content: 'Heading Text', rendermode: H2)

-- Text bound to page parameter attribute
dynamictext productName (content: '$Product.Name', rendermode: H3)

-- Explicit template with page parameter binding
dynamictext greeting (content: 'Welcome, {1}!', contentparams: [{1} = $Customer.Name])

-- Template with attribute from current DataView context
dynamictext email (content: 'Email: {1}', contentparams: [{1} = Email])
```

**ContentParams Reference Types:**
| Syntax | Context | Example |
|--------|---------|---------|
| `$ParamName.Attr` | Page parameter attribute | `$Product.Name` |
| `AttrName` | Current DataView/Gallery entity | `Name`, `Email` |
| `'literal'` | String literal expression | `'Hello'` |

### ACTIONBUTTON

```sql
actionbutton widgetName (caption: 'Caption', action: ACTION_TYPE [, buttonstyle: style])
```

**Action Bindings:**
- `action: save_changes` / `action: save_changes close_page`
- `action: cancel_changes` / `action: close_page` / `action: delete`
- `action: microflow Module.MicroflowName` / `action: microflow Module.MicroflowName(Param: $value)`
- `action: nanoflow Module.NanoflowName` / `action: nanoflow Module.NanoflowName(Param: $value)`
- `action: show_page Module.PageName` / `action: show_page Module.PageName(Param: $value)`
- `action: create_object Module.Entity then show_page Module.PageName`

**Button Styles:** `default`, `primary`, `success`, `info`, `warning`, `danger`, `Inverse`

```sql
actionbutton btnSave (caption: 'Save', action: save_changes, buttonstyle: primary)
actionbutton btnEdit (caption: 'Edit', action: show_page Module.EditPage(Product: $Product))
actionbutton btnDelete (caption: 'Delete', action: microflow Module.ACT_Delete(Target: $currentObject), buttonstyle: danger)
actionbutton btnNew (caption: 'New', action: create_object Module.Product then show_page Module.Product_Edit, buttonstyle: primary)
```

Use `$currentObject` inside DATAGRID, LISTVIEW, or GALLERY columns to reference the current row's object.

### LAYOUTGRID

```sql
layoutgrid gridName {
  row rowName {
    column colName (desktopwidth: 8) {
      -- Nested widgets
    }
    column col2 (desktopwidth: 4) {
      -- Nested widgets
    }
  }
}
```

| Property | Values | Default |
|----------|--------|---------|
| `desktopwidth` | 1-12 or `autofill` | `autofill` |
| `tabletwidth` | 1-12 or `autofill` | auto |
| `phonewidth` | 1-12 or `autofill` | auto |

### DATAGRID

```sql
datagrid gridName (
  datasource: database from Module.Entity where [condition] sort by attributename asc|desc,
  selection: single|multiple|none
) {
  column colName (attribute: attributename, caption: 'Label')
}
```

**Column Properties:**

| Property | Values | Default |
|----------|--------|---------|
| `attribute` | attribute name | (required) |
| `caption` | string | attribute name |
| `Alignment` | `left`, `center`, `right` | `left` |
| `WrapText` | `true`, `false` | `false` |
| `Sortable` | `true`, `false` | `true` |
| `Resizable` | `true`, `false` | `true` |
| `Hidable` | `yes`, `hidden`, `no` | `yes` |
| `ColumnWidth` | `autofill`, `autoFit`, `manual` | `autofill` |
| `Size` | integer (px) | `1` |
| `visible` | expression string | `true` |
| `DynamicCellClass` | expression string | (empty) |
| `tooltip` | text string | (empty) |

**Paging Properties:**

| Property | Values | Default |
|----------|--------|---------|
| `PageSize` | positive integer | 20 |
| `Pagination` | `buttons`, `virtualScrolling`, `loadMore` | `buttons` |
| `PagingPosition` | `bottom`, `top`, `both` | `bottom` |

**Datasource Types:**
- `datasource: database from Module.Entity`
- `datasource: $Variable` / `datasource: microflow Module.GetData()`
- `datasource: selection widgetName`
- `datasource: $currentObject/Module.Assoc`

### DATAVIEW

```sql
dataview dvName (datasource: $VariableName) {
  textbox txtName (label: 'Name', attribute: Name)
  textarea txtDescription (label: 'Description', attribute: description)

  footer footer1 {
    actionbutton btnSave (caption: 'Save', action: save_changes, buttonstyle: primary)
    actionbutton btnCancel (caption: 'Cancel', action: cancel_changes)
  }
}
```

### Input Widgets

Must be inside a DATAVIEW context. Use `attribute:` to bind to attributes:

```sql
textbox txtName (label: 'Label', attribute: attributename)
textarea txtDescription (label: 'Description', attribute: description)
checkbox cbActive (label: 'Active', attribute: IsActive)
radiobuttons rbStatus (label: 'Status', attribute: status)
datepicker dpCreated (label: 'Created Date', attribute: CreatedDate)

-- Enumeration mode
combobox cbCountry (label: 'Country', attribute: Country)

-- Association mode
combobox cmbCustomer (label: 'Customer', attribute: Order_Customer, datasource: database MyModule.Customer, CaptionAttribute: Name)
```

### GALLERY

```sql
gallery galleryName (
  datasource: database from Module.Entity sort by Name asc,
  selection: single|multiple|none,
  DesktopColumns: 3,
  TabletColumns: 2,
  PhoneColumns: 1
) {
  filter filter1 {
    textfilter searchName (attribute: Name)
    numberfilter searchScore (attribute: Score)
    dropdownfilter searchStatus (attribute: status)
    datefilter searchDate (attribute: CreatedAt)
  }
  template template1 {
    dynamictext name (content: '{1}', contentparams: [{1} = Name], rendermode: H4)
    dynamictext email (content: '{1}', contentparams: [{1} = Email])
  }
}
```

**Filter Types:**
- `textfilter` — Text search (`filtertype`: `contains`, `startsWith`, `endsWith`, `equal`)
- `numberfilter` — Numeric range
- `datefilter` — Date range
- `dropdownfilter` — Dropdown selection

### CONTAINER / CUSTOMCONTAINER

```sql
container card1 (class: 'card', style: 'padding: 16px;') {
  dynamictext title (content: 'Card Title', rendermode: H4)
}

-- Note: Empty CONTAINER crashes at runtime — always include at least one child widget
```

### FOOTER and HEADER

```sql
footer footerName {
  actionbutton btnSave (caption: 'Save', action: save_changes, buttonstyle: primary)
  actionbutton btnCancel (caption: 'Cancel', action: cancel_changes)
}

header headerName {
  dynamictext title (content: 'Form Title', rendermode: H3)
}
```

### SNIPPETCALL

```sql
snippetcall snippetName (snippet: Module.SnippetName)
snippetcall actions (snippet: Module.EntityActions, params: {entity: $currentObject})
```

### IMAGE

```sql
image imgLogo (width: 200, height: 100)

-- Static image from file (use PLUGGABLEWIDGET for advanced control)
pluggablewidget 'com.mendix.widget.web.image.Image' imgLogo (
  datasource: imageUrl,
  imageUrl: 'img/logo.svg',
  widthUnit: pixels, width: 48,
  heightUnit: pixels, height: 48
)
```

</widgets>

<complete_examples>

### Customer Edit Page

```sql
create or replace page CRM.CustomerEdit
(
  params: { $Customer: CRM.Customer },
  title: 'Edit Customer',
  layout: Atlas_Core.PopupLayout
)
{
  dataview dvCustomer (datasource: $Customer) {
    textbox txtName (label: 'Name', attribute: Name)
    textbox txtEmail (label: 'Email', attribute: Email)
    textbox txtPhone (label: 'Phone', attribute: Phone)
    checkbox cbActive (label: 'Active', attribute: IsActive)

    footer footer1 {
      actionbutton btnSave (caption: 'Save', action: save_changes, buttonstyle: primary)
      actionbutton btnCancel (caption: 'Cancel', action: cancel_changes)
    }
  }
}
```

### Master-Detail Page

```sql
create page CRM.Customer_MasterDetail
(
  title: 'Customer Management',
  layout: Atlas_Core.Atlas_Default
)
{
  layoutgrid mainGrid {
    row row1 {
      column colMaster (desktopwidth: 4) {
        gallery customerList (datasource: database from CRM.Customer sort by Name asc, selection: single) {
          template template1 {
            dynamictext name (content: '{1}', contentparams: [{1} = Name], rendermode: H4)
            dynamictext email (content: '{1}', contentparams: [{1} = Email])
          }
        }
      }
      column colDetail (desktopwidth: 8) {
        dataview customerDetail (datasource: selection customerList) {
          textbox txtName (label: 'Name', attribute: Name)
          textbox txtEmail (label: 'Email', attribute: Email)
          footer footer1 {
            actionbutton btnSave (caption: 'Save', action: save_changes, buttonstyle: primary)
            actionbutton btnCancel (caption: 'Cancel', action: cancel_changes)
          }
        }
      }
    }
  }
}
```

</complete_examples>

<conditional_visibility>

```sql
-- Conditionally visible widget
textbox txtName (label: 'Name', attribute: Name, visible: [IsActive])

-- Conditionally editable input
textbox txtStatus (label: 'Status', attribute: status, editable: [status != 'Closed'])

-- Static values
textbox txtReadOnly (label: 'Read Only', attribute: Name, editable: Never)
textbox txtHidden (label: 'Hidden', attribute: Name, visible: false)
```

</conditional_visibility>

<known_limitations>

| Feature | Workaround |
|---------|------------|
| Nested dataviews filtering by parent | Use microflow datasource or configure in Studio Pro |
| Complex conditional visibility | Configure visibility rules in Studio Pro |
| Widget-level security | Configure access rules in Studio Pro |

> **Empty CONTAINER crashes at runtime.** Always include at least one child widget.

> **`content: ''` (empty string) fails MxBuild.** Use a single space instead: `content: ' '`

</known_limitations>

<pluggablewidget_escape_hatch>

When a shorthand widget doesn't expose a property, use full PLUGGABLEWIDGET syntax:

```sql
pluggablewidget 'com.mendix.widget.web.image.Image' imgLogo (
  datasource: imageUrl, imageUrl: 'img/logo.svg',
  widthUnit: pixels, width: 48, heightUnit: pixels, height: 48
)
```

Run `./mxcli widget docs -p <project>.mpr` to generate complete property documentation for all pluggable widgets in the project.

</pluggablewidget_escape_hatch>

<related_commands>

```sql
alter page Module.PageName { ... }     -- Modify page widgets in-place
alter snippet Module.SnippetName { ... } -- Modify snippet widgets in-place
describe page Module.PageName          -- View page source in MDL format
show pages [in module]                 -- List all pages
drop page Module.PageName              -- Delete a page
```

</related_commands>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
