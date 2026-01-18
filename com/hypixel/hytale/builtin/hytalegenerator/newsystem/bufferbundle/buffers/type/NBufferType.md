---
description: Architectural reference for NBufferType
---

# NBufferType

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.bufferbundle.buffers.type
**Type:** Value Object

## Definition
```java
// Signature
public class NBufferType {
```

## Architecture & Concepts
NBufferType is an immutable descriptor object that serves as a schema for a specific type of NBuffer within the world generation system. It is not a service or a manager; rather, it is a static definition that provides the blueprint for creating and identifying buffers.

This class encapsulates three critical pieces of information for a buffer:
1.  **Identity:** A unique integer *index* for fast, unambiguous lookups and comparisons.
2.  **Type Contract:** The concrete *bufferClass* that this type represents, ensuring type safety.
3.  **Instantiation Logic:** A *bufferSupplier* which acts as a factory for creating new instances of the associated NBuffer.

By bundling this information, NBufferType decouples the systems that request or consume buffers from the concrete implementation details of how those buffers are created. It forms the foundation of a registry-based system, where these type definitions are used as keys to manage and access different data layers in a generation pipeline. The *name* field is provided for debugging and tooling purposes only and is explicitly excluded from equality checks.

## Lifecycle & Ownership
-   **Creation:** NBufferType instances are intended to be created once at application startup and stored as static final constants in a central registry or constants class. They represent the fixed, known set of buffer types available to the generator.
-   **Scope:** Application-scoped. Once defined, these objects persist for the entire lifetime of the application. They are static metadata.
-   **Destruction:** There is no explicit destruction logic. Instances are garbage collected during JVM shutdown.

## Internal State & Concurrency
-   **State:** **Immutable**. All fields are declared final and are set exclusively through the constructor. An NBufferType object's state cannot be modified after creation.
-   **Thread Safety:** **Inherently thread-safe**. Due to its immutability, a single NBufferType instance can be safely shared and read across multiple threads without any need for locks or other synchronization primitives.

## API Surface
The public API consists of validation methods and standard object methods. The primary interaction is through the constructor during initialization and by accessing its public final fields at runtime.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isValidType(Class) | boolean | O(1) | Checks if a given class matches this type's defined buffer class. |
| isValid(NBuffer) | boolean | O(1) | Checks if a given buffer instance is compatible with this type. |
| equals(Object) | boolean | O(1) | Compares identity based on index, bufferClass, and bufferSupplier. **Warning:** The name field is ignored. |
| hashCode() | int | O(1) | Generates a hash code consistent with the equals method. |

## Integration Patterns

### Standard Usage
NBufferType objects should be defined as public static final constants. Other systems use these constants as keys to retrieve or identify buffers from a manager, such as a BufferBundle.

```java
// In a central constants or registry class
public class BufferTypes {
    public static final NBufferType TERRAIN_HEIGHT = new NBufferType(
        "TerrainHeight", 0, HeightBuffer.class, HeightBuffer::new
    );
    public static final NBufferType BIOME_ID = new NBufferType(
        "BiomeID", 1, BiomeBuffer.class, BiomeBuffer::new
    );
}

// In a generator system
BufferBundle bundle = worldGenContext.getBufferBundle();
HeightBuffer heightData = bundle.getBuffer(BufferTypes.TERRAIN_HEIGHT);
```

### Anti-Patterns (Do NOT do this)
-   **Dynamic Instantiation:** Do not create new NBufferType instances at runtime to perform checks. They are meant to be treated as singleton constants. Reference equality (`==`) is the preferred method of comparison.
-   **Relying on Name for Logic:** Never use the `name` field for comparisons or any control flow logic. It is not included in `equals` or `hashCode` and is reserved for debugging.
-   **Mutable Suppliers:** The provided `Supplier<NBuffer>` should not depend on external, mutable state. It should be a pure factory, like a constructor reference (`HeightBuffer::new`), to ensure predictable buffer creation.

## Data Pipeline
NBufferType is not a direct participant in the data processing pipeline. Instead, it is a metadata component used to **define and construct** the data containers (NBuffers) used in the pipeline.

> Flow:
> Generator System Startup -> **NBufferType Registry** (instantiates all types) -> World Generation Task -> Requests buffer from `BufferBundle` using `NBufferType` constant -> `BufferBundle` uses `bufferSupplier` from `NBufferType` -> Creates new `NBuffer` instance -> `NBuffer` is populated and used in pipeline.

