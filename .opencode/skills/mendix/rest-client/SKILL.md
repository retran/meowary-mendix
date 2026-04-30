---
name: 'rest-client'
description: 'Integrate Mendix with external REST APIs: OpenAPI import, manual REST client definition, inline REST CALL, Data Transformers (JSLT), and import/export mappings'
compatibility: opencode
---

<role>
REST integration expert for Mendix. Covers three approaches: OpenAPI import, manual REST client, and inline REST CALL. Also covers Data Transformers (JSLT) and import/export mappings.
</role>

<summary>
Mendix offers three ways to call REST APIs from microflows: OpenAPI import (fastest), manual REST client definition, and inline REST CALL (most flexible). All approaches can be combined with Data Transformers and Import/Export Mappings.
</summary>

<triggers>
- Integrate with an external REST API
- Create a REST client document for an API
- Import an OpenAPI spec to generate a REST client
- Write a microflow that calls an external HTTP endpoint
- Transform JSON responses before importing
- Use JSLT data transformers
</triggers>

<three_approaches>
| Approach | When to Use | Artifacts |
|----------|-------------|-----------|
| **OpenAPI import** | API has an OpenAPI 3.0 spec — auto-generate from the spec | REST client document generated in one command |
| **REST Client (manual)** | No spec available, or need fine-grained control | REST client document + microflow |
| **REST CALL (inline)** | One-off calls, quick prototyping, dynamic URLs, low-level HTTP control | Microflow only |
</three_approaches>

<approach_0_openapi>
If the API has an OpenAPI 3.0 spec (JSON or YAML), generate the REST client in one command:

```sql
-- From a local file (relative to the .mpr file)
create or modify rest client CapitalModule.CapitalAPI (
  OpenAPI: 'specs/capital.json'
);

-- From a URL
create or modify rest client PetStoreModule.PetStoreAPI (
  OpenAPI: 'https://petstore3.swagger.io/api/v3/openapi.json'
);

-- Override the base URL
create or modify rest client PetStoreModule.PetStoreStaging (
  OpenAPI: 'https://petstore3.swagger.io/api/v3/openapi.json',
  BaseUrl: 'https://staging.petstore.example.com/api/v3'
);
```

This generates all operations, resource groups, and basic auth if declared in the spec.

**Preview without writing:**
```sql
describe contract operation from openapi 'specs/capital.json';
```
</approach_0_openapi>

<approach_1_rest_client_manual>
### Step 1 — Create the REST Client

```sql
create rest client Module.OpenMeteoAPI (
  BaseUrl: 'https://api.open-meteo.com/v1',
  authentication: none
)
{
  operation GetForecast {
    method: get,
    path: '/forecast',
    query: ($latitude: decimal, $longitude: decimal, $current: string),
    headers: ('Accept' = 'application/json'),
    timeout: 30,
    response: json as $WeatherJson
  }

  operation PostData {
    method: post,
    path: '/submit',
    headers: ('Content-Type' = 'application/json'),
    body: json from $JsonPayload,
    response: none
  }
};
```

### Authentication

```sql
authentication: none
authentication: basic (username: 'user', password: 'secret')
```

### Body Types

```sql
body: json from $JsonPayload
body: template '{ "name": "{name}", "value": {value} }'
body: mapping Module.RequestEntity {
  name = Name,
  email = Email,
}
```

### Response Types

```sql
response: json as $Result
response: string as $text
response: file as $Document
response: status as $Code
response: none
response: mapping Module.ResponseEntity {
  "Id" = "id",
  "status" = "status",
  create Module.Items_Response/Module.Item = items {
    "Sku" = "sku",
    "Quantity" = "quantity",
  }
}
```

### Step 2 — Call from a Microflow

```sql
create microflow Module.ACT_GetWeather ()
returns Module.WeatherInfo as $Weather
begin
  send rest request Module.OpenMeteoAPI.GetForecast;

  -- Extract response body from system variable
  declare $RawJson string = $latestHttpResponse/content;

  -- (Optional) Transform with JSLT
  $SimplifiedJson = transform $RawJson with Module.SimplifyWeather;

  -- Import into entity
  $Weather = import from mapping Module.IMM_Weather($SimplifiedJson);

  return $Weather;
end;
/
```

**CRITICAL**: After `send rest request`, the response is in `$latestHttpResponse`:
- `$latestHttpResponse/content` — response body (String)
- `$latestHttpResponse/StatusCode` — HTTP status (Integer)

### Show / Describe / Drop

```sql
show rest clients [in module];
describe rest client Module.ClientName;
drop rest client Module.ClientName;
create or modify rest client Module.ClientName ...  -- idempotent
```
</approach_1_rest_client_manual>

