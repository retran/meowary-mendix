---
name: 'migrate-k2-nintex'
description: 'Assess and migrate K2/Nintex applications to Mendix: SmartObjects to entities, SmartForms to pages, workflows to microflows'
compatibility: opencode
---

<role>
Migration expert for K2/Nintex to Mendix. Covers SmartObject mapping, SmartForms conversion, workflow translation, and migration strategy.
</role>

<summary>
Comprehensive guidance for assessing and migrating K2 (now Nintex K2) applications to Mendix using MDL.
</summary>

<triggers>
- Analyzing K2/Nintex applications for migration to Mendix
- Converting SmartObjects to Mendix domain models
- Mapping SmartForms Views to Mendix pages
- Translating K2 Workflows to Mendix microflows or workflows
- Planning a migration strategy for legacy K2 systems
</triggers>

<k2_architecture>
K2 applications are fundamentally different from Mendix in how they're stored and structured.

### K2 Storage Model

**Key Difference**: K2 applications are **server-side/database-stored**, not file-based like Mendix. All K2 project elements and data are saved into the K2 database, not as files.

| Aspect | K2/Nintex | Mendix |
|--------|-----------|--------|
| Storage | Server database | `.mpr` file (SQLite) |
| Versioning | Server-managed | Git-based (MPR v2) |
| Export format | `.kspx` package | `.mpk` package |
| Project file | No single file | `.mpr` project file |

### K2 Artifact Types

#### SmartObjects

Middle layer between data providers (SQL, SAP, SharePoint) and data consumers (forms, workflows, reports).

| SmartObject Type | Description | Mendix Mapping |
|------------------|-------------|----------------|
| SmartBox | Stores data in K2's own database | Persistable Entity |
| SQL Connector | Connects to SQL Server tables | External Database Connector |
| SAP Connector | Connects to SAP systems | OData/REST Integration |
| SharePoint Connector | Connects to SharePoint lists | REST Client |
| Service Object | Exposes services/methods | Microflow/Java Action |

#### SmartForms

Browser-based forms composed of Views and Forms:

| SmartForms Element | Description | Mendix Mapping |
|--------------------|-------------|----------------|
| View | Reusable collection of controls + rules bound to SmartObjects | Snippet or Page Section |
| Form | Container for views, accessible via URL | Page |
| Control | UI element (text, date, dropdown, etc.) | Widget |
| Rule | Event-driven logic ("when button clicked, execute method") | Nanoflow or Microflow |

#### Workflows

| Workflow Element | Description | Mendix Mapping |
|------------------|-------------|----------------|
| Workflow | Full process definition | Workflow or Microflow chain |
| Activity | Individual step in workflow | Microflow activity or User Task |
| Task | Human task requiring user action | User Task |
| Destination Rule | Routing logic for tasks | Decision (microflow) |
| Datafield | Workflow data variable | Parameter or Variable |
</k2_architecture>

<export_options>
### 1. K2 Package (.kspx)

The K2 Package and Deployment tool packages K2 artifacts into a single file.

```
K2 Management Site → Solutions → Package → export
```

### 2. Legacy Project Files

| File Type | Extension | Contents |
|-----------|-----------|----------|
| Project file | `.k2proj` | Project structure and references |
| Workflow definition | `.kprx` | Workflow definition |
| SmartObject definition | `.sodx` | SmartObject schema |

### 3. K2 APIs

```csharp
using SourceCode.SmartObjects.Client;

SmartObjectClientServer server = new SmartObjectClientServer();
server.CreateConnection();
SmartObject so = server.GetSmartObject("CustomerSO");
```
</export_options>

<migration_strategy>
### Layer 1: Data Model (SmartObjects → Entities)

| SmartObject Type | Migration Approach |
|------------------|-------------------|
| SmartBox SmartObjects | Direct translation to Mendix persistable entities |
| SQL Connector SmartObjects | Options: (a) Import data to Mendix, (b) External Database Connector |
| Service SmartObjects | Microflows that call external services |

