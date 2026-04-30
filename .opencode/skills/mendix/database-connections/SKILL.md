---
name: 'database-connections'
description: 'Create External Database Connections — CREATE DATABASE CONNECTION syntax, query definitions, column mapping, microflow execution with EXECUTE DATABASE QUERY, and bulk import'
compatibility: opencode
---

<role>MDL external database author — connect to JDBC databases, define queries, and execute them from microflows.</role>

<summary>
> Covers creating external database connections (Oracle, PostgreSQL, MySQL, SQL Server, Snowflake) using the External Database Connector. Includes connection syntax, query definitions, parameterized queries, column mapping, execute database query from microflows, and bulk import via SQL CONNECT.
</summary>

<triggers>
Load when:
- User asks to connect to an external database from a Mendix app (via JDBC)
- User needs to query data from Oracle, PostgreSQL, MySQL, SQL Server, or other JDBC databases
- Creating database connection configurations
- Defining SQL queries with parameter binding
- Mapping query results to Mendix entities
</triggers>

<overview>

> **Tip:** Use `generate connector` to auto-create all constants, entities, and queries from a database schema:
> ```
> SQL CONNECT postgres 'postgres://user:pass@host/db' AS source;
> SQL source GENERATE CONNECTOR INTO MyModule;
> -- Or generate for specific tables and execute immediately:
> SQL source GENERATE CONNECTOR INTO MyModule TABLES (employees, departments) EXEC;
> ```
> For manual exploration, use `sql source show tables;` and `sql source describe tablename;`.

**Requirements:** Mendix 9.22+, stable on Mendix 10.10+.

</overview>

<prerequisites>

```sql
-- Entity to hold query results (must be NON-PERSISTENT)
create non-persistent entity MyModule.EmployeeRecord (
  EmployeeId: integer,
  EmployeeName: string(100),
  Department: string(50),
  Salary: decimal
);

-- Connection credentials stored in constants
create constant MyModule.DbConnectionString type string
  default 'jdbc:oracle:thin:@//hostname:1521/SERVICENAME'
  comment 'JDBC connection string for external database';

create constant MyModule.DbUsername type string
  default 'app_user'
  comment 'Database username';

create constant MyModule.DbPassword type string
  default ''
  PRIVATE
  comment 'Database password - inject via environment variable in production';
```

</prerequisites>

<database_connection_syntax>

```sql
create database connection Module.ConnectionName
type '<database-type>'
connection string @Module.ConnectionStringConstant
username @Module.UsernameConstant
password @Module.PasswordConstant
begin
  -- Query definitions go here
end;
```

### Supported Database Types

| Database | TYPE Value |
|----------|------------|
| Oracle | `'Oracle'` |
| PostgreSQL | `'PostgreSQL'` |
| MySQL | `'MySQL'` |
| SQL Server | `'MSSQL'` or `'SQLServer'` |
| Snowflake | `'Snowflake'` |
| Amazon Redshift | `'Redshift'` |

</database_connection_syntax>

<query_definition_syntax>

### Simple Query (No Parameters)

```sql
query QueryName
  sql 'SELECT column1, column2 FROM table_name'
  returns Module.EntityName;
```

### Parameterized Query

```sql
query QueryName
  sql 'SELECT * FROM table_name WHERE column = {paramName}'
  parameter paramName: string
  returns Module.EntityName;
```

### Query with Column Mapping

```sql
query QueryName
  sql 'SELECT emp_id, emp_name, dept_no FROM employees'
  returns Module.EmployeeRecord
  map (
    emp_id as EmployeeId,
    emp_name as EmployeeName,
    dept_no as DepartmentNumber
  );
```

### Supported Parameter Types

`string`, `integer`, `decimal`, `boolean`, `datetime`

### Parameter Test Values

```sql
-- Test value (used in Studio Pro's Execute Query dialog)
parameter empName: string default 'Smith'

-- Test with NULL value
parameter optionalDate: datetime null
```

</query_definition_syntax>

<complete_example>

```sql
-- Step 1: Create constants
create constant OracleDemo.OracleConnectionString type string
  default 'jdbc:oracle:thin:@//10.211.55.2:1522/ORCLPDB1';

create constant OracleDemo.OracleUser type string default 'scott';

create constant OracleDemo.OraclePassword type string default 'tiger' PRIVATE;

-- Step 2: Create non-persistent entity
create non-persistent entity OracleDemo.EmpRecord (
  EMPNO: decimal,
  ENAME: string(10),
  JOB: string(9),
  SAL: decimal,
  DEPTNO: decimal
);

-- Step 3: Create database connection
create database connection OracleDemo.HRDatabase
type 'Oracle'
connection string @OracleDemo.OracleConnectionString
username @OracleDemo.OracleUser
password @OracleDemo.OraclePassword
begin
  query GetAllEmployees
    sql 'SELECT EMPNO, ENAME, JOB, SAL, DEPTNO FROM EMP ORDER BY EMPNO'
    returns OracleDemo.EmpRecord;

  query GetEmployeeByName
    sql 'SELECT EMPNO, ENAME, JOB, SAL, DEPTNO FROM EMP WHERE ENAME = {empName}'
    parameter empName: string
    returns OracleDemo.EmpRecord;
end;
```

</complete_example>

<viewing_connections>

```sql
-- List all database connections
show database connections;

-- List connections in a specific module
show database connections in MyModule;

-- View connection source code
describe database connection MyModule.MyDatabase;
```

</viewing_connections>

<execute_database_query>

Once a database connection and queries are defined, execute them from microflows using `execute database query`. The query is referenced by its **3-part qualified name**: `Module.Connection.Query`.

```sql
-- Execute a query and store results
$ResultList = execute database query Module.Connection.QueryName;

-- Fire-and-forget (no output variable)
execute database query Module.Connection.QueryName;

-- Dynamic SQL Override
$ResultList = execute database query Module.Connection.QueryName
  dynamic 'SELECT id, name FROM employees WHERE active = true LIMIT 10';

-- Parameterized query
$Drivers = execute database query Module.Connection.GetDriversByNationality
  (nation = $NationalityVar);

-- Runtime connection override (multiple databases, same schema)
$Results = execute database query Module.Connection.QueryName
  connection (DBSource = $url, DBUsername = $user, DBPassword = $Pass);
```

**CRITICAL**: Parameter names must exactly match those in the query definition.

**Note**: `execute database query` only supports `on error rollback`. `on error continue` is **not supported**.

</execute_database_query>

<importing_data>

To bulk-import data from an external database directly into the Mendix app's PostgreSQL database, use `import from` instead of the Database Connector:

```sql
sql connect postgres 'postgres://user:pass@host:5432/legacydb' as source;
import from source query 'SELECT name, email FROM employees'
  into HRModule.Employee
  map (name as Name, email as Email);
```

</importing_data>

<best_practices>

- Store JDBC URLs in constants for environment-specific overrides (`MX_Module_ConstantName` env vars)
- Use `PRIVATE` flag for password constants during development
- Never commit real passwords to version control
- Use NON-PERSISTENT entities for query results
- Use MAP clause when column names differ from attribute names
- Use parameterized queries to prevent SQL injection

</best_practices>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