<approach_2_rest_call_inline>
```sql
-- Simple GET returning a string
$response = rest call get 'https://api.example.com/data'
  header Accept = 'application/json'
  timeout 30
  returns string;

-- GET with URL template parameters
$response = rest call get 'https://api.example.com/users/{1}' with (
  {1} = toString($UserId)
)
  header Accept = 'application/json'
  returns string;

-- POST with body
$response = rest call post 'https://api.example.com/items'
  header 'Content-Type' = 'application/json'
  body '{"name": "test"}'
  returns string;

-- With basic auth
$response = rest call get 'https://api.example.com/secure'
  auth basic 'username' password 'password'
  returns string;

-- With import mapping (JSON → entity)
$item = rest call get 'https://api.example.com/item/1'
  header Accept = 'application/json'
  returns mapping Module.IMM_Item as Module.Item;

-- Fire and forget
rest call delete 'https://api.example.com/item/1'
  returns nothing;

-- Error handling
$response = rest call get 'https://api.example.com/data'
  returns string
  on error continue;
```
</approach_2_rest_call_inline>

<data_transformers>
Transform complex JSON responses into simpler structures before import mapping (Mendix 11.9+).

```sql
-- Define the transformer
create data transformer Module.SimplifyWeather
source json '{"latitude": 52.52, "current": {"temperature_2m": 12.8, "wind_speed_10m": 18.3}}'
{
  jslt $$
{
  "temp": .current.temperature_2m,
  "wind": .current.wind_speed_10m,
  "lat": .latitude
}
  $$;
};

-- Use in a microflow
$SimplifiedJson = transform $RawJson with Module.SimplifyWeather;
```

Single-line JSLT: `jslt '{ "temp": .current.temperature_2m }';`
Multi-line JSLT: `jslt $$ { ... } $$;`

```sql
list data transformers [in module];
describe data transformer Module.Name;
drop data transformer Module.Name;
```
</data_transformers>

<complete_pipeline_example>
```sql
-- 1. Entity
create non-persistent entity Module.CurrentWeather (
  Temperature: decimal,
  WindSpeed: decimal,
  Latitude: decimal,
  ObservationTime: datetime
);
/

-- 2. Data Transformer (simplify API response)
create data transformer Module.SimplifyWeather
source json '{"latitude":52.52,"current":{"time":"2024-01-15T14:00","temperature_2m":12.8,"wind_speed_10m":18.3}}'
{
  jslt $$
{
  "temperature": .current.temperature_2m,
  "windSpeed": .current.wind_speed_10m,
  "latitude": .latitude,
  "observationTime": .current.time
}
  $$;
};

-- 3. JSON Structure + Import Mapping
create json structure Module.JSON_Weather
snippet '{"temperature":12.8,"windSpeed":18.3,"latitude":52.52,"observationTime":"2024-01-15T14:00"}';

create import mapping Module.IMM_Weather
  with json structure Module.JSON_Weather
{
  create Module.CurrentWeather {
    Temperature = temperature,
    WindSpeed = windSpeed,
    Latitude = latitude,
    ObservationTime = observationTime
  }
};

-- 4. REST Client
create rest client Module.WeatherAPI (
  BaseUrl: 'https://api.open-meteo.com/v1',
  authentication: none
)
{
  operation GetCurrent {
    method: get,
    path: '/forecast',
    query: ($latitude: decimal, $longitude: decimal, $current: string),
    headers: ('Accept' = 'application/json'),
    response: json as $Result
  }
};

-- 5. Microflow (REST Client → Transform → Import)
create microflow Module.ACT_GetWeather ()
returns Module.CurrentWeather as $Weather
begin
  send rest request Module.WeatherAPI.GetCurrent;
  declare $RawJson string = $latestHttpResponse/content;
  $SimplifiedJson = transform $RawJson with Module.SimplifyWeather;
  $Weather = import from mapping Module.IMM_Weather($SimplifiedJson);
  return $Weather;
end;
/
```
</complete_pipeline_example>

<validation_rules>
| Rule | Error | Fix |
|------|-------|-----|
| Every operation MUST have an Accept header | CE7062 | Auto-added by serializer if missing |
| POST/PUT/PATCH MUST have a body | CE7064 | Auto-added by serializer (empty JSON body) |
| Template placeholders must match parameters | CE7056 | `{name}` requires a parameter named `name` |
| No custom error handling on SEND REST REQUEST | CE6035 | Always uses abort-on-error semantics |
| Data Transformer requires 11.9+ | version check | `checkFeature("integration", "data_transformer", ...)` |
</validation_rules>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
