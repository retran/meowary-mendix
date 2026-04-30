---
name: 'organize-project'
description: 'Organize Mendix project documents into folders and move them between folders and modules using MDL MOVE and FOLDER syntax'
compatibility: opencode
---

<role>
Project organization expert for Mendix. Covers folder conventions, creating documents in folders, moving documents between folders and modules, and managing folder lifecycle.
</role>

<summary>
Covers organizing Mendix project documents (pages, microflows, snippets, nanoflows) into folders and moving them between folders and modules.
</summary>

<triggers>
- Organizing documents into folder hierarchies within a module
- Moving documents between folders
- Moving documents between modules
- Restructuring a project for better maintainability
- Setting up folder conventions for a new module
</triggers>

<folder_conventions>
Organize by **functional grouping** — keep all artifacts for a feature together, not separated by document type.

```
CRM/
├── Customer/
│   ├── Customer_Overview        -- Overview page
│   ├── Customer_NewEdit         -- Edit page
│   ├── CustomerCard             -- Snippet
│   ├── ACT_Customer_Save        -- Save microflow
│   ├── ACT_Customer_Delete      -- Delete microflow
│   ├── ACT_Customer_New         -- New microflow
│   ├── VAL_Customer             -- Validation microflow
│   └── DS_Customer_Filter       -- Data source microflow
├── Order/
│   ├── Order_Overview
│   ├── Order_NewEdit
│   ├── ACT_Order_Save
│   └── VAL_Order
└── Shared/                      -- Cross-cutting concerns
    ├── SUB_SendNotification
    └── Navigation_Snippet
```

**Why functional grouping over type grouping:**
- All related artifacts are in one place — easier to navigate and review
- Adding or removing a feature is a single folder operation
- Naming prefixes (ACT_, VAL_, SUB_, DS_) already indicate document type
</folder_conventions>

<creating_documents_in_folders>
### Microflows

Use the `folder` keyword after the return type, before `begin`:

```mdl
create microflow MyModule.ACT_ProcessOrder ($Order: MyModule.Order)
returns boolean as $success
folder 'Order'
begin
  commit $Order;
  return true;
end;
```

### Pages

Use the `folder` property inside the page properties:

```sql
create page MyModule.Customer_Overview
(
  title: 'Customer Overview',
  layout: Atlas_Core.Atlas_Default,
  folder: 'Customer'
)
{
  -- widgets
}
```

### Snippets

```sql
create snippet MyModule.CustomerCard
(
  folder: 'Customer'
)
{
  -- widgets
}
```

### Nested Folders

Use `/` to create nested folder paths. Missing folders are created automatically:

```mdl
-- Creates 'Order', then 'Order/Batch' if they don't exist
create microflow MyModule.ACT_BatchProcess ($list: list of MyModule.Order)
folder 'Order/Batch'
begin
  loop $Order in $list begin
    commit $Order;
  end loop;
  return;
end;
```
</creating_documents_in_folders>

<moving_documents>
### Move to a Folder (Same Module)

```mdl
move page MyModule.CustomerEdit to folder 'Customer';
move microflow MyModule.ACT_ProcessOrder to folder 'Order';
move snippet MyModule.NavigationMenu to folder 'Shared';
move nanoflow MyModule.NAV_OpenCustomer to folder 'Customer';
move enumeration MyModule.OrderStatus to folder 'Shared';
```

### Move to Module Root (Out of Folder)

```mdl
move page MyModule.CustomerEdit to MyModule;
```

### Move Across Modules

```mdl
-- Move to another module's root
move page OldModule.CustomerPage to NewModule;

-- Move to a folder in another module
move page OldModule.CustomerPage to folder 'Pages' in NewModule;
```

### Cross-Module Move Warning

Cross-module moves change the qualified name — this **breaks by-name references**. Always check impact first:

```mdl
show impact of OldModule.CustomerPage;
-- Review the output, then move if safe:
move page OldModule.CustomerPage to NewModule;
```
</moving_documents>

<folder_rules>
- Folder names are **case-sensitive**
- Use `/` as separator for nested folders: `'Parent/Child/Grandchild'`
- Folders are **created automatically** if they don't exist
- Moving to a folder that doesn't exist creates it
- Empty folders are preserved in the project
</folder_rules>

<supported_document_types>
| Document Type | FOLDER on Create | MOVE Command |
|---------------|-----------------|--------------|
| Page          | `folder: 'path'` (property) | `move page ...` |
| Microflow     | `folder 'path'` (keyword) | `move microflow ...` |
| Nanoflow      | `folder 'path'` (keyword) | `move nanoflow ...` |
| Snippet       | `folder: 'path'` (property) | `move snippet ...` |
| Enumeration   | N/A | `move enumeration ...` |
| Entity        | N/A | `move entity ...` (module only, no folders) |

**Note:** Pages and snippets use property syntax (`folder: 'path'` inside parentheses). Microflows and nanoflows use keyword syntax (`folder 'path'` before `begin`).
</supported_document_types>

<reorganize_example>
```mdl
-- Group all Customer artifacts together
move page CRM.Customer_Overview to folder 'Customer';
move page CRM.Customer_NewEdit to folder 'Customer';
move microflow CRM.ACT_Customer_Save to folder 'Customer';
move microflow CRM.ACT_Customer_Delete to folder 'Customer';
move microflow CRM.ACT_Customer_New to folder 'Customer';
move microflow CRM.VAL_Customer to folder 'Customer';
move snippet CRM.CustomerCard to folder 'Customer';

-- Group all Order artifacts together
move page CRM.Order_Overview to folder 'Order';
move page CRM.Order_NewEdit to folder 'Order';
move microflow CRM.ACT_Order_Save to folder 'Order';
move microflow CRM.ACT_Order_Process to folder 'Order/Processing';

-- Move shared artifacts to a Shared folder or common module
show impact of CRM.Header_Snippet;
move snippet CRM.Header_Snippet to folder 'Shared' in Common;

-- Move entity to different module
show impact of CRM.Customer;
move entity CRM.Customer to CustomerModule;

-- Move enumeration to different module
move enumeration CRM.OrderStatus to SharedModule;
```
</reorganize_example>

<moving_folders>
Use `move folder` to reorganize folders:

```sql
-- Move a folder into another folder
move folder MyModule.Resources to folder 'Archive';

-- Move a nested folder (use double quotes for paths with /)
move folder MyModule."Orders/Archive" to MyModule;

-- Move a folder to a different module
move folder MyModule.SharedWidgets to CommonModule;

-- Move a folder into a folder in another module
move folder MyModule.Templates to folder 'Shared' in CommonModule;
```
</moving_folders>

<deleting_folders>
Use `drop folder` to remove empty folders. The folder must not contain any documents or sub-folders.

```sql
-- Drop an empty folder
drop folder 'OldPages' in MyModule;

-- Drop a nested folder (only the leaf is removed)
drop folder 'Orders/Archive' in MyModule;

-- Move contents out first, then drop
move microflow MyModule.ACT_Process to MyModule;
drop folder 'Processing' in MyModule;
```
</deleting_folders>

<validation_checklist>
- [ ] Folder paths use `/` separator (not `\`)
- [ ] FOLDER keyword placement is correct (before BEGIN for microflows, inside properties for pages)
- [ ] Cross-module moves: checked impact with `show impact of` first
- [ ] Folder naming is consistent across modules
- [ ] DROP FOLDER: verify folder is empty before dropping
</validation_checklist>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
