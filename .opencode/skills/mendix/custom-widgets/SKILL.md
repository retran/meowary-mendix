---
name: 'custom-widgets'
description: 'Custom & Pluggable Widgets in MDL — GALLERY, COMBOBOX, third-party widget .def.json and template JSON, 6 operations, mode conditions, and engine internals'
compatibility: opencode
---

<role>MDL pluggable widget author — wire built-in and third-party pluggable widgets in CREATE PAGE / ALTER PAGE statements.</role>

<summary>
> Covers built-in widget syntax (GALLERY, COMBOBOX), adding third-party widgets via .def.json and template JSON extraction, the 6 mapping operations, mode conditions, and engine internals including the PropertyTypeIDMap and MPK augmentation.
</summary>

<triggers>
Load when:
- Writing MDL for GALLERY, COMBOBOX, or third-party pluggable widgets
- Adding a new custom widget via .def.json
- Debugging CE0463 "widget definition changed" errors
- Understanding child slots (TEMPLATE/FILTER) for custom widgets
</triggers>

<built_in_pluggable_widgets>

### GALLERY

```sql
gallery galleryName (
  datasource: database from Module.Entity sort by Name asc,
  selection: single | multiple | none,
  DesktopColumns: 3,
  TabletColumns: 2,
  PhoneColumns: 1
) {
  template template1 {
    dynamictext title (content: '{1}', contentparams: [{1} = Name], rendermode: H4)
    dynamictext info  (content: '{1}', contentparams: [{1} = Email])
  }
  filter filter1 {
    textfilter   searchName  (attribute: Name)
    numberfilter searchScore (attribute: Score)
    dropdownfilter searchStatus (attribute: status)
    datefilter   searchDate  (attribute: CreatedAt)
  }
}
```

- `template` block → mapped to `content` property (child widgets rendered per row)
- `filter` block → mapped to `filtersPlaceholder` property (shown above list)
- `selection: none` omits the selection property (default if omitted)

### COMBOBOX

```sql
-- Enumeration mode (Attribute is an enum)
combobox cbStatus (label: 'Status', attribute: status)

-- Association mode (Attribute is an association)
combobox cmbCustomer (
  label: 'Customer',
  attribute: Order_Customer,
  datasource: database Module.Customer,
  CaptionAttribute: Name
)
```

- Engine detects association mode when `datasource` is present (`hasDataSource` condition)
- `CaptionAttribute` is the display attribute on the **target** entity
- In association mode, DataSource mapping must resolve before Association

</built_in_pluggable_widgets>

<adding_third_party_widget>

### Step 1 — Extract .def.json from .mpk

```bash
./mxcli widget extract --mpk widgets/MyWidget.mpk
# Output: .mxcli/widgets/mywidget.def.json

# Override MDL keyword
./mxcli widget extract --mpk widgets/MyWidget.mpk --mdl-name MYWIDGET
```

The `extract` command parses the .mpk and auto-infers operations from XML property types:

| XML Type | Operation | MDL Source Key |
|----------|-----------|----------------|
| attribute | attribute | `attribute` |
| association | association | `association` |
| datasource | datasource | `datasource` |
| selection | selection | `selection` |
| widgets | widgets (child slot) | container name (key uppercased) |
| boolean/string/enumeration/integer/decimal | primitive | hardcoded `value` from defaultValue |
| action/expression/textTemplate/object/icon/image/file | *skipped* | too complex for auto-mapping |

Skipped types require manual configuration in the .def.json.

### Step 2 — Extract BSON template from Studio Pro

```bash
# 1. In Studio Pro: drag the widget onto a test page, save the project
# 2. Extract the widget's BSON:
./mxcli bson dump -p App.mpr --type page --object "Module.TestPage" --format json
# 3. Extract the type and object fields from the customwidget, save as:
#    project/.mxcli/widgets/mywidget.json
```

Template JSON format:

```json
{
  "widgetId": "com.vendor.widget.MyWidget",
  "name": "My widget",
  "version": "1.0.0",
  "extractedFrom": "TestModule.TestPage",
  "type": { ... },
  "object": { ... }
}
```

