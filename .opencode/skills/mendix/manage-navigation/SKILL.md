---
name: 'manage-navigation'
description: 'Navigation Management — SHOW/DESCRIBE navigation, CREATE OR REPLACE NAVIGATION with home page, role-based routing, menu tree, login page, not-found page, and catalog queries'
compatibility: opencode
---

<role>MDL navigation manager — inspect and modify Mendix navigation profiles: home pages, menus, login pages, and role-based routing.</role>

<summary>
> Covers inspecting and modifying Mendix navigation profiles via MDL. Includes SHOW/DESCRIBE commands, CREATE OR REPLACE NAVIGATION with all clauses, role-based home pages, hierarchical menu trees, round-trip workflow, and catalog queries.
</summary>

<triggers>
Load when:
- User asks to view or change navigation home pages
- Viewing or modifying the navigation menu structure
- Setting login or not-found pages
- Configuring role-based home page routing
- Discovering which pages are navigation entry points
- Setting up navigation for a new project
</triggers>

<navigation_concepts>

- **Navigation Profiles** — Every Mendix project has profiles: Responsive, Phone, Tablet, and optionally Native. Each has its own home page, menu, and login page.
- **Home Page** — Default page shown after login. Can be a PAGE or MICROFLOW.
- **Role-Based Home Pages** — Override the default home page per user role.
- **Menu Items** — Hierarchical menu tree. Sub-menus nest with `menu 'caption' (...)`.
- **Login Page** — Custom login page (optional; Mendix provides a default).
- **Not-Found Page** — Custom 404 page (optional).

</navigation_concepts>

<show_commands>

```sql
-- Summary of all navigation profiles (home pages, menu counts)
show navigation;

-- Full MDL description of a profile (round-trippable output)
describe navigation Responsive;
describe navigation;              -- all profiles

-- Menu tree for a specific profile
show navigation menu Responsive;
show navigation menu;             -- all profiles

-- Home page assignments across all profiles and roles
show navigation homes;
```

</show_commands>

<create_or_replace_navigation>

This command fully replaces a navigation profile's configuration. All clauses are optional — omitted clauses clear that section.

### Basic: Set Home and Login Page

```sql
create or replace navigation Responsive
  home page MyModule.Home_Web
  login page Administration.Login;
```

### Role-Based Home Pages

```sql
create or replace navigation Responsive
  home page MyModule.Home_Web
  home page MyModule.AdminDashboard for Administration.Administrator
  home page MyModule.CustomerPortal for MyModule.Customer
  login page Administration.Login;
```

### Full Menu Tree

```sql
create or replace navigation Responsive
  home page MyModule.Home_Web
  login page Administration.Login
  menu (
    menu item 'Home' page MyModule.Home_Web;
    menu 'Orders' (
      menu item 'All Orders' page Orders.Order_Overview;
      menu item 'New Order' page Orders.Order_New;
    );
    menu 'Admin' (
      menu item 'Users' page Administration.Account_Overview;
      menu item 'Run Report' microflow Reports.ACT_GenerateReport;
    );
  );
```

### Clear the Menu

```sql
create or replace navigation Responsive
  home page MyModule.Home_Web
  menu ();
```

### Not-Found Page

```sql
create or replace navigation Responsive
  home page MyModule.Home_Web
  not found page MyModule.Custom404;
```

### Microflow as Home Page

```sql
create or replace navigation Responsive
  home microflow MyModule.ACT_ShowHome;
```

</create_or_replace_navigation>

<round_trip_workflow>

The DESCRIBE output is directly executable:

```sql
-- Step 1: Inspect current state
describe navigation Responsive;

-- Step 2: Copy the output, modify as needed, paste back
create or replace navigation Responsive
  home page MyModule.Home_Web
  login page Administration.Login
  menu (
    menu item 'Home' page MyModule.Home_Web;
    menu item 'New Feature' page MyModule.NewFeature;
  );

-- Step 3: Verify
describe navigation Responsive;
```

</round_trip_workflow>

<catalog_queries>

```sql
refresh catalog full;

-- Find all pages that are navigation entry points
select SourceName, TargetName, RefKind
from CATALOG.REFS
where RefKind in ('home_page', 'menu_item', 'login_page');

-- What references point to a specific page?
show references to MyModule.Home_Web;

-- Impact analysis: what breaks if I change this page?
show impact of MyModule.Home_Web;
```

</catalog_queries>

<common_patterns>

### New Project Setup

```sql
-- Create home page
create page MyModule.Home_Web
(
  title: 'Home',
  layout: Atlas_Core.Atlas_Default
)
{
  container ctnMain {
    dynamictext txtWelcome (content: 'Welcome!')
  }
}

-- Configure navigation
create or replace navigation Responsive
  home page MyModule.Home_Web
  menu (
    menu item 'Home' page MyModule.Home_Web;
  );
```

### Adding a New Page to Navigation

```sql
-- First inspect current menu
describe navigation Responsive;

-- Re-apply with the new item added
create or replace navigation Responsive
  home page MyModule.Home_Web
  login page Administration.Login
  menu (
    menu item 'Home' page MyModule.Home_Web;
    menu item 'Customers' page MyModule.Customer_Overview;  -- new
    menu 'Admin' (
      menu item 'Users' page Administration.Account_Overview;
    );
  );
```

</common_patterns>

<checklist>

- [ ] Profile name matches an existing profile (Responsive, Phone, Tablet, or a native profile)
- [ ] All PAGE/MICROFLOW targets are fully qualified (`Module.Name`)
- [ ] Role references in `for` clauses are fully qualified (`Module.Role`)
- [ ] Every `menu item` and `menu 'caption' (...)` ends with `;`
- [ ] Sub-menu items are wrapped in `menu 'caption' ( ... );`
- [ ] Use `describe navigation` to verify changes after applying

</checklist>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
