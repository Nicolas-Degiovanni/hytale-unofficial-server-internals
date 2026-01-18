---
description: Architectural reference for DamageCalculator
---

# DamageCalculator

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server.combat
**Type:** Transient Data Object

## Definition
```java
// Signature
public class DamageCalculator {
```

## Architecture & Concepts
The DamageCalculator is a data-driven configuration object that defines the rules for calculating damage for a specific game interaction, such as a weapon attack or spell effect. It is not a service or manager, but rather a blueprint for damage computation that is deserialized from game asset files. This design allows game designers and modders to define complex combat mechanics without modifying core engine code.

Its primary architectural function is to decouple the *definition* of damage from the *application* of damage. A higher-level system, such as the combat module, uses an instance of DamageCalculator to produce a set of damage values, which are then applied to a target entity.

A key internal concept is the transformation of damage cause identifiers. During asset loading, a human-readable map of damage causes (String to float) is converted into a performance-optimized integer-keyed map. This is handled by the static CODEC's *afterDecode* hook, which uses the global DamageCause asset map to resolve string names to integer indices. This avoids costly string operations during the critical path of combat calculations in the main game loop.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale codec system during the server's asset loading phase. The public static final field CODEC defines the deserialization logic from a data file (e.g., JSON) into a Java object. Manual instantiation is strongly discouraged.

- **Scope:** The lifetime of a DamageCalculator instance is bound to its owning asset. For example, if it defines the damage for a specific sword, it will be loaded into memory along with the sword's definition and will persist as long as that definition is required by the server.

- **Destruction:** The object is eligible for garbage collection when its parent asset is unloaded from memory. It manages no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The object is stateful but designed to be **immutable after creation**. All its configuration fields are populated once during deserialization by the codec. The primary method, calculateDamage, only reads this state; it does not modify it. The field baseDamage is marked *transient* and is populated programmatically in the *afterDecode* hook, serving as a performance cache.

- **Thread Safety:** The class is **conditionally thread-safe**. After its state is fully initialized by the codec system during the single-threaded asset loading phase, its public methods can be safely called from multiple threads concurrently. The calculateDamage method is free of side effects and does not modify internal state, making it safe for use in a multi-threaded game loop or job system.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| calculateDamage(double durationSeconds) | Object2FloatMap<DamageCause> | O(N) | Computes the final damage values based on the object's configuration. N is the number of base damage entries. Returns null if no base damage is defined. |

## Integration Patterns

### Standard Usage
A DamageCalculator is typically embedded within a larger configuration asset, such as an item or ability definition. Game logic retrieves the calculator from the asset and invokes it when the corresponding game event occurs.

```java
// Conceptual example: A system processes a weapon hit
ItemDefinition sword = itemRegistry.get("diamond_sword");
DamageCalculator calculator = sword.getDamageCalculator();

// Calculate damage for a 0.5 second swing duration
Object2FloatMap<DamageCause> damageMap = calculator.calculateDamage(0.5);

// Apply the calculated damage to the target
if (damageMap != null) {
    combatSystem.applyDamage(targetEntity, damageMap);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DamageCalculator()`. This creates an empty, unconfigured object that will not function correctly. Instances must be created via the codec system from asset files.

- **State Mutation:** Do not attempt to modify the fields of a DamageCalculator after it has been initialized. The system relies on the assumption that these objects are immutable once loaded. Modifying them at runtime can lead to unpredictable behavior and race conditions.

## Data Pipeline
The DamageCalculator acts as a transformation step in the server's combat data flow. It converts static configuration data into dynamic, calculated damage values.

> Flow:
> Game Asset File (JSON) -> Hytale Codec System -> **DamageCalculator Instance** -> Combat System Logic -> `calculateDamage()` -> Damage Event -> Entity Health Update

