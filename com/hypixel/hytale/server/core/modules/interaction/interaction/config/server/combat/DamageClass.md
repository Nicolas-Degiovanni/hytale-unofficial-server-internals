---
description: Architectural reference for DamageClass
---

# DamageClass

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server.combat
**Type:** Type-Safe Enumeration

## Definition
```java
// Signature
public enum DamageClass {
```

## Architecture & Concepts
The DamageClass enum provides a compile-time constant, type-safe classification for all damage-dealing interactions within the server's combat module. Its primary architectural role is to standardize the categorization of attacks, moving this logic from fragile, string-based identifiers to a robust, immutable type.

This enum is a foundational data structure, not an active service. It acts as a data contract for any system that produces or consumes combat events, such as ability systems, weapon configurations, and network event handlers.

The inclusion of a static **CODEC** field is a critical design choice. It signals that DamageClass is a serializable component, intended to be part of the data transfer layer. This allows the enum to be reliably encoded into network packets or persisted in configuration files, ensuring consistency between different parts of the distributed system (e.g., server and client) or between game sessions.

## Lifecycle & Ownership
-   **Creation:** Enum constants are instantiated by the Java Virtual Machine (JVM) during class loading. This process is automatic and occurs once, very early in the server's startup sequence, before any game logic is executed.
-   **Scope:** The instances (UNKNOWN, LIGHT, CHARGED, SIGNATURE) are static, final, and globally accessible. They persist for the entire lifetime of the server process.
-   **Destruction:** The enum and its instances are garbage collected only when the JVM shuts down. There is no manual memory management.

## Internal State & Concurrency
-   **State:** DamageClass is fundamentally immutable. Its instances are constants with no mutable fields. Their state is defined entirely at compile time.
-   **Thread Safety:** This enum is intrinsically thread-safe. As immutable, globally shared constants, instances of DamageClass can be safely read and passed between any number of threads without requiring locks or any other synchronization primitives.

## API Surface
The public contract consists of the predefined enum constants and the static codec for data handling.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| UNKNOWN | DamageClass | O(1) | A default or fallback category for unclassified damage sources. |
| LIGHT | DamageClass | O(1) | Represents damage from a standard, fast, or low-cost attack. |
| CHARGED | DamageClass | O(1) | Represents damage from a powerful attack that typically requires a wind-up or resource cost. |
| SIGNATURE | DamageClass | O(1) | Represents damage from a unique, ultimate, or character-defining ability. |
| CODEC | EnumCodec | O(1) | Provides static, reusable logic for serializing and deserializing the enum. |

## Integration Patterns

### Standard Usage
DamageClass should be used to define the type of an attack within combat configuration files or event objects. Logic should then use these constants for branching, such as in a switch statement to apply different effects or damage modifiers.

```java
// Example: Processing a combat event
void handleDamageEvent(CombatEvent event) {
    switch (event.getDamageClass()) {
        case LIGHT:
            // Apply standard damage calculation
            break;
        case CHARGED:
            // Apply bonus damage and stagger effects
            break;
        case SIGNATURE:
            // Trigger special visual effects and apply unique debuffs
            break;
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **String Comparison:** Never rely on the string representation of the enum for logical comparisons. Using `damageClass.toString().equals("LIGHT")` is inefficient, error-prone, and defeats the purpose of a type-safe enum. Always use direct object comparison (`==`) or the `equals` method.
-   **Incomplete Switch Statements:** When using a switch statement, always include a `default` case to handle potential future additions to the enum. Failure to do so can result in silent failures where new damage types are not processed.
-   **Extending Functionality:** Do not attempt to add complex behavior to the enum itself. Enums should represent constant data. If behavior differs per class, use a Strategy pattern or a lookup map that uses the DamageClass as a key.

## Data Pipeline
DamageClass is not a processing stage in a pipeline; rather, it is the data that flows through it. It originates from game configuration or runtime combat logic and is embedded within data structures that are passed between systems.

> Flow:
> Weapon Configuration File -> Config Loader (uses **CODEC**) -> Weapon Data Object -> Combat System -> CombatEvent(..., damageClass: **DamageClass.CHARGED**) -> Network Serializer (uses **CODEC**) -> Network Packet