**Example MDL:**
```sql
-- SmartBox SmartObject "Customer" → Mendix Entity
create persistent entity CRM.Customer (
  CustomerCode: string(50),
  CustomerName: string(200),
  Email: string(200),
  Phone: string(50),
  IsActive: boolean default true,
  CreatedDate: datetime
);

-- SmartBox SmartObject "Order" → Mendix Entity
create persistent entity CRM.Order (
  OrderNumber: string(50),
  OrderDate: datetime,
  status: CRM.OrderStatus,
  TotalAmount: decimal
);

-- SmartObject relationship → Association
create association CRM.Order_Customer (
  CRM.Order [*] -> CRM.Customer [1]
);
```

### Layer 2: UI (SmartForms → Pages)

| SmartForms Control | Mendix Widget |
|--------------------|---------------|
| Text Box | TEXTBOX |
| Text Area | TEXTAREA |
| Drop-down List | COMBOBOX |
| Date Picker | DATEPICKER |
| Check Box | CHECKBOX |
| Radio Button | RADIOBUTTONS |
| Data Label | DYNAMICTEXT |
| Button | ACTIONBUTTON |
| List View | LISTVIEW or DATAGRID |
| Subview | SNIPPETCALL |
| Tab Control | Tab container pattern |

#### SmartForms Rules to Mendix Events

| SmartForms Rule | Mendix Implementation |
|-----------------|----------------------|
| When Control is Clicked | Button action microflow/nanoflow |
| When View is Initialized | Page data source microflow |
| When Control Value Changes | OnChange nanoflow |
| When Data Loads | Data source microflow |
| Execute SmartObject Method | Microflow calling entity operations |
| Transfer Data | Variable assignment in microflow |
| Show/Hide Control | Conditional visibility |
| Enable/Disable Control | Editable expression |

**Example MDL (SmartForm View → Mendix Page):**
```sql
create page CRM.Customer_Edit
(
  params: { $Customer: CRM.Customer },
  title: 'Edit Customer',
  layout: Atlas_Core.PopupLayout
)
{
  dataview dvCustomer (datasource: $Customer) {
    textbox txtCode (label: 'Customer Code', attribute: CustomerCode)
    textbox txtName (label: 'Customer Name', attribute: CustomerName)
    textbox txtEmail (label: 'Email', attribute: Email)
    textbox txtPhone (label: 'Phone', attribute: Phone)
    checkbox chkActive (label: 'Active', attribute: IsActive)

    footer footer1 {
      actionbutton btnSave (caption: 'Save', action: save_changes, buttonstyle: primary)
      actionbutton btnCancel (caption: 'Cancel', action: cancel_changes)
    }
  }
}
```

### Layer 3: Process (Workflows → Microflows/Workflows)

| K2 Workflow Element | Mendix Mapping |
|--------------------|----------------|
| Start | Microflow start / Workflow start |
| Task (human) | User Task activity |
| Reference (call SmartObject) | Microflow activities (Create, Change, Retrieve) |
| Decision | Decision (split/merge) |
| Send Email | Email activity |
| Generate Document | Generate document microflow |
| Web Service Call | REST/Web service call |
| Script | Java action or expressions |
| End | End event / Microflow return |
| Escalation | Scheduled event or timer |
| Destination Rule | Microflow logic for task assignment |

**Example MDL (K2 Task → Mendix Microflow):**
```sql
create microflow CRM.ACT_Order_SubmitForReview ($Order: CRM.Order)
begin
  change $Order (status = CRM.OrderStatus.PendingReview);
  commit $Order with events;
  show page CRM.Order_Review ($Order = $Order);
end;

create microflow CRM.ACT_Order_ProcessApproval ($Order: CRM.Order)
returns boolean as $Approved
begin
  declare $Approved boolean = false;

  if $Order/TotalAmount > 5000 then
    call microflow CRM.ACT_Order_SubmitForManagerReview ($Order = $Order);
  else
    change $Order (status = CRM.OrderStatus.Approved);
    commit $Order with events;
    set $Approved = true;
  end if;

  return $Approved;
end;
```
</migration_strategy>

<assessment_workflow>
### Step 1: Inventory SmartObjects

```markdown
| SmartObject Name | type | data source | entity count | Mendix mapping |
|------------------|------|-------------|--------------|----------------|
| CustomerSO | SmartBox | K2 DB | single | CRM.Customer entity |
| OrderSO | SmartBox | K2 DB | single | CRM.Order entity |
| EmployeeSO | sql | HR database | single | Integration or import |
| SAPOrderSO | SAP | SAP ERP | multiple | odata service |
```

