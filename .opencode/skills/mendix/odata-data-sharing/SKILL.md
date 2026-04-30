---
name: 'odata-data-sharing'
description: 'Share data between Mendix apps via OData services using view entities, published services, external entities, and OData clients'
compatibility: opencode
---

<role>
OData integration expert for Mendix inter-app data sharing. Covers view entity abstraction, OData service publishing, consumer setup, read-write APIs, and versioning.
</role>

<summary>
Covers how to use OData services to share data between Mendix applications, with emphasis on using view entities as an abstraction layer to decouple the API contract from the internal domain model.
</summary>

<triggers>
- Expose data from one Mendix app to another
- Set up inter-app communication via OData
- Create an API layer that abstracts internal entities
- Configure external entities or consumed/published OData services
- Decouple modules or apps for independent deployment
- Use the view entity pattern for OData services
- Work with local metadata files or offline OData development
</triggers>

<metadata_url_formats>
`CREATE ODATA CLIENT` supports three formats for the `MetadataUrl` parameter:

| Format | Example | Stored In Model |
|--------|---------|-----------------|
| **HTTP(S) URL** | `https://api.example.com/odata/v4/$metadata` | Unchanged |
| **Absolute file:// URI** | `file:///Users/team/contracts/service.xml` | Unchanged |
| **Relative path** | `./metadata/service.xml` or `metadata/service.xml` | **Normalized to absolute `file://`** |

**Path Normalization:**
- Relative paths are **automatically converted** to absolute `file://` URLs in the Mendix model
- With project loaded (`-p` flag or REPL): relative paths resolved against the `.mpr` file's directory
- Without project: relative paths resolved against the current working directory

**Use Cases for Local Metadata:**
- **Offline development** — no network access required
- **Testing and CI/CD** — reproducible builds with metadata snapshots
- **Version control** — commit metadata files alongside code
- **Pre-production** — test against upcoming API changes before deployment
</metadata_url_formats>

<service_url_must_be_constant>
**IMPORTANT:** The `ServiceUrl` parameter **must always be a constant reference** (prefixed with `@`).

```sql
-- Correct
CREATE CONSTANT ProductClient.ProductDataApiLocation
  TYPE String
  DEFAULT 'http://localhost:8080/odata/productdataapi/v1/';

CREATE ODATA CLIENT ProductClient.ProductDataApiClient (
  ODataVersion: OData4,
  MetadataUrl: 'https://api.example.com/$metadata',
  ServiceUrl: '@ProductClient.ProductDataApiLocation'  -- ✅ Constant reference
);

-- Incorrect
CREATE ODATA CLIENT ProductClient.ProductDataApiClient (
  ODataVersion: OData4,
  MetadataUrl: 'https://api.example.com/$metadata',
  ServiceUrl: 'https://api.example.com/odata'  -- ❌ Direct URL not allowed
);
```
</service_url_must_be_constant>

<architecture_overview>
OData data sharing follows a **producer/consumer** pattern with three layers:

```
┌─────────────────────────────────────────────┐
│  PRODUCER APP                               │
│                                             │
│  persistent entities  ──▶  view entities    │
│  (Shop.Customer,          (Api.CustomerVE)  │
│   Shop.Address)                             │
│                          ▼                  │
│                    odata service             │
│                   (Api.CustomerApi)          │
└──────────────────────┬──────────────────────┘
                       │ HTTP/OData4
┌──────────────────────▼──────────────────────┐
│  CONSUMER APP                               │
│                                             │
│                    odata client             │
│                  (Client.CustomerApiClient)  │
│                          ▼                  │
│                  external entities           │
│                 (Client.CustomersEE)         │
└─────────────────────────────────────────────┘
```

### Why View Entities?

Publishing persistent entities directly exposes your internal schema. **View entities** solve this:

1. **Stable API contract** — the view's shape stays the same even when underlying tables change
2. **Flattened data** — joins across multiple tables into a single flat resource
3. **Computed fields** — add calculated columns using OQL expressions
4. **Filtered datasets** — restrict what's visible
5. **Aggregations** — expose pre-aggregated metrics
</architecture_overview>

<read_only_api_steps>
### Step 1: Create the Producer Module and Role

```sql
create module ProductApi;

create module role ProductApi.ApiUser
  description 'Role for OData API access';
```

### Step 2: Create View Entities as the API Layer

```sql
/**
 * Flattened product with current active price.
 */
create view entity ProductApi.ProductWithPriceVE (
  ProductId: integer,
  Name: string,
  description: string,
  PriceInEuro: decimal
) as (
  select p.ID         as ProdId
  ,      p.ProductId  as ProductId
  ,      p.Name       as Name
  ,      p.Description as description
  ,      ( select pr.PriceInEuro
           from   Shop.Price as pr
           where  pr.StartDate <= '[%BeginOfTomorrow%]'
           and    pr/Shop.Price_Product = p.ID
           order  by pr.StartDate desc
           limit  1
         ) as PriceInEuro
  from   Shop.Product as p
  where  p.IsActive
);

grant ProductApi.ApiUser on ProductApi.ProductWithPriceVE
  (read *, write *);
```

### Step 3: Publish the OData Service

