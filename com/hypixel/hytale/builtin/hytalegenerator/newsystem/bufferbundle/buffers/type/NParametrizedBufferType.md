---
description: Architectural reference for NParametrizedBufferType
---

# NParametrizedBufferType

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.bufferbundle.buffers.type
**Type:** Transient / Value Object

## Definition
```java
// Signature
public class NParametrizedBufferType extends NBufferType {
```

## Architecture & Concepts
NParametrizedBufferType is a specialized descriptor class that extends NBufferType to add a generic type parameter to a buffer's definition. It serves as a type-safe identifier for buffers that are not only defined by their container class (e.g., a list-based buffer) but also by the type of element they contain.

In the context of the world generation system, a base NBufferType might define a generic "HeightmapBuffer". An NParametrizedBufferType provides greater specificity, defining, for example, a "HeightmapBuffer" that is explicitly parameterized with the type "Float". This allows the generation pipeline to enforce stricter type contracts, reducing the need for runtime casting and preventing class cast exceptions.

This class is fundamentally a metadata object. It is used as a key in collections and as a formal definition during the registration and retrieval of buffers from a central management system, such as a BufferBundle. It does not hold any world data itself; it merely describes the structure and type of a buffer that *will* hold data.

### Lifecycle & Ownership
-   **Creation:** Instances are created during the bootstrap phase of the world generator. A central registry or configuration service defines all known buffer types, including parametrized ones, before any generation tasks begin.
-   **Scope:** These objects are intended to be long-lived. Once defined, they persist for the entire server session, acting as static, canonical definitions for buffer types.
-   **Destruction:** Instances are de-referenced and eligible for garbage collection only when the world generator system is fully shut down or reconfigured, at which point the entire set of buffer definitions is discarded. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** **Immutable**. All fields, including the critical `parameterClass`, are declared final and are set exclusively within the constructor. This immutability is essential for its role as a reliable key in hash-based collections.
-   **Thread Safety:** **Fully thread-safe**. As an immutable object, an instance of NParametrizedBufferType can be safely shared, read, and used as a key across multiple threads in a parallelized world generation environment without requiring any locks or synchronization.

## API Surface
The public contract is focused on validation and identity, consistent with its role as a type descriptor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isValidType(bufferClass, parameterClass) | boolean | O(1) | Performs a strict check, validating that both the buffer's base class and its parameter class match this type definition. |
| isValid(buffer) | boolean | O(1) | **Warning:** This overridden method only validates that the buffer is an instance of the defined `bufferClass`. It does *not* perform a runtime check of the buffer's generic parameter. |
| equals(o) | boolean | O(1) | Provides value-based equality, comparing the name, index, buffer class, and the specific `parameterClass`. |
| hashCode() | int | O(1) | Generates a hash code based on all defining properties, including the `parameterClass`. |

## Integration Patterns

### Standard Usage
This class is intended to be used as a static definition or key when interacting with a buffer management system. A generator stage will use a predefined NParametrizedBufferType instance to request a correctly typed buffer.

```java
// Example: Defining a buffer type for a list of BlockState objects
NParametrizedBufferType blockStateListType = new NParametrizedBufferType(
    "hytale:block_state_list",
    BUFFER_ID_BLOCKS,
    NListBuffer.class,
    BlockState.class,
    NListBuffer::new
);

// A generator stage would then retrieve a buffer of this specific type
// from a central BufferBundle or similar manager.
NListBuffer<BlockState> blockBuffer = bufferBundle.getBuffer(blockStateListType);
blockBuffer.add(new BlockState("stone"));
```

### Anti-Patterns (Do NOT do this)
-   **Runtime Generic Check:** Do not rely on the `isValid(NBuffer buffer)` method to guarantee the generic type of a buffer instance. Java's type erasure prevents this. The parameterization is a design-time contract enforced by the system's architecture, not by this specific method call.
-   **Direct Instantiation in Stages:** Generator stages should not create new instances of NParametrizedBufferType. They should reference statically defined, canonical instances provided by a central registry to ensure type consistency across the entire pipeline.
-   **Reference Comparison:** Never compare NParametrizedBufferType instances using the `==` operator. Always use the `.equals()` method to ensure a correct value-based comparison.

## Data Pipeline
NParametrizedBufferType is not part of a data processing pipeline. Instead, it is a foundational component of the system's *configuration and setup*, enabling the data pipeline to function correctly.

> Flow:
> World Generator Initialization -> **NParametrizedBufferType Definition** -> Registration in BufferBundle -> Generator Stage uses Type as a key -> BufferBundle allocates and returns a matching Buffer instance -> Generator Stage populates Buffer with data

