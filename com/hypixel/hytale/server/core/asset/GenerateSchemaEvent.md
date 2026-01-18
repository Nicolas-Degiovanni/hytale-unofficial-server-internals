---
description: Architectural reference for GenerateSchemaEvent
---

# GenerateSchemaEvent

**Package:** com.hypixel.hytale.server.core.asset
**Type:** Transient

## Definition
```java
// Signature
public class GenerateSchemaEvent implements IEvent<Void> {
```

## Architecture & Concepts
The GenerateSchemaEvent is a specialized event object used within the server's asset compilation and build pipeline. It functions as a transient data container and a command object, designed to orchestrate the aggregation of various data schemas from disparate parts of the codebase.

Its primary role is to decouple schema providers from the central schema generation process. A core system initiates the process by firing this event. Various listeners, each responsible for a specific domain (e.g., block definitions, NPC behaviors, item configurations), receive the event and contribute their schemas to the shared payload.

This pattern allows for a modular and extensible asset system. New content types can provide their own schemas simply by registering a new event listener, without modifying the core generation logic. The event also carries a BsonDocument specifically for generating Visual Studio Code configuration, indicating its role extends to developer tooling and improving the content creation workflow.

## Lifecycle & Ownership
- **Creation:** Instantiated by a high-level asset management or build system at the beginning of a schema generation pass. The creator provides the initial, often empty, collections that will be populated by listeners.
- **Scope:** Extremely short-lived. An instance of GenerateSchemaEvent exists only for the duration of a single event dispatch cycle. It is created, passed sequentially to all relevant listeners, and then immediately becomes eligible for garbage collection.
- **Destruction:** The object is dereferenced by the event bus after all listeners have been invoked. There is no explicit cleanup or destruction method. Listeners must not retain a reference to the event after their handler method completes.

## Internal State & Concurrency
- **State:** Highly mutable. While the direct field references are final, the `schemas` map and `vsCodeConfig` BsonDocument they point to are intended to be modified by multiple listeners. The event acts as a shared, mutable context that is progressively built up during the event dispatch.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be processed by a single thread within the event bus. Concurrent modification of the internal map or BsonDocument from multiple threads will result in race conditions and data corruption. All operations on a given instance must be performed synchronously within the scope of an event handler.

## API Surface
The public API is designed for contribution. Listeners add data to the event payload; they do not typically read data from it.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addSchemaLink(name, paths, extension) | void | O(N) | Registers a schema to be associated with a set of file paths in the VS Code configuration. N is the number of paths provided. Modifies the internal BsonDocument state. |
| addSchema(fileName, schema) | void | O(1) | Adds a compiled Schema object to the central map, keyed by its intended output filename. Modifies the internal Map state. |

## Integration Patterns

### Standard Usage
The canonical use case is within an event listener that responds to the schema generation process. The listener receives the event and uses its methods to contribute its specific schemas.

```java
// Example of a listener contributing block schemas
@Subscribe
public void onGenerateSchemas(GenerateSchemaEvent event) {
    // Create a schema for block definitions
    Schema blockSchema = buildBlockSchema();
    event.addSchema("blocks", blockSchema);

    // Associate the schema with all .block files for IDE support
    List<String> blockFilePaths = List.of("content/blocks/**/*.block");
    event.addSchemaLink("blocks", blockFilePaths, ".block");
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Listeners must never create their own instance with `new GenerateSchemaEvent()`. The event is a message passed from a central authority and must not be fabricated by a consumer.
- **Storing References:** Do not cache or store a reference to the event object in a field. Its state is only valid for the immediate scope of the event handler method.
- **Asynchronous Modification:** Do not offload modifications to another thread. The event bus expects the event to be fully processed upon the handler's return. Asynchronous writes will cause severe race conditions.

## Data Pipeline
The GenerateSchemaEvent acts as a mobile container that aggregates data as it moves through the event bus.

> Flow:
> Asset Build Coordinator -> Instantiates **GenerateSchemaEvent** -> Event Bus Dispatch -> Listener 1 (adds NPC schemas) -> Listener 2 (adds Item schemas) -> Listener N (...) -> Asset Build Coordinator -> Serializes aggregated schemas and config to disk.

