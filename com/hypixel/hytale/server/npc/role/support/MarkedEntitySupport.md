---
description: Architectural reference for MarkedEntitySupport
---

# MarkedEntitySupport

**Package:** com.hypixel.hytale.server.npc.role.support
**Type:** Transient

## Definition
```java
// Signature
public class MarkedEntitySupport {
```

## Architecture & Concepts

The MarkedEntitySupport class is a stateful component owned by a single NPCEntity. It functions as a dedicated memory and targeting system, allowing an NPC to maintain references to other entities within the game world. This system is fundamental to complex AI behaviors such as combat, escorting, or interacting with specific world objects.

Conceptually, it provides a set of named "slots" (e.g., LockedTarget, FollowTarget, InteractionPoint). AI behaviors can then set or retrieve entity references from these slots by name, decoupling the high-level logic from the low-level details of reference management.

The system is built upon the engine's safe reference type, Ref<EntityStore>, which prevents issues related to dangling pointers. If a targeted entity is unloaded or destroyed, its corresponding Ref becomes invalid, and this class correctly handles these cases by treating the reference as null.

A key architectural feature is its integration with the flocking system via the flockSetTarget method. This allows a single NPC, often a flock leader, to designate a target and have that designation broadcast to all other members of its EntityGroup. This facilitates coordinated group AI without requiring each member to perform its own target acquisition logic.

### Lifecycle & Ownership
- **Creation:** An instance of MarkedEntitySupport is created by its parent NPCEntity during the entity's own initialization phase. It is constructed with a reference to its parent but is not fully operational at this stage.
- **Scope:** The lifecycle of a MarkedEntitySupport object is strictly bound to its parent NPCEntity. It persists as long as the parent NPC is loaded in the world.
- **Destruction:** The unloaded method is invoked by the parent NPCEntity when it is being removed from the world. This method is responsible for clearing all stored Ref<EntityStore> instances, breaking reference cycles and ensuring clean garbage collection.

## Internal State & Concurrency
- **State:** This class is highly mutable. Its primary state is contained within the entityTargets array, which holds the references to marked entities. This array is dynamically configured during initialization and its contents are frequently modified at runtime by AI behaviors.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be accessed exclusively from the main server thread, which manages the game state for its parent NPCEntity. All internal collections are non-concurrent. Any attempt to modify or read its state from an asynchronous task or a different thread will lead to critical race conditions, data corruption, or server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| postRoleBuilder(support) | void | O(N) | Initializes the component from a builder. Configures target slots and allocates internal arrays. **CRITICAL:** Must be called before any other method. |
| setMarkedEntity(slot, target) | void | O(1) | Sets or clears the entity reference for a given slot index or name. Automatically handles invalid or null targets by clearing the slot. |
| getMarkedEntityRef(slot) | Ref<EntityStore> | O(1) | Retrieves a valid entity reference from a slot. Returns null if the slot is empty or the stored reference has become invalid. |
| flockSetTarget(slot, target, store) | void | O(M) | Propagates a target to all members of the parent's flock. M is the number of members in the flock. |
| getTargetReferenceToIgnoreForAvoidance() | Ref<EntityStore> | O(1) | Retrieves a specific target reference intended for consumption by pathfinding or avoidance systems. |
| unloaded() | void | O(N) | Lifecycle method for cleanup. Clears all stored entity references. N is the number of target slots. |

## Integration Patterns

### Standard Usage
This component is retrieved from an NPCEntity instance and used by AI behaviors to manage targets. The primary interaction involves setting a target in a predefined slot and later retrieving it.

```java
// Example from within an NPC AI behavior
NPCEntity self = getParentEntity();
MarkedEntitySupport support = self.getMarkedEntitySupport();

// Find a new enemy and store it in the default target slot
Ref<EntityStore> newEnemy = findTarget();
support.setMarkedEntity(MarkedEntitySupport.DEFAULT_TARGET_SLOT, newEnemy);

// Propagate this target to the entire flock
support.flockSetTarget(MarkedEntitySupport.DEFAULT_TARGET_SLOT, newEnemy, self.getStore());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually create an instance with `new MarkedEntitySupport()`. This component is managed internally by the NPCEntity and must be retrieved from it.
- **Skipping Initialization:** Calling any method like setMarkedEntity before postRoleBuilder has been invoked by the engine will result in a NullPointerException, as the internal arrays will not have been allocated.
- **Cross-Thread Access:** Never access an instance of MarkedEntitySupport from a separate thread. All interactions must be synchronized with the main server game loop.

## Data Pipeline

MarkedEntitySupport acts as a state repository rather than a data processing pipeline. Data flows into it from AI systems and flows out to other systems that require target information, such as pathfinding or combat logic.

> **Flow 1: Target Acquisition**
> AI Behavior Logic -> `findTarget()` -> `setMarkedEntity()` -> **MarkedEntitySupport** (State Updated)

> **Flow 2: Target Propagation (Flocking)**
> Flock Leader AI -> `flockSetTarget()` -> **MarkedEntitySupport** -> EntityGroup -> Other Flock Members

> **Flow 3: System Integration (Avoidance)**
> Pathfinding System -> `getTargetReferenceToIgnoreForAvoidance()` -> **MarkedEntitySupport** -> Returns Ignored Target Ref

