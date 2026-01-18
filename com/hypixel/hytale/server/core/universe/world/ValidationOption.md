---
description: Architectural reference for ValidationOption
---

# ValidationOption

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum ValidationOption {
```

## Architecture & Concepts
ValidationOption is a type-safe enumeration that defines the discrete components of a world chunk or region that can be subjected to a validation process. It serves as a set of configuration flags passed to the world validation system, allowing for granular control over the scope and performance of integrity checks.

By representing these options as an enum rather than string literals or integer constants, the system guarantees compile-time safety, prevents invalid option values, and provides a clear, self-documenting contract for validation routines. These options are typically used in combination within a Set or EnumSet to specify a multi-faceted validation task.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine (JVM) during class loading. This process is managed entirely by the JVM and occurs only once.
- **Scope:** The enum and its constants exist for the entire lifetime of the server application. They are effectively static and globally accessible.
- **Destruction:** The constants are garbage collected only when the defining ClassLoader is unloaded, which for core engine components, means upon application shutdown.

## Internal State & Concurrency
- **State:** ValidationOption constants are immutable. Their state is defined at compile time and cannot be altered at runtime.
- **Thread Safety:** As immutable, statically-initialized singletons, enums are inherently thread-safe. They can be safely passed between threads and used in concurrent validation tasks without requiring any synchronization mechanisms.

## API Surface
The API surface consists of the predefined enum constants. These are not methods but static, final instances of ValidationOption.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PHYSICS | Constant | N/A | Specifies validation of physics-related data structures and properties within a world chunk. |
| BLOCKS | Constant | N/A | Specifies validation of block types and their fundamental integrity. |
| BLOCK_STATES | Constant | N/A | Specifies validation of block state data, such as orientation or growth stage. |
| ENTITIES | Constant | N/A | Specifies validation of all entities residing within the target area. |
| BLOCK_FILLER | Constant | N/A | Specifies validation of the procedural generation block filler rules and their results. |

## Integration Patterns

### Standard Usage
ValidationOption is intended to be used within a collection, typically an EnumSet, to configure a validation service or task. This provides a highly efficient and type-safe way to specify multiple validation criteria.

```java
// How a developer should normally use this
import java.util.EnumSet;

// Configure a validation task to check only blocks and entities
EnumSet<ValidationOption> options = EnumSet.of(
    ValidationOption.BLOCKS,
    ValidationOption.ENTITIES
);

WorldValidator validator = world.getValidator();
ValidationReport report = validator.validateChunk(chunkCoordinates, options);
```

### Anti-Patterns (Do NOT do this)
- **Reliance on Ordinal:** Do not use the `ordinal()` method for persistent storage or conditional logic. The order of enum constants is not guaranteed to be stable across code revisions. If a new constant is inserted, all subsequent ordinals will shift, breaking saved data or logic.
- **String Comparison:** Avoid converting the enum to a string for comparison. Use direct object equality (`==`) which is guaranteed to work for enums and is significantly more performant.

## Data Pipeline
ValidationOption acts as a configuration input to a data processing pipeline, not as a stage within it. It dictates which validation sub-routines are executed.

> Flow:
> System Configuration -> `EnumSet<ValidationOption>` -> **WorldValidator** -> Validation Report -> Logging/Correction System