**CRITICAL**: Template must include both `type` (PropertyTypes schema) and `object` (default WidgetObject). Extract from a real Studio Pro MPR — do NOT generate programmatically. Mismatched structure causes CE0463.

### Step 3 — Place files

```
project/.mxcli/widgets/mywidget.def.json   <- project scope (highest priority)
project/.mxcli/widgets/mywidget.json       <- template json (same directory)
~/.mxcli/widgets/mywidget.def.json         <- global scope
```

Set `"templateFile": "mywidget.json"` in the .def.json. Project definitions override global ones; global overrides embedded.

### Step 4 — Use in MDL

```sql
MYWIDGET myWidget1 (datasource: database Module.Entity, attribute: Name) {
  template content1 {
    dynamictext label1 (content: '{1}', contentparams: [{1}=Name])
  }
}
```

</adding_third_party_widget>

<def_json_reference>

```json
{
  "widgetId":        "com.vendor.widget.web.mywidget.MyWidget",
  "mdlName":         "MYWIDGET",
  "templateFile":    "mywidget.json",
  "defaultEditable": "Always",
  "propertyMappings": [
    {"propertyKey": "datasource",  "source": "datasource", "operation": "datasource"},
    {"propertyKey": "attribute",   "source": "attribute",  "operation": "attribute"},
    {"propertyKey": "someFlag",    "value":  "true",       "operation": "primitive"}
  ],
  "childSlots": [
    {"propertyKey": "content", "mdlContainer": "template", "operation": "widgets"}
  ],
  "modes": [
    {
      "name": "association",
      "condition": "hasDataSource",
      "propertyMappings": [
        {"propertyKey": "optionsSource", "value": "association", "operation": "primitive"},
        {"propertyKey": "assocDS",       "source": "datasource",  "operation": "datasource"},
        {"propertyKey": "assoc",         "source": "association", "operation": "association"}
      ]
    },
    {
      "name": "default",
      "propertyMappings": [
        {"propertyKey": "attr", "source": "attribute", "operation": "attribute"}
      ]
    }
  ]
}
```

### Mode Conditions

| Condition | Checks |
|-----------|--------|
| `hasDataSource` | AST widget has a `datasource` property |
| `hasAttribute` | AST widget has an `attribute` property |
| `hasProp:XYZ` | AST widget has a property named `XYZ` |

Modes are evaluated in definition order — first match wins.

### 6 Built-in Operations

| Operation | What it does |
|-----------|-------------|
| `attribute` | Sets `Value.AttributeRef` on a WidgetProperty |
| `association` | Sets `Value.AttributeRef` + `Value.EntityRef` |
| `primitive` | Sets `Value.PrimitiveValue` |
| `datasource` | Sets `Value.DataSource` (serialized BSON) |
| `selection` | Sets `Value.Selection` (mode string) |
| `widgets` | Replaces `Value.Widgets` array with child widget BSON |
| `texttemplate` | Sets text in `Value.TextTemplate` (Forms$ClientTemplate) |
| `action` | Sets `Value.Action` with serialized client action BSON |

**Mapping order**: `association` source must come **after** `datasource` source in the mappings array.

</def_json_reference>

<verify_and_debug>

```bash
# List registered widgets
./mxcli widget list -p App.mpr

# Check after creating a page
./mxcli check script.mdl -p App.mpr --references

# Full mx check (catches CE0463)
~/.mxcli/mxbuild/*/modeler/mx check App.mpr

# Debug CE0463 — compare NDSL dumps
./mxcli bson dump -p App.mpr --type page --object "Module.PageName" --format ndsl
```

</verify_and_debug>

<common_mistakes>

| Mistake | Fix |
|---------|-----|
| CE0463 after page creation | Template version mismatch — extract fresh template from Studio Pro MPR |
| Widget not recognized | Check `mxcli widget list`; .def.json must be in `.mxcli/widgets/` |
| TEMPLATE content missing | Widget needs `childSlots` entry with `"mdlContainer": "template"` |
| Association COMBOBOX shows enum behavior | Add `datasource` to trigger association mode (`hasDataSource` condition) |
| Association mapping fails | Ensure DataSource mapping appears **before** Association mapping |
| Placeholder ID leak error | Template JSON has unreferenced `$ID` values starting with `aa000000` |

</common_mistakes>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