### Step 2: Inventory SmartForms

```markdown
| Form Name | views Used | SmartObjects | Mendix mapping |
|-----------|------------|--------------|----------------|
| Customer Entry | CustomerView, AddressView | CustomerSO, AddressSO | Customer_Edit page |
| Order Dashboard | OrderListView, FilterView | OrderSO | Order_Overview page |
```

### Step 3: Inventory Workflows

```markdown
| workflow Name | Tasks | Activities | Complexity | Mendix mapping |
|---------------|-------|------------|------------|----------------|
| Order Approval | 3 | 12 | Medium | microflow chain |
| New Employee Onboarding | 8 | 25 | High | workflow module |
```

### Step 4: Map Rules and Logic

```markdown
| rule ID | Location | description | Mendix Implementation |
|---------|----------|-------------|----------------------|
| R-001 | CustomerView | Email format validation | validation microflow |
| R-002 | OrderWorkflow | Orders > $5000 need manager approval | decision in microflow |
```
</assessment_workflow>

<migration_execution_order>
### Phase 1: Domain Model
```sql
create enumeration CRM.OrderStatus as (
  Pending: 'Pending',
  Approved: 'Approved',
  Rejected: 'Rejected',
  Completed: 'Completed'
);

create persistent entity CRM.Customer (...);
create persistent entity CRM.Order (...);
create association CRM.Order_Customer (...);
```

### Phase 2: Business Logic (Microflows)
```sql
create microflow CRM.ACT_Customer_Save ($Customer: CRM.Customer)
begin
  if $Customer/Email = empty then
    validation feedback $Customer/Email message 'Email is required';
    return false;
  end if;
  commit $Customer with events;
  return true;
end;
```

### Phase 3: Pages
```sql
create page CRM.Customer_Overview (...);
create page CRM.Customer_Edit (...);
```

### Phase 4: Security
```sql
create module role CRM.Manager description 'Can approve orders and manage customers';
create module role CRM.User description 'Can create and edit own records';
grant CRM.Manager on CRM.Order (create, delete, read *, write *);
grant CRM.User on CRM.Order (create, read *, write *) where [owner = '[%CurrentUser%]'];
```
</migration_execution_order>

<common_challenges>
### Challenge 1: Server-Side Storage
**Problem**: K2 stores everything in a database, no single project file.
**Solution**: Use K2 Package (.kspx) export or K2 APIs.

### Challenge 2: SmartObject Connectors
| Connector Type | Mendix Options |
|----------------|----------------|
| SQL Direct | External Database Connector or data migration |
| SAP | SAP BAPI Connector, OData, or REST |
| SharePoint | REST integration via Microsoft Graph API |
| Web Service | REST/SOAP consumption |

### Challenge 3: Complex Rules
```
event-based UI logic → nanoflows
validation → validation microflows + validation feedback
data manipulation → microflows
Complex calculations → microflow expressions
```

### Challenge 4: Workflow Participants
```sql
create microflow CRM.SUB_GetManager ($Employee: HR.Employee)
returns HR.Employee as $Manager
begin
  declare $Manager HR.Employee;
  retrieve $Manager from HR.Employee
    where [HR.Employee_Reports = $Employee];
  return $Manager;
end;
```
</common_challenges>

<pre_migration_checklist>
Before starting migration:
- [ ] Obtain K2 Package (.kspx) export for all artifacts
- [ ] Get SmartObject documentation or extract via APIs
- [ ] Document all SmartForms views and their rules
- [ ] Map workflow steps and decision logic
- [ ] Identify external system integrations (SQL, SAP, SharePoint)
- [ ] Understand K2 security/role model
- [ ] Plan for data migration (SmartBox data → Mendix entities)

During migration:
- [ ] Create entities in dependency order
- [ ] Create enumerations before entities that use them
- [ ] Create microflows before pages that reference them
- [ ] Test validation rules thoroughly
- [ ] Verify workflow logic paths

After migration:
- [ ] Run `mxcli check script.mdl -p app.mpr --references`
- [ ] Open in Mendix Studio Pro to verify
- [ ] Test all workflow scenarios
- [ ] Validate data migration completeness
</pre_migration_checklist>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
