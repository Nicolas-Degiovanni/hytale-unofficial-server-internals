---
description: Architectural reference for SupportDropType
---

# SupportDropType

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Enum

## Definition
```java
// Signature
public enum SupportDropType {
```

## Architecture & Concepts
The SupportDropType enum defines a constrained set of behaviors for how a block should react when its supporting block is removed. It serves as a type-safe configuration value within the broader block asset definition system, ensuring that block behaviors are predictable and free from data entry errors.

This enum is not a service or a manager; it is a fundamental data type. Its primary architectural significance lies in its integration with the asset serialization pipeline, indicated by the static EnumCodec field. This allows human-readable strings within asset files (e.g., JSON) to be safely decoded into their corresponding in-memory enum constants during server initialization. It represents a contract between the raw asset data and the game engine's runtime representation.

The two defined values represent distinct gameplay mechanics:
*   **BREAK:** The block drops its item only when the support is broken through normal gameplay actions (e.g., a player mining it).
*   **DESTROY:** The block drops its item regardless of how the support is removed, including environmental effects, explosions, or command-based world edits.

## Lifecycle & Ownership
-   **Creation:** Enum constants are instantiated automatically by the Java Virtual Machine (JVM) during class loading. The static CODEC instance is also initialized at this time. This process is managed entirely by the JVM and occurs early in the server startup sequence.
-   **Scope:** Application-wide. The enum constants persist for the entire lifetime of the server process. They are global, immutable singletons.
-   **Destruction:** The constants are garbage collected by the JVM when the application shuts down and its classloader is unloaded. No manual cleanup is required or possible.

## Internal State & Concurrency
-   **State:** Inherently immutable. The state of an enum constant cannot be modified after its creation.
-   **Thread Safety:** This class is fully thread-safe. As immutable singletons, its constants can be safely accessed and passed between any number of threads without requiring synchronization or locks. The static CODEC field is also designed for concurrent access during asset loading.

## API Surface
The public contract consists of the enum constants themselves and the static codec for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BREAK | SupportDropType | O(1) | Constant representing that a block drops when its support is broken. |
| DESTROY | SupportDropType | O(1) | Constant representing that a block drops when its support is destroyed by any means. |
| CODEC | EnumCodec | O(1) | A static codec instance used by the asset system to serialize and deserialize this enum from data files. |

## Integration Patterns

### Standard Usage
This enum is not intended to be used procedurally. It is primarily referenced within asset definition files or used by the asset loading system. Developers will typically encounter it as a property of a larger block configuration object.

```java
// Example of how the asset loader might use the CODEC
// This is a conceptual example of the deserialization process.
String rawValue = "BREAK"; // Value from a JSON file
SupportDropType type = SupportDropType.CODEC.decode(context, rawValue);

// In-game logic would then check the value
if (blockConfig.getSupportDropType() == SupportDropType.DESTROY) {
    // ... logic to drop item
}
```

### Anti-Patterns (Do NOT do this)
-   **String Comparison:** Do not compare the enum to a raw string. This defeats the purpose of type safety and is highly error-prone.
    -   BAD: `if (blockConfig.getSupportDropType().name().equals("BREAK"))`
    -   GOOD: `if (blockConfig.getSupportDropType() == SupportDropType.BREAK)`
-   **Extensibility:** Enums in Java are final and cannot be extended. Do not attempt to create subclasses. New drop types must be added directly to the enum definition, which constitutes a source code modification.

## Data Pipeline
SupportDropType acts as a data model within the server's asset loading pipeline. It is the deserialized, in-memory representation of a configuration value defined in a text-based asset file.

> Flow:
> Block Asset File (e.g., JSON) -> Asset Deserializer -> **EnumCodec** -> **SupportDropType Constant** -> BlockTypeConfig Object -> Game Simulation

