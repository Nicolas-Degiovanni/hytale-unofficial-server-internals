---
description: Architectural reference for VirtualPath
---

# VirtualPath

**Package:** com.hypixel.hytale.codec.schema.metadata
**Type:** Transient / Data Transfer Object

## Definition
```java
// Signature
public class VirtualPath implements Metadata {
```

## Architecture & Concepts
The VirtualPath class is a simple, immutable data carrier that encapsulates a single piece of metadata: a resource's virtual path. It exists as part of the schema configuration system and implements the Metadata interface, signifying its role as a schema modifier.

Architecturally, this class follows the **Command Pattern**. Each instance represents a single, discrete operation—setting the virtual path—that can be applied to a target Schema object. This design decouples the schema-building logic from the specific metadata details, allowing various Metadata implementations to be applied interchangeably to configure a Schema object. It is not a service or manager, but rather a short-lived instruction object.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by configuration parsers or schema builders. A new VirtualPath is typically created when a source file (e.g., a JSON asset definition) containing a virtual path property is processed.
- **Scope:** Extremely short-lived. An instance's lifecycle is typically confined to the duration of a single schema modification transaction. It is created, its modify method is called, and it is then immediately eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or explicit cleanup requirements.

## Internal State & Concurrency
- **State:** Immutable. The internal *path* field is private and final, assigned only once during construction. The object's state cannot be changed after creation.
- **Thread Safety:** Inherently thread-safe due to its immutability. The same VirtualPath instance can be safely shared and used across multiple threads.
    - **Warning:** While the VirtualPath object itself is thread-safe, the Schema object passed to its modify method may not be. External synchronization is required if multiple threads are modifying the same Schema instance concurrently.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modify(Schema schema) | void | O(1) | Applies the encapsulated path to the target Schema. Throws NullPointerException if the schema parameter is null. |

## Integration Patterns

### Standard Usage
The intended use is to instantiate VirtualPath with a path string and immediately apply it to a Schema object as part of a larger configuration process.

```java
// A Schema object is retrieved or created
Schema schemaToConfigure = new Schema();

// A VirtualPath is created, often from parsed data
VirtualPath pathMetadata = new VirtualPath("hytale/world/zone1/props.json");

// The metadata is applied to the target schema
pathMetadata.modify(schemaToConfigure);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not hold references to VirtualPath instances expecting their internal state to change. They are immutable and should be treated as single-use command objects.
- **Null Arguments:** Do not construct a VirtualPath with a null string. While the constructor may permit this, it will lead to a NullPointerException when the modify method is eventually called on a valid Schema.

## Data Pipeline
VirtualPath acts as a carrier in a configuration pipeline, not a data processing pipeline. It transports a configuration detail from a source to a target object.

> Flow:
> Configuration Source (e.g., Asset Definition File) -> Deserializer/Parser -> **new VirtualPath(...)** -> `modify(Schema)` -> In-Memory Schema Object

