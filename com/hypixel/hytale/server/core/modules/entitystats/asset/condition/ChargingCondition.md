---
description: Architectural reference for ChargingCondition
---

# ChargingCondition

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.condition
**Type:** Transient Data Object

## Definition
```java
// Signature
public class ChargingCondition extends Condition {
```

## Architecture & Concepts
The ChargingCondition is a specialized predicate class used within the server-side entity statistics and combat systems. Its primary function is to determine if an entity is either actively performing a "charging" interaction or has recently completed one within a configurable time window.

Architecturally, this class acts as a bridge between two distinct server modules:
1.  **InteractionModule:** It queries the live state of an entity's `InteractionManager` to check for any active interactions of the type `ChargingInteraction`. This provides a real-time, instantaneous check.
2.  **Damage System:** It inspects the `DamageDataComponent` for a timestamp of the last charge event. This provides a historical check, allowing for game mechanics that depend on actions immediately following a charge.

This dual-check mechanism makes the condition robust, capable of supporting logic that requires either an in-progress action or a recently completed one. It is designed to be defined in data files (e.g., weapon or ability assets) and deserialized at runtime, making it a key component for data-driven game design.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly using a constructor. They are exclusively instantiated by the Hytale `Codec` system during the server's asset loading phase. The static `CODEC` field defines the deserialization logic, parsing a `Delay` value from a configuration file and constructing a `ChargingCondition` object.
-   **Scope:** The object's lifetime is tied to the asset that defines it. For example, if this condition is part of a weapon's definition, the `ChargingCondition` instance will persist in memory as long as that weapon asset is loaded. It is effectively a static piece of configuration data.
-   **Destruction:** Managed by the Java Garbage Collector. Once the parent asset is unloaded and all references are cleared, the instance is eligible for collection.

## Internal State & Concurrency
-   **State:** The internal state consists of a single field, `delay`, of type `Duration`. This state is considered immutable after deserialization. It is set once during creation by the `CODEC` and is not modified during the object's lifetime.
-   **Thread Safety:** This class is inherently thread-safe. Its only state, `delay`, is immutable. The `eval0` method is a pure function with respect to the class instance; its outcome depends entirely on the arguments passed to it.

    **WARNING:** While the class itself is thread-safe, the `ComponentAccessor` and `Ref` objects passed into `eval0` are not guaranteed to be. The calling system, typically the server's main game loop or a dedicated worker thread, is responsible for ensuring that access to entity component data is properly synchronized.

## API Surface
The public contract is focused entirely on its role as a `Condition`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval0(accessor, ref, time) | boolean | O(1) | Evaluates the condition against a specific entity. Returns true if the entity has an active `ChargingInteraction` or if its `DamageDataComponent` records a charge time within the configured `delay` from the provided `currentTime`. The complexity is technically O(N) on the number of active interactions, but this number is almost always a small constant. |

## Integration Patterns

### Standard Usage
A developer or designer will not interact with this class directly in Java code. Instead, it is configured within an asset file (e.g., JSON or HOCON) that defines a piece of game logic, such as an item ability or an NPC behavior. The engine's stat system then uses the configured condition to perform checks.

A conceptual asset configuration might look like this:
```json
// Example: A weapon stat modifier that applies only while charging
{
  "modifier": "bonus_damage_20_percent",
  "conditions": [
    {
      "type": "ChargingCondition",
      "Delay": "0.5s"
    }
  ]
}
```
In this example, the engine would deserialize the configuration, create a `ChargingCondition` instance with a 500ms delay, and invoke its `eval0` method whenever it needs to check if the `bonus_damage_20_percent` modifier should be active.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructor is `protected`. Do not attempt to create instances with `new ChargingCondition()`. The class is designed to be data-driven and must be created via the `CODEC` system to ensure proper initialization.
-   **State Mutation:** Do not use reflection or other means to modify the `delay` field after the object has been created. The immutability of its configuration is critical for predictable and stable game mechanics.

## Data Pipeline
The flow of data from configuration to execution is managed entirely by the engine's asset and entity-component systems.

> Flow:
> Game Asset File (JSON/HOCON) -> Server Asset Loader -> **ChargingCondition.CODEC** -> In-Memory **ChargingCondition** Instance -> Entity Stat Evaluation System -> `eval0(entityRef)` -> Boolean Result -> Game Logic Change (e.g., Damage Calculation)

