---
name: 'debug-bson'
description: 'BSON Serialization Debugging — mxcli dump-bson workflow, array version markers, object structures, field naming, nested WidgetObjects, property value types, TextTemplate fixes, and CE0463 troubleshooting'
compatibility: opencode
---

<role>BSON serialization debugger — compare SDK-generated BSON against Studio Pro reference to find and fix serialization mismatches.</role>

<summary>
> Guidance for debugging BSON serialization issues when implementing or fixing Mendix SDK writers. Covers the dump-bson workflow, common BSON patterns (array version markers, object structures, field naming), nested WidgetObject requirements, property value types, Texts$Translation vs Texts$TextItem, and CE0463 TextTemplate fixes.
</summary>

<triggers>
Load when:
- Objects created via MDL don't appear correctly in Studio Pro
- Properties are missing, empty, or showing default values
- Widget captions, parameters, or nested structures aren't stored correctly
- Need to compare SDK-generated BSON with Studio Pro-generated BSON
- Encountering "(Empty caption)" issues in Studio Pro
- Debugging CE0463 "widget definition has changed" errors
</triggers>

<debugging_workflow>

### Step 1: Identify the Problem

Symptoms that indicate BSON serialization issues:
- MDL `describe` command shows correct data, but Studio Pro shows empty/default
- Object is created but properties don't persist
- Nested structures (templates, parameters) are missing
- "(Empty caption)" or similar placeholder text in Studio Pro

### Step 2: Create a Reference Object in Studio Pro

1. Open the MPR project in Mendix Studio Pro
2. Manually create or fix the problematic object
3. Save the project (this writes the correct BSON)
4. Note the exact name/path of the fixed object

### Step 3: Dump Both BSON Structures

```bash
# Dump the SDK-generated object (the broken one)
./mxcli dump-bson -p app.mpr -o "PgTest.BrokenPage" > broken.json

# Dump the Studio Pro-generated object (the fixed one)
./mxcli dump-bson -p app.mpr -o "PgTest.FixedPage" > fixed.json

# Compare the two
diff broken.json fixed.json
```

Or use the `--compare` flag:
```bash
./mxcli dump-bson -p app.mpr --compare "PgTest.BrokenPage" "PgTest.FixedPage"
```

### Step 4: Identify Differences

| Area | Common Issues |
|------|---------------|
| Field names | `caption` vs `CaptionTemplate`, `Name` vs `InternalName` |
| Object vs value | String `""` vs nested `{$type: "..."}` object |
| Array markers | `[2, ...]` vs `[3, ...]` for different contexts |
| Missing fields | Required fields that SDK omits |
| Type mismatches | `int32` vs `int64`, `string` vs `binary` |

### Step 5: Fix the Serialization Code

The serialization code lives in `sdk/mpr/writer_*.go` files:
- `writer_widgets.go` - Page widget serialization
- `writer_microflows.go` - Microflow activity serialization
- `writer_entities.go` - Entity/attribute serialization

</debugging_workflow>

<common_bson_patterns>

### Array Version Markers

Mendix uses version markers at the start of arrays:

```go
// Empty array (version 3 marker)
bson.A{int32(3)}

// Non-empty array — context dependent!
// parameters arrays use version 2
bson.A{int32(2), item1, item2, ...}

// Texts$Text.Items arrays use version 3
bson.A{int32(3), item1, item2, ...}
```

**Critical**: The version marker differs by context. Always check a working Studio Pro example.

### Object Structures

```go
// WRONG: string value
{key: "FallbackValue", value: ""}

// CORRECT: Nested Texts$text object
{key: "Fallback", value: bson.D{
    {key: "$ID", value: idToBsonBinary(generateUUID())},
    {key: "$type", value: "Texts$text"},
    {key: "Items", value: bson.A{int32(3)}},
}}
```

### Field Naming

```go
// WRONG: Simplified name
{key: "caption", value: caption}

// CORRECT: Full field name
{key: "CaptionTemplate", value: caption}
```

</common_bson_patterns>

<debugging_tools>

