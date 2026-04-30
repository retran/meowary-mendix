---
name: 'create-custom-widget'
description: 'Create Custom Pluggable Widget — scaffold React+TypeScript widget with package.json, widget.xml, entry component, CSS, build, and install steps'
compatibility: opencode
---

<role>Mendix pluggable widget author — build a React+TypeScript widget from scratch that produces an .mpk file ready for Studio Pro.</role>

<summary>
> Step-by-step guide for building a Mendix pluggable widget from scratch using React + TypeScript. Covers scaffolding, widget.xml property definitions, entry component patterns, CSS, editor config, build, and install.
</summary>

<triggers>
Load when:
- User wants to create a new custom pluggable widget for Mendix
- Building a React+TypeScript widget with specific properties
- Understanding widget.xml property type definitions
- Troubleshooting CE0463 or build errors for custom widgets
</triggers>

<prerequisites>

- Node.js >= 16
- npm

</prerequisites>

<step_1_scaffold>

Create a directory and generate all source files. Use PascalCase for the widget name.

```bash
mkdir -p <WidgetName>/src/components <WidgetName>/src/ui
```

### package.json

```json
{
  "name": "<widget-name>",
  "widgetName": "<WidgetName>",
  "version": "1.0.0",
  "description": "<description>",
  "license": "Apache-2.0",
  "config": {
    "projectPath": "./tests/testProject",
    "mendixHost": "http://localhost:8080",
    "developmentPort": 3000
  },
  "packagePath": "com.example.widgets",
  "scripts": {
    "dev": "pluggable-widgets-tools start:web",
    "build": "pluggable-widgets-tools build:web",
    "lint": "pluggable-widgets-tools lint",
    "lint:fix": "pluggable-widgets-tools lint:fix"
  },
  "devDependencies": {
    "@mendix/pluggable-widgets-tools": "^11.6.0",
    "@types/big.js": "^6.0.2"
  },
  "dependencies": {
    "classnames": "^2.2.6"
  },
  "resolutions": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0"
  },
  "overrides": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0"
  }
}
```

**Naming rules:**
- `name`: kebab-case (npm package name)
- `widgetName`: PascalCase (matches .xml and .tsx filename)
- `packagePath`: reverse domain, dot-separated (e.g. `com.example.widgets`)

### tsconfig.json

```json
{
  "extends": "./node_modules/@mendix/pluggable-widgets-tools/configs/tsconfig.base.json"
}
```

### src/package.xml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<package xmlns="http://www.mendix.com/package/1.0/">
    <clientModule name="<WidgetName>" version="1.0.0" xmlns="http://www.mendix.com/clientModule/1.0/">
        <widgetFiles>
            <widgetFile path="<WidgetName>.xml"/>
        </widgetFiles>
        <files>
            <file path="com/example/widgets/<widgetname>"/>
        </files>
    </clientModule>
</package>
```

The `<file path>` must match `packagePath` + lowercase widget name, with dots replaced by `/`.

</step_1_scaffold>

<step_2_widget_xml>

### src/\<WidgetName\>.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<widget id="com.example.widgets.<widgetname>.<WidgetName>"
        pluginWidget="true"
        needsEntityContext="true"
        offlineCapable="true"
        supportedPlatform="Web"
        xmlns="http://www.mendix.com/widget/1.0/"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.mendix.com/widget/1.0/ ../node_modules/mendix/custom_widget.xsd">
    <name><Widget Name></name>
    <description><description></description>
    <icon/>
    <properties>
        <propertyGroup caption="General">
            <!-- Add properties here -->
        </propertyGroup>
    </properties>
</widget>
```

The `id` attribute must be `<packagePath>.<widgetname>.<WidgetName>` — the second-to-last segment is the **lowercase** widget name. Set `needsEntityContext="true"` when the widget needs entity data.

### Property Type Reference

| XML Type | Mendix Type | Use Case |
|----------|------------|----------|
| `string` | Static text | Labels, titles |
| `boolean` | Toggle | Show/hide, enable |
| `integer` | Number | Counts, sizes |
| `decimal` | Decimal | Measurements |
| `enumeration` | Enum choice | Mode selection |
| `expression` | Dynamic value | Computed text |
| `textTemplate` | Template text | Formatted text with params |
| `attribute` | Entity attribute | Data binding |
| `datasource` | List data source | Lists, grids |
| `widgets` | Child widgets | Content slots |
| `action` | On-click action | Buttons, links |
| `icon` | Icon | Decorative |
| `image` | Image | Avatar, logo |
| `object` | Compound | Complex config |

### Enumeration Example

```xml
<property key="alignment" type="enumeration" defaultValue="center">
    <caption>Alignment</caption>
    <description/>
    <enumerationValues>
        <enumerationValue key="left">left</enumerationValue>
        <enumerationValue key="center">Center</enumerationValue>
        <enumerationValue key="right">right</enumerationValue>
    </enumerationValues>
</property>
```

### Attribute Binding Example