```sql
create odata service ProductApi.ProductDataApi (
  path: 'odata/productdataapi/v1/',
  version: '1.0.0',
  ODataVersion: OData4,
  namespace: 'DefaultNamespace',
  ServiceName: 'ProductDataApi',
  Summary: 'Product and customer data API',
  PublishAssociations: No
)
authentication basic
{
  publish entity ProductApi.ProductWithPriceVE as 'Product' (
    ReadMode: ReadFromDatabase,
    InsertMode: NotSupported,
    UpdateMode: NotSupported,
    DeleteMode: NotSupported
  )
  expose (
    ProductId as 'ProductId' (Filterable, Sortable, key),
    Name as 'Name' (Filterable, Sortable),
    description as 'Description' (Filterable, Sortable),
    PriceInEuro as 'PriceInEuro' (Filterable, Sortable)
  );
};

grant access on odata service ProductApi.ProductDataApi
  to ProductApi.ApiUser;
```

### Step 4: Set Up the Consumer App

```sql
create module ProductClient;
create module role ProductClient.User;

create constant ProductClient.ProductDataApiLocation
  type string
  default 'http://localhost:8080/odata/productdataapi/v1/';

create odata client ProductClient.ProductDataApiClient (
  ODataVersion: OData4,
  MetadataUrl: 'http://localhost:8080/odata/productdataapi/v1/$metadata',
  timeout: 300,
  ServiceUrl: '@ProductClient.ProductDataApiLocation',
  UseAuthentication: Yes,
  HttpUsername: 'MxAdmin',
  HttpPassword: '1'
);

-- External entities (mapped from published service)
create external entity ProductClient.ProductsEE
from odata client ProductClient.ProductDataApiClient
(
  EntitySet: 'Product',
  RemoteName: 'Product',
  Countable: Yes
)
(
  ProductId: long,
  Name: string,
  description: string,
  PriceInEuro: decimal
);

grant ProductClient.User on ProductClient.ProductsEE (read *);
```

**Bulk alternative:**
```sql
-- All entities from the service
create external entities from ProductClient.ProductDataApiClient;

-- Or specific ones only
create external entities from ProductClient.ProductDataApiClient
  entities (Product, CustomerAddress);

-- Idempotent re-import
create or modify external entities from ProductClient.ProductDataApiClient;
```
</read_only_api_steps>

<read_write_api>
For write operations, the OData service delegates to microflows that map between the view entity and the underlying persistent entities.

### CUD Microflows on the Producer

```sql
create microflow ProductApi.InsertProductWithPriceVE (
  $ProductWithPriceVE: ProductApi.ProductWithPriceVE,
  $HttpRequest: System.HttpRequest
)
begin
  $Product = create Shop.Product (
    Name = $ProductWithPriceVE/Name,
    description = $ProductWithPriceVE/description,
    IsActive = true
  );
  commit $Product;

  $Price = create Shop.Price (
    PriceInEuro = $ProductWithPriceVE/PriceInEuro,
    StartDate = '[%CurrentDateTime%]'
  );
  change $Price (Shop.Price_Product = $Product);
  commit $Price;
end;

grant execute on microflow ProductApi.InsertProductWithPriceVE
  to ProductApi.ApiUser;
```

### Wire Microflows to Published Entity

```sql
  publish entity ProductApi.ProductWithPriceVE as 'Product' (
    ReadMode: ReadFromDatabase,
    InsertMode: microflow ProductApi.InsertProductWithPriceVE,
    UpdateMode: microflow ProductApi.UpdateProductWithPriceVE,
    DeleteMode: microflow ProductApi.DeleteProductWithPriceVE
  )
  expose (...);
```

### Grant Write Access on External Entity

```sql
grant ProductClient.User on ProductClient.ProductsEE
  (create, delete, read *, write *);
```
</read_write_api>

<exploration_commands>
```sql
-- List all published and consumed services
show odata services;
show odata clients;

-- Inspect a specific service
describe odata service ShopViews.ShopViewsApi;
describe odata client ShopViewsClient.ShopViewsApiClient;

-- See external entities and view entities
show entities in ShopViewsClient;
show external entities;
show external actions;

-- Browse available assets from cached $metadata contract
show contract entities from ShopViewsClient.ShopViewsApiClient;
show contract actions from ShopViewsClient.ShopViewsApiClient;
describe contract entity ShopViewsClient.ShopViewsApiClient.Product;
describe contract entity ShopViewsClient.ShopViewsApiClient.Product format mdl;

-- Check security setup
show access on odata service ShopViews.ShopViewsApi;
show module roles in ShopViews;
```
</exploration_commands>

<module_organization>
| Module | Purpose | Contains |
|--------|---------|----------|
| `Shop` | Core domain | Persistent entities, business logic |
| `ShopApi` or `ShopViews` | API layer (producer) | View entities, OData service, CUD microflows |
| `ShopClient` or `ShopViewsClient` | API consumer | OData client, external entities, client constants |
</module_organization>

<checklist>
Before publishing:
- [ ] View entities expose only the fields consumers need
- [ ] View entity has at least one `key` field for OData identity
- [ ] Module role created and granted on view entities (READ, optionally WRITE)
- [ ] OData service has AUTHENTICATION set
- [ ] GRANT ACCESS ON ODATA SERVICE to the API module role
- [ ] CUD microflows (if writable) accept `($ViewEntity, $HttpRequest)` parameters

Before consuming:
- [ ] Location constant created for environment-specific URLs
- [ ] OData client `MetadataUrl` points to HTTP(S) URL, absolute `file://`, or relative path
- [ ] OData client uses `ServiceUrl: '@Module.Constant'` for runtime endpoint
- [ ] External entities match the published exposed names and types
- [ ] Module role created and granted on external entities
</checklist>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
