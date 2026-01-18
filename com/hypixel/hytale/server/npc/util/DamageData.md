---
description: Architectural reference for DamageData
---

# DamageData

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Transient State Object

## Definition
```java
// Signature
public class DamageData {
```

## Architecture & Concepts
The DamageData class is a stateful data container designed to aggregate and summarize combat statistics for a single entity, typically a server-side NPC. It functions as a short-term "combat memory," enabling AI systems to make informed decisions based on recent events.

This class is not a standalone service but rather a component-level data structure. It encapsulates the logic for tracking damage dealt and received, kills, and threat assessment. Its primary purpose is to answer questions for an AI behavior, such as:
*   Who is the most significant threat?
*   Who have I dealt the most damage to?
*   What types of damage am I receiving?
*   Have I killed a specific target?

A critical architectural aspect is its integration with the server's Entity Component System (ECS). The method onSufferedDamage requires a CommandBuffer to query components of the damage source. This allows for complex game logic, such as verifying if a player-inflicted damage event should be ignored (e.g., from a player in Creative mode), preventing NPCs from reacting to non-threatening interactions.

### Lifecycle & Ownership
- **Creation:** DamageData is a plain Java object, instantiated directly via its constructor. It is typically created and owned by a higher-level AI manager or a specific combat-related component attached to an NPC. An instance is often created when an entity enters a combat state.
- **Scope:** The object's lifetime is intentionally short and context-dependent. It is meant to track a single encounter or a specific behavioral phase. The presence of the public reset method indicates that instances are designed to be reused across multiple encounters for the same NPC, avoiding repeated object allocation.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for collection when its owning component is destroyed or when it is explicitly dereferenced, for example, at the end of a combat sequence.

## Internal State & Concurrency
- **State:** DamageData is highly mutable. Its core consists of several maps that cache combat events. Every call to a mutator method like onInflictedDamage or onSufferedDamage alters this internal state. The object's value is derived entirely from the sequence of method calls made upon it since the last reset.
- **Thread Safety:** **This class is not thread-safe.** The internal collections (HashMap, Object2DoubleOpenHashMap) are not synchronized. All method calls that read or write state must be performed on the same thread, which is expected to be the main server thread responsible for the entity's game logic updates.

**WARNING:** Concurrent access from multiple threads will lead to race conditions, data corruption, and unpredictable server behavior. Do not share instances of this class across different world-update threads without external locking, which is strongly discouraged.

## API Surface
The public API is divided into mutators for recording events and accessors for querying the aggregated state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| reset() | void | O(N) | Clears all tracked statistics. N is the number of entries in the internal maps. |
| onInflictedDamage(target, amount) | void | O(1) | Records damage dealt to a target entity. Updates the most damaged victim. |
| onSufferedDamage(buffer, damage) | void | O(1) | Records damage received. Contains logic to ignore creative players. |
| onKill(victim, position) | void | O(1) | Records that the owning entity has killed a victim at a specific position. |
| getMostDamagingAttacker() | Ref | O(1) | Returns a reference to the entity that has dealt the most damage. Returns null if none. |
| getMostDamagedVictim() | Ref | O(1) | Returns a reference to the entity that has received the most damage. Returns null if none. |
| getAnyAttacker() | Ref | O(N) | Returns any valid attacker from the list. N is the number of unique attackers. |
| clone() | DamageData | O(N) | Creates a deep copy of the current combat state. |

## Integration Patterns

### Standard Usage
DamageData is typically held as a field within an NPC's combat management component. This component subscribes to game events and delegates them to the DamageData instance. The AI system then queries this instance during its update tick to make tactical decisions.

```java
// Within an AI or Entity Component update method
// Assume 'combatState' is a field holding a DamageData instance

// 1. When a damage event is received for our NPC
public void handleDamageEvent(Damage damageEvent, CommandBuffer buffer) {
    this.combatState.onSufferedDamage(buffer, damageEvent);
}

// 2. During the AI's decision-making tick
public void updateAI() {
    Ref<EntityStore> primaryThreat = this.combatState.getMostDamagingAttacker();

    if (primaryThreat != null) {
        // Set the NPC's target to the primary threat
        this.setTarget(primaryThreat);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Leakage:** Failing to call reset() between distinct combat encounters. This will cause the NPC to carry threat data from previous fights, leading to incorrect and unpredictable targeting behavior.
- **Cross-Thread Access:** Reading or writing to a DamageData instance from any thread other than the entity's primary update thread. This will cause severe concurrency issues.
- **Shared Instances:** Using a single DamageData instance for multiple NPCs. Each NPC requires its own unique combat memory.

## Data Pipeline
DamageData acts as a stateful aggregator within the server's event processing flow. It does not generate data but rather consumes and transforms it into a queryable summary for the AI.

> Flow:
> Player Action -> Network Packet -> Server Damage System -> **Damage Event** -> NPC Combat Component -> **DamageData.onSufferedDamage()** -> AI Behavior Tree -> **DamageData.getMostDamagingAttacker()** -> AI Action (e.g., Change Target)

