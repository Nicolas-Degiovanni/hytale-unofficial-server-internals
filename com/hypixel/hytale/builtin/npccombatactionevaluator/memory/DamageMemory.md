---
description: Architectural reference for DamageMemory
---

# DamageMemory

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.memory
**Type:** Transient

## Definition
```java
// Signature
public class DamageMemory extends Object implements Component<EntityStore> {
```

## Architecture & Concepts
DamageMemory is a data component within Hytale's Entity Component System (ECS). Its sole purpose is to attach state to an entity, specifically an NPC, tracking damage received during a combat encounter. It does not contain any logic itself; it is a passive data container manipulated and read by various game systems.

This component is a foundational element for the AI's **NPCCombatActionEvaluator** system. By storing both short-term (recent) and long-term (total) damage values, it enables AI systems to make more sophisticated decisions. For example, an NPC might trigger a defensive ability if its *recentDamage* spikes, or attempt to flee if its *totalCombatDamage* exceeds a critical threshold.

The separation of this state into a component is a classic ECS pattern, decoupling the data (what damage was taken) from the systems that act upon it (the combat system that applies damage and the AI system that reacts to it).

### Lifecycle & Ownership
- **Creation:** DamageMemory components are not instantiated directly. They are added to an Entity by a managing system, typically a combat or AI initialization system, when an NPC first enters a combat state.
- **Scope:** The component's lifetime is strictly bound to the Entity it is attached to. It persists as long as the NPC is considered to be in an active or recent combat encounter.
- **Destruction:** The component is destroyed automatically when its parent Entity is removed from the world. It may also be explicitly removed by a system if an NPC successfully disengages from combat and its memory is cleared.

## Internal State & Concurrency
- **State:** The internal state is mutable and designed for high-frequency updates during combat. It maintains two distinct floating-point counters:
    - **recentDamage:** A short-term accumulator, intended to be cleared periodically by a managing system to track bursts of incoming damage.
    - **totalCombatDamage:** A long-term accumulator that tracks all damage taken since the beginning of the current combat encounter.
- **Thread Safety:** **This component is not thread-safe.** As with most ECS components, all mutations and reads must be performed on the main world update thread or within a job that has exclusive write access to this component type for the given entity. Unsynchronized access from multiple threads will lead to race conditions and non-deterministic AI behavior.

## API Surface
The public API is minimal, focusing on state mutation and retrieval. Simple getters are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier used to query for this component from the EntityStore. |
| addDamage(float) | void | O(1) | Adds the specified damage amount to both the recent and total damage counters. |
| clearRecentDamage() | void | O(1) | Resets the recentDamage counter to zero. This is typically called by a system on a timer. |
| clearTotalDamage() | void | O(1) | Resets both total and recent damage counters. Used when an NPC exits combat. |

## Integration Patterns

### Standard Usage
A system responsible for processing combat events retrieves the component from the target entity and updates its state. Another system, the AI evaluator, reads this state to inform its decisions.

```java
// In a system that processes damage events...
Entity targetEntity = event.getTarget();
DamageMemory memory = targetEntity.getComponent(DamageMemory.getComponentType());

if (memory != null) {
    memory.addDamage(event.getDamageAmount());
}

// In an AI evaluation system...
float totalDamage = memory.getTotalCombatDamage();
if (totalDamage > LOW_HEALTH_THRESHOLD) {
    // Trigger a defensive or retreat behavior
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new DamageMemory()`. A component instance created this way is untracked by the ECS and will not be visible to any game systems. Components must be added to entities via the EntityStore or an equivalent entity manager.
- **State Management:** Do not use this component to store permanent character statistics. It is designed exclusively for tracking transient combat state. Its data is expected to be cleared when combat ends.

## Data Pipeline
DamageMemory acts as a stateful intermediary between the cause of damage and the AI's reaction to it.

> Flow:
> Damage Event -> Combat Resolution System -> **DamageMemory** (Write State) -> AI Action Evaluator System (Read State) -> AI Behavior Tree (Decision)

