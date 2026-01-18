---
description: Architectural reference for Modifier
---

# Modifier

**Package:** com.hypixel.hytale.server.core.modules.entitystats.modifier
**Type:** Base Class / Data Model

## Definition
```java
// Signature
public abstract class Modifier implements NetworkSerializable<com.hypixel.hytale.protocol.Modifier> {
```

## Architecture & Concepts
The Modifier class is an abstract base for any operation that alters an entity statistic. It forms the foundation of the server-side stat calculation system, providing a polymorphic contract for applying changes to values like health, damage, or speed. This design allows for a flexible and extensible system where different types of modifiers (e.g., additive, multiplicative, overriding) can be defined and applied without the core stat system needing to know their specific implementation details.

Its primary role is to encapsulate a single mathematical transformation. The class is deeply integrated with the Hytale codec system, evidenced by the static `CodecMapCodec`. This indicates that modifiers are intended to be data-driven, defined in external configuration files (like item or ability definitions) and deserialized at runtime.

The implementation of `NetworkSerializable` suggests that some state about active modifiers can be synchronized with the client, but the `toPacket` method contains a critical limitation: it only supports `StaticModifier` subclasses. This implies that complex, dynamic server-side modifiers cannot be fully represented on the client.

## Lifecycle & Ownership
-   **Creation:** Modifier instances are primarily created by the `CodecMapCodec` during the deserialization of game data from configuration files. They can also be instantiated programmatically by game logic, for example, when a temporary status effect is applied to an entity.
-   **Scope:** The lifetime of a Modifier is strictly tied to its owner. If it is part of an item's definition, it is long-lived and shared. If it is part of a temporary buff on an entity, it persists only for the duration of that buff.
-   **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for cleanup once they are no longer referenced by any component, such as when a buff expires or an entity is removed from the world.

## Internal State & Concurrency
-   **State:** The base class holds minimal, immutable-after-construction state, specifically the `ModifierTarget`. Concrete subclasses are expected to contain the actual values and logic for the modification, which may or may not be mutable.
-   **Thread Safety:** This class is **not thread-safe** by default. While the base state is immutable, it is designed to be used within the single-threaded server game loop. Subclasses could introduce mutable state, making concurrent access from multiple threads dangerous. All interactions with Modifier instances should be synchronized externally or confined to the main server thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(float var1) | abstract float | Varies | Abstract contract for applying the modification to a base value. |
| getTarget() | ModifierTarget | O(1) | Returns whether the modifier targets the MIN or MAX value of a stat. |
| toPacket() | com.hypixel.hytale.protocol.Modifier | O(1) | Serializes the modifier to a network packet. **WARNING:** Throws UnsupportedOperationException if the instance is not a StaticModifier. |

## Integration Patterns

### Standard Usage
Modifiers are typically managed by a higher-level system, such as a stat component on an entity. This system collects all relevant modifiers and applies them in a deterministic order to calculate a final stat value.

```java
// Conceptual example of a stat calculation system
float baseHealth = 100.0f;
float finalHealth = baseHealth;

List<Modifier> activeModifiers = entity.getStatComponent().getModifiersFor(Stat.HEALTH);

for (Modifier modifier : activeModifiers) {
    finalHealth = modifier.apply(finalHealth);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation of Abstract Class:** Do not attempt `new Modifier()`. You must instantiate a concrete subclass like `StaticModifier`.
-   **Network Serialization of Dynamic Modifiers:** Do not call `toPacket` on any Modifier subclass that is not a `StaticModifier`. This will result in a server crash. This is a critical system constraint.
-   **Unordered Application:** Applying modifiers in a non-deterministic order can lead to different outcomes, especially when mixing additive and multiplicative modifiers. A central system must always control the application sequence.

## Data Pipeline
The Modifier class acts as a processing node within two primary data pipelines: configuration loading and network synchronization.

> **Configuration Flow:**
> Game Data (JSON/Config File) -> Hytale Codec System -> **Modifier Instance** -> Entity Stat Component

> **Network Synchronization Flow:**
> **Modifier Instance** (Server-side) -> `toPacket()` -> Protocol Buffer -> Network Layer -> Client Game State

