---
description: Architectural reference for EntityStatValue
---

# EntityStatValue

**Package:** com.hypixel.hytale.server.core.modules.entitystats
**Type:** Transient Data Model

## Definition
```java
// Signature
public class EntityStatValue {
```

## Architecture & Concepts

The EntityStatValue class is the runtime representation of a single, dynamic attribute for a game entity, such as Health, Mana, or Armor. It is a fundamental data model within the server-side entity component system, designed to be both performant and flexible.

This class is not a simple container for a floating-point number. It encapsulates a complete state machine for a statistic, including its current value, its operational boundaries (minimum and maximum), and a sophisticated system for applying and recalculating temporary or permanent modifications (e.g., buffs, debuffs, equipment bonuses).

A critical design principle is the separation of the static asset definition (defined in an EntityStatType asset) from the live, mutable instance. This class represents the live instance, which is initialized from an asset but can be dynamically altered during gameplay.

The inclusion of a static CODEC field signifies its role in the engine's data persistence and synchronization pipeline. Instances of EntityStatValue are designed to be serialized for game saving and deserialized for loading, ensuring that an entity's precise state can be preserved.

## Lifecycle & Ownership

-   **Creation:** An EntityStatValue is instantiated by a higher-level container system, typically an entity's primary stat management component, when the entity is first created or loaded into the world. The primary constructor requires an EntityStatType asset, which provides the baseline configuration. The parameterless constructor exists exclusively for use by the serialization framework (CODEC).
-   **Scope:** The lifetime of an EntityStatValue instance is strictly tied to the lifetime of its parent entity. It persists as long as the entity exists in the server's memory.
-   **Destruction:** The object is marked for garbage collection when its parent entity is unloaded or destroyed. There is no explicit destruction or cleanup method; ownership is managed entirely by the parent component.

## Internal State & Concurrency

-   **State:** This class is highly **mutable**. Its core purpose is to track the changing value of a statistic. It caches the results of modifier calculations in its min and max fields. The internal state is frequently updated by gameplay systems, such as combat, status effects, and regeneration ticks.
-   **Thread Safety:** This class is **not thread-safe**. All methods perform direct, unsynchronized mutations on instance fields. There are no locks, volatile fields, or other concurrency primitives.

    **WARNING:** All interactions with an EntityStatValue instance must be performed on the main server game thread. Accessing or modifying an instance from a worker thread will lead to race conditions, data corruption, and unpredictable behavior.

## API Surface

The public API is minimal, as most complex operations are handled by protected methods invoked by other systems within the same package.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | float | O(1) | Returns the current, clamped value of the statistic. |
| asPercentage() | float | O(1) | Returns the current value as a normalized percentage between min and max. |
| getMin() | float | O(1) | Returns the computed minimum value after all modifiers are applied. |
| getMax() | float | O(1) | Returns the computed maximum value after all modifiers are applied. |
| synchronizeAsset(index, asset) | boolean | O(N) | Re-initializes the stat from its backing asset. Triggers a full re-computation of all modifiers. N is the number of active modifiers. |
| putModifier(key, modifier) | Modifier | O(N) | *Protected.* Adds or replaces a modifier and triggers a full re-computation of the stat's min/max values. |
| removeModifier(key) | Modifier | O(N) | *Protected.* Removes a modifier and triggers a full re-computation of the stat's min/max values. |

## Integration Patterns

### Standard Usage

Direct interaction with EntityStatValue is uncommon for most gameplay code. It is designed to be managed by a "friend" class within the same package, likely an EntityStatsComponent or a similar manager. Gameplay systems interact with the manager, which in turn manipulates the underlying EntityStatValue instances.

```java
// Hypothetical usage from a managing component
// NOTE: putModifier is protected and only accessible from the same package.

EntityStatType healthType = getHealthAsset();
EntityStatValue health = new EntityStatValue(0, healthType);

// Create a modifier that adds 50 to the max health
Modifier buff = new StaticModifier(Modifier.ModifierTarget.MAX, StaticModifier.CalculationType.ADDITIVE, 50.0f);

// The managing component applies the modifier
health.putModifier("strong_buff_id", buff);

// The health.getMax() will now return its base value + 50
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new EntityStatValue()` from general gameplay code. Instances must be created by the entity's stat management system, which provides the required EntityStatType asset for proper initialization.
-   **External State Mutation:** Never modify the internal fields like value, min, or max directly. All changes must go through the provided methods (like the protected set, putModifier, removeModifier) to ensure values are properly clamped and modifiers are re-calculated.
-   **Cross-Thread Access:** Do not share instances of this class across threads. All reads and writes must be marshaled to the main server thread.

## Data Pipeline

EntityStatValue is a key participant in both the gameplay logic loop and the data persistence pipeline.

**Gameplay Modification Flow:**
> Gameplay Event (e.g., Status Effect Applied) -> EntityStatsManager -> `putModifier(...)` -> **EntityStatValue.computeModifiers()** -> Internal State (min, max) Updated

**Serialization Flow (Saving State):**
> Live **EntityStatValue** Instance -> `EntityStatValue.CODEC` -> Serialized Data (Binary/JSON) -> Network Packet or World Save File

**Deserialization Flow (Loading State):**
> Serialized Data -> `EntityStatValue.CODEC` -> **EntityStatValue** Instance (rehydrated) -> Attached to a newly loaded Entity