```bash
# List all pages in project
./mxcli dump-bson -p app.mpr --type page --list

# List all microflows
./mxcli dump-bson -p app.mpr --type microflow --list

# Dump specific page as JSON
./mxcli dump-bson -p app.mpr --type page --object "PgTest.MyPage"

# Save dump to file for comparison
./mxcli dump-bson -p app.mpr --type page --object "PgTest.MyPage" > mypage.json

# Compare two objects (outputs both as JSON)
./mxcli dump-bson -p app.mpr --type page --compare "PgTest.Broken,PgTest.Fixed"

# Supported types: page, microflow, nanoflow, enumeration, snippet, layout
```

</debugging_tools>

<common_issues_and_solutions>

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| "(Empty caption)" in Studio Pro | Wrong field name or structure | Check `CaptionTemplate` vs `caption`, verify nested object |
| Parameters not showing | Wrong array version marker | Use `[2, ...]` for non-empty Parameters |
| Template text missing | Missing Texts$Text structure | Ensure proper TextItem serialization |
| Fallback empty | Using string instead of object | Use `Fallback` with Texts$Text object |
| Widget property ignored | Field name mismatch | Compare with Studio Pro BSON exactly |
| Columns not in Page Explorer | Incomplete nested WidgetObjects | Create ALL properties from template |
| TypeCacheUnknownTypeException | Wrong BSON $Type name | Use `Texts$Translation` not `Texts$TextItem` |
| CE0642 "Property is required" | Wrong value format or missing | Check ValueType.Type in template, use correct BSON field |
| NullReferenceException in GetExpectedExpressionType | Using Expression for non-Expression type | Use `PrimitiveValue` for Boolean/Enum/Integer types |
| CE0495 Duplicate name errors | Same widget in multiple properties | Set content/filter to empty widget arrays |

</common_issues_and_solutions>

<nested_widget_objects>

**Key Insight**: Pluggable widgets with nested objects (like DataGrid2 columns) require **ALL properties** to be created, not just the ones with explicit values.

When creating nested `WidgetObject` instances (e.g., DataGrid2 columns), creating only the properties with explicit values results in objects that don't appear in the Page Explorer.

**Example: DataGrid2 Columns**

DataGrid2 columns need all 21 properties:
```
showContentAs, attribute, content, dynamictext, exportValue, header, tooltip,
filter, visible, sortable, resizable, draggable, hidable, allowEventPropagation,
width, minWidth, minWidthLimit, size, alignment, columnClass, wrapText
```

Iterate through ALL `PropertyTypes` in the template's `ObjectType` and create a `WidgetProperty` for each one, using default values for properties without explicit values.

```bash
# Count properties in both versions
./mxcli dump-bson -p app.mpr --type page --object "PgTest.BrokenPage" | grep "WidgetProperty" | wc -l
./mxcli dump-bson -p app.mpr --type page --object "PgTest.FixedPage" | grep "WidgetProperty" | wc -l

# Check the template for required properties
grep '"PropertyKey"' sdk/widgets/templates/mendix-11.6/datagrid.json | head -30
```

</nested_widget_objects>

<property_value_types>

**Key Insight**: Pluggable widget properties require different value formats based on their `ValueType.Type` field in the widget template. Using the wrong format causes CE0642 or NullReferenceException.

| ValueType.Type | BSON Field | Example Value |
|----------------|------------|---------------|
| `expression` | `expression` | `"true"`, `"$currentObject/Name"` |
| `boolean` | `PrimitiveValue` | `"true"`, `"false"` |
| `enumeration` | `PrimitiveValue` | `"left"`, `"autofill"` |
| `integer` | `PrimitiveValue` | `"100"`, `"0"` |
| `decimal` | `PrimitiveValue` | `"10.5"` |
| `string` | `PrimitiveValue` | `"text value"` |
| `widgets` | `widgets` | `bson.A{...}` |
| `object` | `objects` | `bson.A{...}` |

**DataGrid2 Column Property Types Reference:**

| Property | ValueType.Type | BSON Field | Default Value |
|----------|---------------|------------|---------------|
| `visible` | Expression | `expression` | `"true"` |
| `sortable` | Boolean | `PrimitiveValue` | `"true"` |
| `resizable` | Boolean | `PrimitiveValue` | `"true"` |
| `draggable` | Boolean | `PrimitiveValue` | `"true"` |
| `wrapText` | Boolean | `PrimitiveValue` | `"false"` |
| `hidable` | Enumeration | `PrimitiveValue` | `"yes"` |
| `alignment` | Enumeration | `PrimitiveValue` | `"left"` |
| `width` | Enumeration | `PrimitiveValue` | `"autofill"` |
| `size` | Integer | `PrimitiveValue` | `"100"` |
| `header` | Object | `objects` | Empty translation |

