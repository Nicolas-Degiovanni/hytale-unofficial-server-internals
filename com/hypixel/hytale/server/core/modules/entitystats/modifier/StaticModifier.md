---
description: Architectural reference for StaticModifier
---

# StaticModifier

**Package:** com.hypixel.hytale.server.core.modules.entitystats.modifier
**Type:** Transient Data Model

## Definition
```java
// Signature
public class StaticModifier extends Modifier {
```

## Architecture & Concepts
The StaticModifier is a fundamental data model within the entity statistics system. It represents a single, fixed, and unchanging mathematical operation applied to a base statistic value. Its design is intentionally simple, serving as a lightweight data container that describes *how* a stat should be altered, but without containing any logic for when or why the alteration occurs.

This class acts as a bridge between configured game data and the live, in-memory stat calculation engine. Through the Hytale `Codec` system, StaticModifier instances are deserialized from asset files (e.g., item definitions, ability configurations) and network packets. This allows game designers to define stat changes in data files without requiring code changes.

There are two primary calculation types:
*   **ADDITIVE:** The modifier's amount is added to the stat's value. Used for flat bonuses like "+10 Health".
*   **MULTIPLICATIVE:** The stat's value is multiplied by the modifier's amount. Used for percentage-based bonuses like "+10% Damage".

The presence of two distinct codecs, CODEC and ENTITY_CODEC, indicates a separation between the canonical data definition and the network-serialized representation, which includes versioning for protocol compatibility.

## Lifecycle & Ownership
A StaticModifier is a short-lived, passive object whose lifecycle is strictly controlled by its owner.

*   **Creation:** Instances are created in two primary scenarios:
    1.  **Deserialization:** The most common path. The Hytale `Codec` engine instantiates StaticModifier objects when loading game assets (items, armor, effects) from configuration files or when decoding network packets that describe entity state.
    2.  **Programmatic Instantiation:** Game logic can create instances directly via the constructor, typically to apply a temporary, dynamically-calculated stat change to an entity.

*   **Scope:** The object's lifetime is bound to its containing component. If a StaticModifier is part of an item's definition, it exists as long as the item definition is loaded in memory. If it is part of a temporary buff applied to an entity, it is discarded when the buff expires. It does not persist globally or across sessions.

*   **Destruction:** The object is managed by the Java garbage collector. There are no native resources or explicit cleanup methods. It is eligible for garbage collection as soon as its owner (the buff, item, or other component) is dereferenced.

## Internal State & Concurrency
*   **State:** The state of a StaticModifier, consisting of its `calculationType` and `amount`, is designed to be effectively immutable after its initial creation. While the fields are not declared final to support the `Codec` builder pattern, they are not intended to be changed after the object is fully constructed and integrated into the stat system.

*   **Thread Safety:** This class is **not thread-safe**. It is a plain data object with no internal locking or synchronization mechanisms.

    **WARNING:** All operations on a StaticModifier instance must be confined to a single thread, typically the main server game loop or "tick" thread. Do not share and mutate instances across threads. It is safe to pass fully-constructed instances between threads as read-only data.

## API Surface
The public contract is minimal, focusing on data retrieval and the core application logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(float statValue) | float | O(1) | Applies the modifier's calculation to the provided base value and returns the result. |
| getCalculationType() | CalculationType | O(1) | Returns the enum defining the mathematical operation (ADDITIVE or MULTIPLICATIVE). |
| getAmount() | float | O(1) | Returns the value used in the calculation. |
| toPacket() | com.hypixel.hytale.protocol.Modifier | O(1) | Converts the object into a network-transferable protocol buffer representation. |

## Integration Patterns

### Standard Usage
A StaticModifier is almost never used in isolation. It is typically held within a list by a higher-level component (like a Buff or an EquipmentManager). A central stat calculation service then retrieves all relevant modifiers for a given stat and applies them in a deterministic order.

```java
// Conceptual example of a stat calculation service
// NOTE: EntityStatsComponent and StatType are hypothetical classes for context.

EntityStatsComponent stats = entity.getStatsComponent();
float baseHealth = stats.getBaseValue(StatType.HEALTH);

// Retrieve all modifiers from buffs, equipment, etc.
List<StaticModifier> modifiers = entity.getActiveModifiers(StatType.HEALTH);

// Apply additive modifiers first
float healthAfterAdditives = baseHealth;
for (StaticModifier mod : modifiers) {
    if (mod.getCalculationType() == CalculationType.ADDITIVE) {
        healthAfterAdditives = mod.apply(healthAfterAdditives);
    }
}

// Then apply multiplicative modifiers
float finalHealth = healthAfterAdditives;
for (StaticModifier mod : modifiers) {
    if (mod.getCalculationType() == CalculationType.MULTIPLICATIVE) {
        finalHealth = mod.apply(finalHealth);
    }
}
```

### Anti-Patterns (Do NOT do this)
*   **State Mutation:** Do not modify a StaticModifier's state after it has been added to a live game system. This can lead to unpredictable stat calculations and desynchronization between client and server. If a change is needed, create a new StaticModifier instance.
*   **Shared Instances:** Avoid using a single StaticModifier instance for multiple distinct effects. For example, if two different buffs both grant "+10 Health", each buff should own its own StaticModifier instance. This prevents accidental side-effects if one buff is removed or changed.

## Data Pipeline
The StaticModifier is a critical link in the chain that translates static game data into dynamic gameplay calculations.

> **Flow 1: Asset Loading**
> Game Asset File (JSON/Binary) -> Hytale Codec Engine -> **StaticModifier** instance -> Stored in an Item or Ability definition

> **Flow 2: Stat Calculation**
> Stat Calculation Request -> Retrieve Base Stat -> Aggregate all **StaticModifier** instances from equipment/buffs -> Sequentially call `apply()` -> Final Calculated Stat -> Used in Game Logic (e.g., Damage Calculation)

