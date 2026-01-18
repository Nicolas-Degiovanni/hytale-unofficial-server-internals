---
description: Architectural reference for MergedBlockFaces
---

# MergedBlockFaces

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Utility Enum

## Definition
```java
// Signature
public enum MergedBlockFaces {
```

## Architecture & Concepts
The MergedBlockFaces enum is a static configuration utility that provides predefined, named collections of BlockFace constants. Its primary purpose is to simplify the definition of block assets and related game logic. Instead of requiring developers to specify exhaustive lists of individual faces for properties like culling, rendering, or collision, this enum offers logical groupings such as CARDINAL_DIRECTIONS or ALL.

This component acts as a domain-specific language enhancement for block configuration. It improves readability and reduces the potential for errors in asset files. The static CODEC field indicates its direct integration with the engine's serialization and deserialization pipeline, allowing these named groups to be used directly in configuration files (e.g., JSON or HOCON) which are then parsed into in-memory objects.

### Lifecycle & Ownership
- **Creation:** Enum instances are created and initialized by the Java Virtual Machine (JVM) during class loading. There is no manual instantiation.
- **Scope:** Application-wide static constants. These instances persist for the entire lifetime of the server or client process.
- **Destruction:** Instances are destroyed only when the application's class loader is garbage collected, which effectively means upon application shutdown.

## Internal State & Concurrency
- **State:** The state of each enum constant is immutable. Each instance holds a final reference to an array of its constituent BlockFace components.
- **Thread Safety:** MergedBlockFaces is inherently thread-safe. As an immutable, globally accessible set of constants, it can be safely read from any thread without synchronization.

**WARNING:** The array returned by getComponents is a direct reference to the internal state. Modifying this array is a severe violation of the component's contract and will lead to unpredictable, global side effects. It should be treated as a read-only structure.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponents() | BlockFace[] | O(1) | Returns the underlying array of BlockFace constants that constitute the group. |

## Integration Patterns

### Standard Usage
This enum is not typically used directly in procedural code but is instead referenced within data-driven asset definitions. The engine's asset loader uses the provided CODEC to deserialize string identifiers into the corresponding enum instance.

A conceptual block definition might look like this:
```json
// Example: conceptual block asset file
{
  "name": "fancy_stone_pillar",
  "rendering": {
    "cullFaces": "BLOCK_SIDES",
    "specialEffectFaces": "UP_CARDINAL_DIRECTIONS"
  }
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not modify the array returned by getComponents. This breaks the immutability guarantee and will affect all systems that reference the enum constant.
    ```java
    // CRITICAL ERROR: Do not do this.
    BlockFace[] faces = MergedBlockFaces.CARDINAL_DIRECTIONS.getComponents();
    faces[0] = BlockFace.UP; // Corrupts global state
    ```
- **Unsafe Deserialization:** Avoid using Enum.valueOf() with untrusted input. The provided CODEC is the designated mechanism for safe deserialization from asset files.

## Data Pipeline
MergedBlockFaces serves as a configuration data type, not a processing component. Its role is at the beginning of the asset loading pipeline.

> Flow:
> Block Asset File (JSON/HOCON) -> Engine Codec Deserializer -> **MergedBlockFaces** Instance -> In-Memory BlockType Configuration -> Rendering & Physics Systems

