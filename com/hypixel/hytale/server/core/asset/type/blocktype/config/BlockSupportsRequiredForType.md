---
description: Architectural reference for BlockSupportsRequiredForType
---

# BlockSupportsRequiredForType

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Utility

## Definition
```java
// Signature
public enum BlockSupportsRequiredForType {
   Any,
   All;
}
```

## Architecture & Concepts
BlockSupportsRequiredForType is a configuration primitive that defines the logical rule for evaluating a block's structural support requirements. It is a critical component of the block physics and world generation systems, dictating whether a block can be placed or remain stable based on its neighboring support blocks.

This enum acts as a type-safe constant, replacing ambiguous booleans or integers in block configuration files. It ensures that the logic for support validation is explicit and constrained to a known set of behaviors.

- **Any:** The block requires at least one of its specified support blocks to be present for stability. This models behaviors like vines needing any part of a tree, or a torch needing any single wall surface.
- **All:** The block requires all of its specified support blocks to be present. This is used for more complex structures, like multi-block portals or machinery that must be fully anchored.

## Lifecycle & Ownership
- **Creation:** As a Java enum, instances are created by the Java Virtual Machine during class loading. These instances are compile-time constants. There is no dynamic instantiation.
- **Scope:** The enum and its constants (Any, All) are static and exist for the entire lifetime of the server process. They are effectively global singletons managed by the JVM.
- **Destruction:** The instances are garbage collected only when the defining ClassLoader is unloaded, which for core engine components, means upon server shutdown.

## Internal State & Concurrency
- **State:** Enum instances are fundamentally immutable. Their state is defined at compile time and cannot be altered at runtime.
- **Thread Safety:** This type is inherently thread-safe. Its instances can be safely accessed and read from any thread without synchronization. This guarantee is provided by the JVM.

## API Surface
The API surface consists solely of the predefined constants.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Any | BlockSupportsRequiredForType | O(1) | Represents a logical OR condition for support blocks. |
| All | BlockSupportsRequiredForType | O(1) | Represents a logical AND condition for support blocks. |

## Integration Patterns

### Standard Usage
This enum is not used directly but is typically deserialized as part of a larger block configuration asset. The game engine then reads this value to apply the correct validation logic.

```java
// Example: Within a hypothetical BlockConfiguration class
// This value would be loaded from a JSON or HOCON asset file.

BlockSupportsRequiredForType rule = blockConfig.getSupportRule();

if (rule == BlockSupportsRequiredForType.All) {
    // ... execute logic that checks if all required neighbors are present
} else {
    // ... execute logic that checks if at least one required neighbor is present
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal-Based Serialization:** Do not serialize or store this enum based on its ordinal value (e.g., 0 for Any, 1 for All). If new constants are added in the future, all existing data will become corrupt. Always serialize using the string name (e.g., "Any").
- **Null Checks:** A field of this type in a well-formed configuration should never be null. Code relying on this enum should treat a null value as a critical asset loading error, not a valid state.

## Data Pipeline
This enum is a terminal piece of configuration data. Its primary flow is from disk to the in-memory representation of a block's properties.

> Flow:
> Block Asset File (e.g., `stone.json`) -> Asset Deserializer -> BlockTypeConfig Object -> **BlockSupportsRequiredForType** field -> Block Placement & Physics System

---

