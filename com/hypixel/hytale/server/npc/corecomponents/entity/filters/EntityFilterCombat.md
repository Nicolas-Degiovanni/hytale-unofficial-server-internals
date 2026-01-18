---
description: Architectural reference for EntityFilterCombat
---

# EntityFilterCombat

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient

## Definition
```java
// Signature
public class EntityFilterCombat extends EntityFilterBase {
```

## Architecture & Concepts
The EntityFilterCombat class is a specialized predicate within the server-side NPC AI framework. Its sole purpose is to evaluate whether a target entity is currently in a specific combat state, such as attacking, blocking, or performing a named attack sequence.

This component operates as a data-driven condition, typically used within higher-level AI constructs like Behavior Trees or Target Selectors. It is not intended for direct manual instantiation; instead, its properties are defined in NPC asset files and it is constructed by the asset loading pipeline via its corresponding builder, BuilderEntityFilterCombat.

Architecturally, it decouples AI decision-making logic from the low-level mechanics of the combat system. It achieves this by querying a dedicated facade, CombatViewSystems, which provides a clean, interpreted view of an entity's active combat data. This allows AI designers to create complex, state-aware behaviors (e.g., "Only use this ability if the player is charging a heavy attack") without needing to understand the underlying combat state machine implementation.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by BuilderEntityFilterCombat during the server's NPC asset loading phase. An instance is created for each filter definition found in an NPC's configuration files.
- **Scope:** The object is stateless and immutable. Its lifetime is tied to the parent NPC definition asset. It persists as long as the NPC type is loaded on the server and is reused for all evaluations across all instances of that NPC.
- **Destruction:** Marked for garbage collection when the server unloads the associated NPC asset bundle, typically during a server shutdown or a dynamic asset reload.

## Internal State & Concurrency
- **State:** **Immutable**. All configuration fields, such as sequence, combatMode, and time ranges, are declared final and are set only once within the constructor. The class holds no mutable runtime state.
- **Thread Safety:** **Fully thread-safe**. Its immutable nature guarantees that a single instance can be safely accessed and used by multiple NPC agents operating on different server threads without any risk of race conditions or data corruption. No synchronization or locking is required.

## API Surface
The public contract is minimal, focusing entirely on evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(N) | Evaluates the target entity against the filter's configured combat state. N is the number of active combat actions on the target. Returns true if a match is found. |
| cost() | int | O(1) | Returns a static integer cost (100). This value is used by AI scheduling systems to prioritize the order of filter evaluations, with higher-cost filters often being checked last. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly by developers. It is configured within an NPC asset file and executed automatically by the AI engine's targeting or behavior systems. The following pseudocode illustrates how the system uses an instance of this filter.

```java
// PSEUDOCODE: How the AI system uses the filter
// An EntityFilterCombat instance is retrieved from the parsed NPC definition.
EntityFilterBase combatFilter = npcDefinition.getBehaviorNode("CounterAttack").getCondition();

// The system evaluates the filter against a potential target.
Entity player = findNearestPlayer();
boolean canCounter = combatFilter.matchesEntity(
    self.getRef(),
    player.getRef(),
    self.getRole(),
    world.getEntityStore()
);

if (canCounter) {
    // The player is in the specified combat state, so trigger the counter-attack behavior.
    npc.getBehaviorTree().setActiveNode("CounterAttack");
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new EntityFilterCombat()`. The class is designed to be configured and created via the data-driven asset pipeline. Manual creation bypasses this design, leading to unmanageable and non-standard NPC behavior.
- **Stateful Modification:** Do not attempt to subclass or modify this filter to hold runtime state. Its immutability and statelessness are critical for ensuring thread safety and predictable behavior across the entire AI system.

## Data Pipeline
The flow of data for a single evaluation is linear and synchronous. The filter acts as a transformation stage, converting raw combat state into a simple boolean decision.

> Flow:
> AI Behavior Tick → Target Entity Reference → **EntityFilterCombat** → Query to CombatViewSystems → InterpretedCombatData List → Internal State Matching → Boolean Result → AI State Transition

