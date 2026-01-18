---
description: Architectural reference for DamageCalculatorSystems
---

# DamageCalculatorSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.damage
**Type:** Utility

## Definition
```java
// Signature
public class DamageCalculatorSystems {
```

## Architecture & Concepts

The DamageCalculatorSystems class is a container for logic and systems related to advanced, stateful damage calculations within the server-side Entity Component System (ECS). It is not a single system itself, but rather a collection of related components that work together to implement complex combat mechanics, such as damage scaling for sequential hits (combos).

Its primary architectural role is to decouple the *initiation* of a damage event from the *modification* of that event based on ongoing combat state. This is achieved through a metadata-driven pattern:

1.  **Damage Event Creation:** An initial system (e.g., handling a weapon swing) creates a standard Damage event.
2.  **Metadata Attachment:** A stateful object, **DamageSequence**, is attached to the Damage event using a MetaKey. This object tracks the number of consecutive hits.
3.  **System-driven Modification:** A dedicated ECS system, **SequenceModifier**, queries for Damage events that have this specific metadata attached. It then reads the state from DamageSequence, applies a damage multiplier, and updates the state for the next potential hit.

This design allows the core damage application pipeline to remain generic, while specialized systems like SequenceModifier can opt-in to modify events based on specific, attached data.

### Lifecycle & Ownership

The top-level DamageCalculatorSystems class is a stateless utility and is not instantiated. The lifecycle described here pertains to its primary active component, the **SequenceModifier** system.

-   **Creation:** The SequenceModifier system is instantiated once by the DamageModule during server bootstrap. It is registered with the world's ECS scheduler.
-   **Scope:** The system instance is a singleton that persists for the entire server session. It is stateless; all operational data is passed into its *handle* method during event processing.
-   **Destruction:** The system is deregistered and garbage collected when the server shuts down and the ECS world is destroyed.

## Internal State & Concurrency

-   **State:** The DamageCalculatorSystems class itself is stateless. The nested SequenceModifier system is also stateless. All state it operates on is contained within the transient Damage event object and its attached DamageSequence metadata. The DamageSequence object is **mutable**, as its internal hit counter is incremented by the SequenceModifier system.

-   **Thread Safety:** This class and its nested systems are **not thread-safe** and must only be accessed from the main server thread that processes the world tick. The ECS relies on a strict, single-threaded execution model for its systems. The SequenceModifier system uses a CommandBuffer to safely queue component modifications (like applying stat changes to an attacker), which are resolved by the engine at a later, synchronized point in the tick. Direct, multi-threaded invocation will lead to race conditions and world corruption.

## API Surface

The primary public contract is the static factory method for creating damage events intended for this pipeline.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| queueDamageCalculator(...) | Damage[] | O(N) | Factory method to create Damage events from a map of causes. N is the number of DamageCause entries. Applies initial modifiers, such as penalties for broken weapons. |
| SequenceModifier.handle(...) | void | O(S) | **Engine-invoked only.** Modifies a Damage event based on its attached DamageSequence metadata. S is the number of EntityStatOnHit effects to process. |

## Integration Patterns

### Standard Usage

The correct pattern is to use DamageCalculatorSystems to create a Damage event and then attach a DamageSequence object to it. This "tags" the event for processing by the SequenceModifier system later in the damage pipeline.

```java
// In a system that handles weapon attacks...

// 1. Define the base damage
Object2FloatMap<DamageCause> damageMap = new Object2FloatOpenHashMap<>();
damageMap.put(DamageCause.PHYSICAL, 10.0f);

// 2. Create the Damage events using the factory
Damage[] damageEvents = DamageCalculatorSystems.queueDamageCalculator(
    world,
    damageMap,
    targetEntityRef,
    commandBuffer,
    damageSource,
    weaponItemStack
);

// 3. Retrieve or create the attacker's combo sequence state
// This state is typically stored on the attacking entity itself.
DamageCalculatorSystems.Sequence sequenceState = getOrCreateSequenceState(attackerRef);
DamageCalculator damageCalculator = getWeaponDamageCalculator(); // From weapon config

// 4. Create the metadata and attach it to EACH damage event
DamageCalculatorSystems.DamageSequence sequenceMeta = new DamageCalculatorSystems.DamageSequence(sequenceState, damageCalculator);

for (Damage damage : damageEvents) {
    damage.getMetaStore().putMetaObject(DamageCalculatorSystems.DAMAGE_SEQUENCE, sequenceMeta);
    // The event is now ready to be fired into the event bus
    world.getEventBus().fire(damage);
}
```

### Anti-Patterns (Do NOT do this)

-   **Directly Calling handle:** Never call `sequenceModifier.handle(...)` directly. This method is designed to be invoked exclusively by the ECS scheduler, which respects the critical dependency ordering. Calling it manually will bypass other essential damage systems and cause unpredictable calculations.

-   **Forgetting Metadata:** Firing a Damage event without attaching the DamageSequence metadata via the DAMAGE_SEQUENCE MetaKey will cause the SequenceModifier system to ignore it. No combo damage scaling or on-hit stat effects will be applied.

-   **State Mismanagement:** The DamageSequence object contains the hit counter. If this object is shared incorrectly between different attack sequences or not reset when a combo breaks, the damage calculations will be incorrect. The state must be managed carefully on the attacking entity.

## Data Pipeline

The flow of data for a damage calculation involving this system is ordered and precise, managed by the ECS dependency graph.

> Flow:
> Player Attack Action -> Weapon Handling System -> **queueDamageCalculator()** -> Damage Event Created -> **DamageSequence Metadata Attached** -> Event Fired to Bus -> **SequenceModifier.handle()** (reads metadata, modifies damage amount, increments hit count) -> Modified Damage Event -> DamageSystems.ApplyDamage (applies final damage to health) -> Entity Health Updated

