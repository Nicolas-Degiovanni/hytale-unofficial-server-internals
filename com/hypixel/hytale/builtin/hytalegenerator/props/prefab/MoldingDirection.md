---
description: Architectural reference for MoldingDirection
---

# MoldingDirection

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props.prefab
**Type:** Utility

## Definition
```java
// Signature
public enum MoldingDirection {
```

## Architecture & Concepts
MoldingDirection is a fundamental enumeration that defines a constrained set of cardinal and vertical directions. Its primary role is within the world generation system, specifically for procedural generation of structures from prefabs. It provides a type-safe constant to describe how a "molding" or decorative element should be oriented or extruded from a surface.

The most critical architectural feature of this enum is its direct integration with the engine's serialization framework via the static **CODEC** field. By exposing an EnumCodec, MoldingDirection declares its contract for being serialized to and deserialized from persistent storage, such as world chunk data or prefab definition files. The use of EnumCodec.EnumStyle.LEGACY indicates a specific, non-ordinal-based serialization format that ensures data compatibility across different versions of the enum.

This type is not a service or a manager; it is a core data type, a value object used to pass directional information between different components of the world generator.

## Lifecycle & Ownership
- **Creation:** As a Java enum, all instances (NONE, UP, DOWN, etc.) are created and initialized by the JVM during class loading. They are compile-time constants. There is no dynamic creation of MoldingDirection instances.
- **Scope:** The enum constants are static and exist for the entire lifetime of the application. They are effectively global singletons.
- **Destruction:** The instances are garbage collected only when the ClassLoader that loaded MoldingDirection is itself unloaded, which typically only happens at application shutdown.

## Internal State & Concurrency
- **State:** Enum instances are immutable by design. Their state is fixed at compile time and cannot be altered at runtime.
- **Thread Safety:** MoldingDirection is inherently thread-safe. All instances are constants, and access is always safe from any thread without requiring synchronization. The static CODEC field is final and is safely published after class initialization.

## API Surface
The public API consists solely of the predefined enum constants.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NONE | MoldingDirection | O(1) | Represents the absence of a molding direction. |
| UP | MoldingDirection | O(1) | Represents the positive Y-axis. |
| DOWN | MoldingDirection | O(1) | Represents the negative Y-axis. |
| NORTH | MoldingDirection | O(1) | Represents the negative Z-axis. |
| SOUTH | MoldingDirection | O(1) | Represents the positive Z-axis. |
| EAST | MoldingDirection | O(1) | Represents the positive X-axis. |
| WEST | MoldingDirection | O(1) | Represents the negative X-axis. |
| CODEC | Codec | O(1) | The static serializer/deserializer for this enum. |

## Integration Patterns

### Standard Usage
MoldingDirection is primarily used as a data carrier in world generation algorithms or deserialized from data files.

```java
// Example: Processing a property from a prefab definition
// The 'codec' would be retrieved from a larger data structure definition.
Codec<MoldingDirection> directionCodec = MoldingDirection.CODEC;

// 'dataInput' is a stream of bytes from a file
MoldingDirection direction = directionCodec.decode(dataInput);

switch (direction) {
    case UP:
        // Logic to apply an upward molding
        break;
    case NORTH:
        // Logic to apply a northward molding
        break;
    // ... other cases
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal Comparison:** Do not rely on the `ordinal()` method for serialization or conditional logic. The order of enum constants can change, which would break saved data and corrupt logic. Always use the constant directly (e.g., `if (dir == MoldingDirection.UP)`). The provided CODEC correctly avoids this anti-pattern.
- **String Comparison:** Do not compare the `name()` or `toString()` result with a string literal. This is inefficient and less safe than direct object comparison.
- **Null Usage:** Functions accepting a MoldingDirection should handle the NONE constant explicitly rather than accepting a null value, which can lead to NullPointerExceptions.

## Data Pipeline
MoldingDirection serves as a deserialized data point that drives generator logic.

> Flow:
> Prefab File on Disk -> Engine File I/O -> **EnumCodec (using MoldingDirection.CODEC)** -> **MoldingDirection Instance** -> World Generation Algorithm -> Voxel Placement
---