```xml
<property key="value" type="attribute">
    <caption>value</caption>
    <description>The attribute to display</description>
    <attributeTypes>
        <attributeType name="string"/>
        <attributeType name="integer"/>
        <attributeType name="decimal"/>
    </attributeTypes>
</property>
```

### Object (Compound) Example — e.g. column definitions

```xml
<property key="columns" type="object" isList="true">
    <caption>columns</caption>
    <description/>
    <properties>
        <propertyGroup caption="column">
            <property key="header" type="textTemplate">
                <caption>header</caption>
                <translations><translation lang="en_US">column</translation></translations>
            </property>
            <property key="attribute" type="attribute" datasource="datasource">
                <caption>attribute</caption>
                <attributeTypes>
                    <attributeType name="string"/>
                    <attributeType name="integer"/>
                </attributeTypes>
            </property>
        </propertyGroup>
    </properties>
</property>
```

Note: `datasource="datasource"` links the attribute picker to the `datasource` property.

</step_2_widget_xml>

<step_3_entry_component>

### src/\<WidgetName\>.tsx

```tsx
import { ReactElement } from "react";
import { <WidgetName>ContainerProps } from "../typings/<WidgetName>Props";
import { MyComponent } from "./components/MyComponent";
import "./ui/<WidgetName>.css";

export function <WidgetName>(props: <WidgetName>ContainerProps): ReactElement {
    return <MyComponent {...relevantProps} />;
}
```

The `typings/<WidgetName>Props.d.ts` file is **auto-generated** by the build tool from the `.xml` definition. Do NOT create it manually.

### Key Mendix Prop Patterns

```tsx
// string property
props.title  // string

// boolean property
props.showHeader  // boolean

// expression property
props.label?.value  // string | undefined

// attribute property (read)
props.value?.displayValue  // string
props.value?.value  // actual typed value

// attribute property (write)
props.value?.setValue(newValue)

// action property
props.onClick?.canExecute  // boolean
props.onClick?.execute()   // trigger the action

// datasource property
props.dataSource?.items  // ObjectItem[] | undefined

// object list property (e.g. columns)
props.columns[0].attribute?.get(item)?.displayValue
```

</step_3_entry_component>

<step_4_react_component>

### src/components/MyComponent.tsx

Keep the component pure React — no Mendix API dependencies.

```tsx
import { ReactElement } from "react";
import classNames from "classnames";

export interface MyComponentProps {
    title: string;
    value?: string;
    className?: string;
}

export function MyComponent({ title, value, className }: MyComponentProps): ReactElement {
    return (
        <div className={classNames("widget-my-component", className)}>
            <h3>{title}</h3>
            {value && <p>{value}</p>}
        </div>
    );
}
```

</step_4_react_component>

<step_5_editor_config>

### src/\<WidgetName\>.editorConfig.ts

```ts
import { <WidgetName>PreviewProps } from "../typings/<WidgetName>Props";

export type properties = PropertyGroup[];
type PropertyGroup = {
    caption: string;
    propertyGroups?: PropertyGroup[];
    properties?: Property[];
};
type Property = {
    key: string;
    caption: string;
    description?: string;
};

export function getProperties(
    _values: <WidgetName>PreviewProps,
    defaultProperties: properties
): properties {
    return defaultProperties;
}
```

</step_5_editor_config>

<step_6_css>

### src/ui/\<WidgetName\>.css

```css
.widget-<widget-name> {
    /* widget styles */
}
```

Use a `.widget-<widget-name>` prefix to avoid CSS collisions.

</step_6_css>

<step_7_build_and_install>

```bash
cd <widget-dir>
npm install
npm run build
```

Output: `dist/<version>/com.example.widgets.<WidgetName>.mpk`

```bash
cp dist/*/*.mpk /path/to/mendix-project/widgets/
```

Then open/reload the project in Studio Pro.

</step_7_build_and_install>

<checklist_before_build>

- [ ] `id` in `.xml` matches `packagePath.WidgetName`
- [ ] `<name>` in package.xml matches `.xml` filename (without extension)
- [ ] `<file path>` in package.xml matches packagePath with `/` separators
- [ ] Entry `.tsx` exports a function with the exact widget name
- [ ] CSS file imported in entry `.tsx`
- [ ] `needsEntityContext` matches whether entity data is needed
- [ ] No manual `Props.d.ts` file (auto-generated by build tool)
- [ ] All `expression` properties have `<returnType>`
- [ ] All `attribute` properties list valid `<attributeType>` entries
- [ ] `object` properties with attributes set `datasource` reference

</checklist_before_build>

<troubleshooting>

| Error | Cause | Fix |
|-------|-------|-----|
| `Cannot find module '../typings/...'` | Haven't built yet | Run `npm run build` first, types are generated |
| `widget not showing in Studio Pro` | Wrong `id` in XML | Ensure `id="packagePath.WidgetName"` |
| `CE0463 widget definition changed` | Property mismatch | Ensure XML and component props match |
| `pluginWidget must be true` | Missing attribute | Add `pluginWidget="true"` to `<widget>` |

</troubleshooting>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
