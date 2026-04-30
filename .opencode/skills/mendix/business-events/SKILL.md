---
name: 'business-events'
description: 'Business Events — CREATE/DROP business event services, message definition syntax, publish/subscribe, and calling PublishBusinessEvent_V2 from microflows'
compatibility: opencode
---

<role>MDL business event author — define event-driven APIs with Kafka/message broker publish and subscribe channels.</role>

<summary>
> Syntax reference for creating and managing business event services. Covers CREATE/DROP, message definition syntax, service properties, publish/subscribe operations, and the microflow pattern for publishing events using Java actions.
</summary>

<triggers>
Load when:
- User wants to define event-driven APIs using Kafka/message brokers
- Creating business event services that publish events
- Viewing or describing existing business event services
- Setting up publish/subscribe message channels
</triggers>

<commands>

### View Business Events

```sql
-- List all business event service documents
show business event services;

-- Filter by module
show business event services in MyModule;

-- List all business event client documents (future)
show business event clients;

-- List individual messages across all services
show business events;

-- Filter messages by module
show business events in MyModule;

-- Full MDL description (round-trippable)
describe business event service Module.ServiceName;
```

### Create a Business Event Service

```sql
create business event service Module.CustomerEventsApi
(
  ServiceName: 'CustomerEventsApi',
  EventNamePrefix: 'com.example'
)
{
  message CustomerChangedEvent (CustomerId: long) publish
    entity Module.PBE_CustomerChangedEvent;
  message AddressChangedEvent (AddressId: long) publish
    entity Module.PBE_AddressChangedEvent;
};
```

### Create or Replace (Overwrite Existing)

```sql
create or replace business event service Module.CustomerEventsApi
(
  ServiceName: 'CustomerEventsApi',
  EventNamePrefix: ''
)
{
  message CustomerChangedEvent (CustomerId: long) publish
    entity Module.PBE_CustomerChangedEvent;
};
```

### Drop a Business Event Service

```sql
drop business event service Module.CustomerEventsApi;
```

</commands>

<message_definition_syntax>

```
message <MessageName> (<AttrName>: <type>, ...) publish|subscribe
  [entity <Module.EntityName>]
  [microflow <Module.MicroflowName>];
```

### Supported Attribute Types
- `string` - Text
- `integer` - 32-bit integer
- `long` - 64-bit integer
- `decimal` - Precise decimal number
- `boolean` - True/false
- `datetime` - Date and time

</message_definition_syntax>

<service_properties>

| Property | Description |
|----------|-------------|
| `ServiceName` | The service name used in the event broker |
| `EventNamePrefix` | Prefix added to event names (can be empty) |
| `folder` | Optional folder path for the service document |

| Operation | Description |
|-----------|-------------|
| `publish` | This service publishes the event (other apps subscribe) |
| `subscribe` | This service subscribes to the event (other apps publish) |

</service_properties>

<publishing_from_microflows>

There is no dedicated microflow activity for publishing business events. Use `call java action` with the BusinessEvents marketplace module:

```sql
-- Create an event entity instance and publish it
create microflow Module.ACT_PublishCustomerChanged
  folder 'ACT'
begin
  declare $event Module.PBE_CustomerChangedEvent;
  $event = create Module.PBE_CustomerChangedEvent (CustomerId = $CustomerId);
  commit $event;
  call java action BusinessEvents.PublishBusinessEvent_V2(EventObject = $event);
end;
```

### Available Java Actions (from BusinessEvents module)

| Java Action | Description |
|-------------|-------------|
| `BusinessEvents.PublishBusinessEvent_V2` | Publish an event (recommended) |
| `BusinessEvents.PublishBusinessEvent` | Publish an event (legacy) |
| `BusinessEvents.ConsumeBusinessEvent` | Consume/acknowledge an event |
| `BusinessEvents.PublishEvents` | Publish multiple events |
| `BusinessEvents.StartupBusinessEvents` | Initialize the event broker connection |
| `BusinessEvents.ShutdownBusinessEvents` | Shut down the event broker connection |

### Typical Pattern

1. Define a Business Event Service with PUBLISH messages
2. Create entities prefixed with `PBE_` that **extend `BusinessEvents.PublishedBusinessEvent`**
3. Entity attributes must **exactly match** the message attributes (no extra attributes on the entity)
4. In microflows, create an instance of the event entity, populate its attributes, commit it
5. Call `BusinessEvents.PublishBusinessEvent_V2` passing the entity instance
6. For subscribe operations, link a handler microflow in the service definition

</publishing_from_microflows>

<checklist>

- [ ] Linked entities must exist before creating the service
- [ ] **Entities must extend `BusinessEvents.PublishedBusinessEvent`** (for published events)
- [ ] **Entity attributes must exactly match the message attributes** (no extra/missing attributes)
- [ ] Entity names use qualified format: `Module.EntityName`
- [ ] Entities for published events are conventionally prefixed with `PBE_`
- [ ] Attribute types must be valid (String, Integer, Long, Decimal, Boolean, DateTime)
- [ ] Use `describe business event service` to verify the result
- [ ] The DESCRIBE output is parseable and can be used as a CREATE statement
- [ ] To publish events from microflows, use `call java action BusinessEvents.PublishBusinessEvent_V2`
- [ ] The `BusinessEvents` module must be included in the project (marketplace module)

</checklist>

<output_rules>Output MDL code only in code blocks. Keep explanations concise.</output_rules>