```bash
# Check the embedded widget template
grep -A5 '"PropertyKey": "visible"' sdk/widgets/templates/mendix-11.6/datagrid.json

# Extract all property types
jq '.PropertyTypes[] | {key: .PropertyKey, type: .ValueType.Type}' sdk/widgets/templates/mendix-11.6/datagrid.json
```

</property_value_types>

<texts_translation>

**Critical**: The correct type for translatable text items is `Texts$Translation`, NOT `Texts$TextItem`.

```go
// WRONG: Texts$TextItem does not exist
{key: "$type", value: "Texts$TextItem"}

// CORRECT: use Texts$Translation with LanguageCode
{key: "$type", value: "Texts$Translation"},
{key: "LanguageCode", value: "en_US"},
{key: "text", value: "Your text here"},
```

```go
// Full example: Building a Header Translation
headerTranslation := bson.D{
    {key: "$ID", value: idToBsonBinary(generateUUID())},
    {key: "$type", value: "Texts$Translation"},
    {key: "LanguageCode", value: "en_US"},
    {key: "text", value: columnHeader},
}

headerText := bson.D{
    {key: "$ID", value: idToBsonBinary(generateUUID())},
    {key: "$type", value: "Texts$text"},
    {key: "Items", value: bson.A{int32(3), headerTranslation}},
}
```

</texts_translation>

<ce0463_texttemplate>

**CE0463 "widget definition has changed"** for filter widgets is often caused by TextTemplate properties being `null` instead of proper `Forms$ClientTemplate` structures.

Every TextTemplate property must have this structure (never null):

```json
"TextTemplate": {
  "$ID": "<32-char-guid>",
  "$type": "Forms$ClientTemplate",
  "Fallback": {
    "$ID": "<32-char-guid>",
    "$type": "Texts$text",
    "Items": []
  },
  "parameters": [],
  "template": {
    "$ID": "<32-char-guid>",
    "$type": "Texts$text",
    "Items": []
  }
}
```

**Affected Filter Widgets:**

| Widget | TextTemplate Properties |
|--------|------------------------|
| TextFilter | `placeholder`, `screenReaderButtonCaption`, `screenReaderInputCaption` |
| DateFilter | `placeholder`, `screenReaderButtonCaption`, `screenReaderCalendarCaption`, `screenReaderInputCaption` |
| DropdownFilter | `emptyOptionCaption`, `ariaLabel`, `emptySelectionCaption`, `filterInputPlaceholderCaption` |
| NumberFilter | `placeholder`, `screenReaderButtonCaption`, `screenReaderInputCaption` |

**Note**: Version markers (`[2]` or `[3]`) only exist in BSON wire format, not in JSON template files. Use truly empty arrays (`[]`) in JSON templates.

</ce0463_texttemplate>

<checklist>

- [ ] Create reference object manually in Studio Pro
- [ ] Dump both BSON structures (SDK vs Studio Pro)
- [ ] Identify all differences
- [ ] Fix field names to match Studio Pro exactly
- [ ] Fix object structures (nested vs value)
- [ ] Fix array version markers
- [ ] Test: Create object via MDL, verify in Studio Pro
- [ ] Update documentation (`docs/05-mdl-specification/10-bson-mapping.md`)

</checklist>

<quick_reference>

```bash
# 1. Find the broken object
./mxcli -p app.mpr -c "describe page PgTest.BrokenPage"

# 2. Create fixed version in Studio Pro, save project

# 3. Dump both objects to JSON files
./mxcli dump-bson -p app.mpr --type page --object "PgTest.BrokenPage" > broken.json
./mxcli dump-bson -p app.mpr --type page --object "PgTest.FixedPage" > fixed.json

# 4. Compare the JSON files
diff broken.json fixed.json

# 5. After fixing code, verify
go build ./... && mxcli exec test.mdl -p app.mpr
```

</quick_reference>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
