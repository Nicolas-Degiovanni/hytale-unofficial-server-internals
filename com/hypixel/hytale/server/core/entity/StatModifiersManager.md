---
description: Architectural reference for StatModifiersManager
---

# StatModifiersManager

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Transient

## Definition
```java
// Signature
public class StatModifiersManager {
```

## Architecture & Concepts
The StatModifiersManager is a server-side stateful calculator responsible for computing and applying all transient and equipment-based statistical modifications to an entity. It acts as the central aggregation point for stats derived from multiple, disparate sources, ensuring that an entity's final stats accurately reflect its current state.

This class decouples the stat calculation logic from the game objects that provide those stats. Assets like Items, Armor, and EntityEffects simply declare their modifiers; the StatModifiersManager is the authoritative system that knows how to discover, interpret, aggregate, and apply them to an entity's core EntityStatMap.

Its primary function is invoked conditionally, gated by a recalculation flag. When triggered, it performs a comprehensive sweep of an entity's state, including:
-   Active status effects from the EffectControllerComponent.
-   Equipped armor from the Inventory.
-   Held items (weapons and utility items) from the Inventory.

The manager correctly handles complex rules, such as applying damage penalties for broken equipment, before committing the final modifiers to the entity's stat map.

## Lifecycle & Ownership
-   **Creation:** The StatModifiersManager is not a persistent component. It is instantiated on-demand by a higher-level entity processing system, likely during the server's main game tick loop when an entity's stats are marked as dirty.
-   **Scope:** The lifecycle of a StatModifiersManager instance is extremely short. It exists only for the duration of a single recalculation operation for a single entity. Its internal state is ephemeral and not intended to persist between ticks or across different entities.
-   **Destruction:** The object is eligible for garbage collection immediately after the `recalculateEntityStatModifiers` method completes. It maintains no persistent external references.

## Internal State & Concurrency
-   **State:** This class is mutable. It maintains two key pieces of state for a single calculation run: an `AtomicBoolean recalculate` flag and an `IntSet statsToClear`. The `recalculate` flag serves as a gate to prevent redundant, computationally expensive calculations. The `statsToClear` set allows for specific stats to be reset to their base value before new modifiers are applied.

-   **Thread Safety:** The class is **not thread-safe** for its primary operations. The core `recalculateEntityStatModifiers` method reads from numerous components and mutates the EntityStatMap directly. It must be executed within a single thread that has exclusive ownership of the entity being processed, typically the main server thread.

    **Warning:** The use of AtomicBoolean for the `recalculate` flag is a specific pattern to allow cross-system signaling. A system on another thread (e.g., a network event handler processing an inventory change) can safely call `setRecalculate(true)`. However, the entity processing system on the main game thread is the only system that should ever read the flag and invoke `recalculateEntityStatModifiers`.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setRecalculate(boolean) | void | O(1) | Atomically sets the flag to signal that a recalculation is required on the next update cycle. |
| queueEntityStatsToClear(int[]) | void | O(N) | Queues an array of entity stat types to be cleared during the next recalculation. N is the array length. |
| recalculateEntityStatModifiers(ref, statMap, accessor) | void | O(Effects + Items) | Executes the full stat recalculation logic if the internal flag is set. This is the primary entry point. |

## Integration Patterns

### Standard Usage
The manager is designed to be used by an entity update system. An external event, such as an inventory change or an effect being applied, signals the need for an update. The main entity loop then consumes this signal and executes the calculation.

```java
// In a system that manages an entity's state
StatModifiersManager statManager = entity.getStatManager(); // Or new StatModifiersManager()

// When an item is equipped or an effect is applied...
statManager.setRecalculate(true);

// Later, during the entity's update tick in the main game loop...
statManager.recalculateEntityStatModifiers(entityRef, entity.getStatMap(), componentAccessor);
```

### Anti-Patterns (Do NOT do this)
-   **Persistent Instances:** Do not retain a single StatModifiersManager instance to service multiple different entities. Its internal state is specific to a single entity's calculation cycle.
-   **Concurrent Execution:** Never call `recalculateEntityStatModifiers` from multiple threads for the same entity. This will cause severe data corruption in the EntityStatMap. The `setRecalculate` method is the only part of the API designed for cross-thread communication.
-   **Manual State Management:** Avoid manually clearing the `recalculate` flag. The `recalculateEntityStatModifiers` method atomically reads and resets the flag itself to ensure exactly-once execution per signal.

## Data Pipeline
The manager orchestrates a multi-stage data flow to compute final stat values. The EntityStatMap is modified in-place.

> Flow:
> 1.  **Signal:** `setRecalculate(true)` is called.
> 2.  **Trigger:** `recalculateEntityStatModifiers` is invoked by the entity update loop.
> 3.  **Pre-computation:**
>     -   The `recalculate` flag is checked and atomically reset.
>     -   Any stats queued via `queueEntityStatsToClear` are minimized in the EntityStatMap.
> 4.  **Aggregation Phase 1 (Effects & Armor):**
>     -   The manager fetches the entity's EffectControllerComponent and Inventory.
>     -   It iterates all active effects, extracts their stat modifiers, and aggregates them into a temporary map.
>     -   It iterates all equipped armor, extracts their stat modifiers, applies broken item penalties, and merges them into the same temporary map.
> 5.  **Application Phase 1:**
>     -   The aggregated modifiers for effects and armor are systematically applied to the EntityStatMap, clearing old modifiers and adding the new, combined values.
> 6.  **Application Phase 2 (Held Items):**
>     -   The manager inspects the held weapon and utility item directly.
>     -   Unlike effects and armor, these modifiers are applied **directly** to the EntityStatMap without an intermediate aggregation map. This represents a distinct data path within the calculation.
> 7.  **Result:** The EntityStatMap passed into the method now contains the fully updated stat modifiers from all relevant sources.

