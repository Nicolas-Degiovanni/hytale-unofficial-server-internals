---
description: Architectural reference for RegeneratingModifier
---

# RegeneratingModifier

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset.modifier
**Type:** Transient / Data Object

## Definition
```java
// Signature
public class RegeneratingModifier {
```

## Architecture & Concepts
The RegeneratingModifier is a data-driven configuration object, not a service. It represents a single, conditional rule within the server-side entity statistics module. Its core purpose is to encapsulate a numeric value, the *amount*, that is applied only when a specific set of *conditions* are met for a given entity at a point in time.

This class is a fundamental building block for creating dynamic and context-aware regeneration effects (e.g., health, mana, energy). It is designed to be defined in external asset files, typically JSON, and deserialized into a live object by the engine's asset loading pipeline. The static CODEC field is the primary mechanism that enables this data-driven behavior, allowing designers to define complex game logic without modifying engine code.

It does not manage state or timers itself; rather, it acts as a stateless calculator that is invoked by a higher-level system, such as a per-entity statistics processor, which is responsible for ticking game logic and applying the results.

### Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale Codec system during the asset loading phase. The engine reads a data file (e.g., an entity definition file) and uses the static CODEC to deserialize the configuration into a RegeneratingModifier object. Manual instantiation via the public constructor should be reserved for unit testing or highly specific procedural generation scenarios.
- **Scope:** The lifetime of a RegeneratingModifier instance is bound to its containing asset. For example, if it is part of an item's definition, it will exist as long as that item definition is loaded in memory. It is a lightweight, short-lived object from the perspective of the entire server session.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup once the parent asset that owns it is unloaded or no longer referenced. There are no native resources or explicit destruction protocols associated with this class.

## Internal State & Concurrency
- **State:** The internal state, consisting of the conditions array and the amount, is established at creation and is not intended to be modified thereafter. While the fields are not marked as final, the public API provides no methods for mutation, making the class behave as an **Immutable** object.
- **Thread Safety:** This class is **Conditionally Thread-Safe**. The `getModifier` method is a pure function relative to the object's own state. Multiple threads can safely call `getModifier` on a shared instance without causing data corruption. Thread safety is therefore dependent on the safety of the arguments passed into the method, specifically the ComponentAccessor and Ref, which are managed by the calling system.

## API Surface
The public contract is minimal, focusing entirely on the evaluation of the modifier.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getModifier(store, ref, currentTime) | float | O(N) | Evaluates the configured conditions against the target entity. N is the number of conditions. Returns the modifier's amount if all conditions are met; otherwise, returns a neutral value of 1.0F. |

## Integration Patterns

### Standard Usage
A RegeneratingModifier is not retrieved from a service registry. Instead, it is accessed as part of a larger configuration object, such as an entity asset. A system responsible for calculating entity stats will iterate over a collection of these modifiers to determine the final regeneration rate.

```java
// Hypothetical usage within an EntityStatsProcessor
EntityStatsAsset statsAsset = entity.getStatsAsset();
float totalRegenModifier = 1.0F;

// The processor iterates over all configured modifiers
for (RegeneratingModifier modifier : statsAsset.getRegenModifiers()) {
    // getModifier is called to evaluate the rule
    totalRegenModifier *= modifier.getModifier(entityStore, entityRef, Instant.now());
}

// The final calculated value is then applied
entity.applyHealthRegen(baseRegen * totalRegenModifier);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not use reflection or other means to modify the internal `conditions` or `amount` fields after an instance has been created. The system relies on the implicit immutability of this object for predictable behavior.
- **Misinterpreting the Default:** The return value of 1.0F for unmet conditions is a multiplicative identity. Systems using this class should be designed to multiply this result, not add it. Treating the 1.0F as an additive value is a common logical error.

## Data Pipeline
The primary flow for this class is from a static data definition file into the live game engine where it is used for calculations.

> Flow:
> Asset File (JSON) -> Hytale Codec Deserializer -> **RegeneratingModifier Instance** (In Memory) -> Entity Stats Calculation Engine -> Final Regeneration Value

