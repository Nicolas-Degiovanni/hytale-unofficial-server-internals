---
description: Architectural reference for SchemaContext
---

# SchemaContext

**Package:** com.hypixel.hytale.codec.schema
**Type:** Transient

## Definition
```java
// Signature
public class SchemaContext {
```

## Architecture & Concepts
The SchemaContext is the central state manager for the JSON Schema generation process. Its primary responsibility is to translate the engine's internal `SchemaConvertable` and `BuilderCodec` objects into a formal, structured, and de-duplicated JSON Schema representation.

It functions as a **schema registry and definition cache**. During a schema generation run, the context ensures that any given data structure (represented by a codec) is defined only once. All subsequent encounters with that same structure are replaced with a JSON Schema `$ref` pointer to the canonical definition. This mechanism is critical for two reasons:

1.  **Handling Recursion:** It correctly models recursive data structures without causing infinite loops during generation.
2.  **Efficiency:** It significantly reduces the size and complexity of the final schema output by eliminating redundant definitions.

The context also manages the resolution of unique names for schema definitions, automatically handling collisions that may arise from different classes sharing the same simple name.

## Lifecycle & Ownership
-   **Creation:** A SchemaContext is instantiated manually at the beginning of a schema generation task. It is not a managed service and is not retrieved from a central registry. A new instance must be created for each distinct generation run.
-   **Scope:** The object's lifetime is strictly bound to a single, complete schema generation operation. It accumulates state, such as resolved names and cached definitions, throughout this process.
-   **Destruction:** Once the top-level schema has been generated and the resulting definition maps have been extracted via `getDefinitions`, the SchemaContext instance holds no further value and is eligible for garbage collection. It does not manage any native resources and has no explicit `close` or `destroy` method.

## Internal State & Concurrency
-   **State:** The SchemaContext is highly mutable and stateful by design. Its internal maps are populated as codecs are processed, effectively building a complete picture of the schema's structure over its lifetime. The order of operations can affect the final output, particularly the insertion order of definitions in the `Object2ObjectLinkedOpenHashMap`.

-   **Thread Safety:** **This class is not thread-safe.** All internal state, including definition caches and name collision counters, is unsynchronized. Concurrent access will lead to race conditions, resulting in corrupted internal state, lost definitions, and non-deterministic schema output.

    **WARNING:** A single SchemaContext instance must only be accessed and mutated from a single thread. Do not share instances across threads.

## API Surface
The public API is designed to drive a schema generation process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addFileReference(fileName, codec) | void | O(1) | Pre-registers a codec as belonging to an external file, ensuring it will be referenced via a file path. |
| refDefinition(convertable, def) | Schema | Amortized O(1) | The primary entry point. Returns a Schema, which is typically a `$ref` to a definition managed by the context. The first call for a new type triggers schema generation and caching. |
| getDefinitions() | Map | O(1) | Returns the final map of generated "common" schema definitions. This is typically called at the end of the process. |
| getOtherDefinitions() | Map | O(1) | Returns the final map of generated "other" schema definitions, used for types that are not standard `BuilderCodec` instances. |

## Integration Patterns

### Standard Usage
The intended pattern is to create a context, use it to recursively process a root-level codec, and then extract the generated definitions for serialization.

```java
// How a developer should normally use this
SchemaContext context = new SchemaContext();

// Process the root object codec to populate the context
Schema rootSchema = context.refDefinition(MyRootObject.CODEC);

// At the end, extract the shared definitions
Map<String, Schema> definitions = context.getDefinitions();

// Now, serialize the root schema and the definitions to JSON files
JsonSerializer.serialize("MyRootObject.json", rootSchema);
JsonSerializer.serialize("common.json", definitions);
```

### Anti-Patterns (Do NOT do this)
-   **Instance Reuse:** Do not reuse a SchemaContext instance for separate, unrelated schema generation tasks. The cached definitions and name collision counts from the first run will pollute the results of the second. Always create a new instance.
-   **Concurrent Modification:** Do not access a SchemaContext from multiple threads. This will corrupt its internal state and is an unsupported usage pattern.
-   **Post-facto Modification:** Do not modify the maps returned by `getDefinitions()`. While technically possible, they are intended as a final, read-only result of the generation process.

## Data Pipeline
The SchemaContext transforms high-level codec objects into low-level, reference-based Schema structures.

> Flow:
> SchemaConvertable Object -> `refDefinition` call -> **SchemaContext** (Performs Name Resolution, Caching, and De-duplication) -> `Schema` Object (often a `$ref`) -> JSON Serialization Layer

