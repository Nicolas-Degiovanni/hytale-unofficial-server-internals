---
description: Architectural reference for KillTask
---

# KillTask

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.task
**Type:** Contract Interface

## Definition
```java
// Signature
public interface KillTask {
   void checkKilledEntity(Store<EntityStore> var1, Ref<EntityStore> var2, Objective var3, NPCEntity var4, Damage var5);
}
```

## Architecture & Concepts
The KillTask interface defines a behavioral contract for evaluating entity death events within the context of a game objective. It is a core component of the server-side Adventure Mode objective system, acting as a specialized listener that decouples the central objective tracking logic from the specific conditions of a "kill quest".

This interface is a classic example of the **Strategy Pattern**. Instead of the main Objective class containing a complex set of conditional checks for all possible kill scenarios (e.g., kill a specific mob type, use a certain weapon, be in a specific zone), the logic is encapsulated within different implementations of KillTask. The active Objective simply holds a reference to a KillTask implementation and delegates the evaluation of a death event to it.

Its primary function is to answer the question: "Does this specific death event satisfy the criteria for advancing this objective?"

### Lifecycle & Ownership
- **Creation:** Implementations of KillTask are not instantiated directly. They are created and configured by the Objective system when a quest or objective is loaded from its definition files. The specific implementation to use is typically determined by data, not hard-coded logic.
- **Scope:** The lifetime of a KillTask instance is tightly bound to the lifetime of the Objective that owns it. It persists as long as the objective is active for a player or group.
- **Destruction:** When an Objective is completed, failed, or otherwise removed from a player's state, all associated KillTask instances are dereferenced and become eligible for garbage collection. There is no manual destruction method.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, concrete implementations are almost always **mutable and stateful**. They are responsible for maintaining progress, such as the number of entities killed, and persisting this state within the parent Objective.
- **Thread Safety:** Implementations of this interface are **not inherently thread-safe**. They are invoked on the main server game thread during the entity death processing phase. Implementations must not perform blocking operations or start new threads. All state modification must be contained to the data passed into the method (such as the Objective object), which the engine guarantees is handled in a thread-safe manner relative to the game loop.

**WARNING:** Do not access or modify shared, static state from within a KillTask implementation without proper synchronization, as this can corrupt the server state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| checkKilledEntity(...) | void | O(N) | Callback invoked by the Objective system when an entity is killed. The implementation inspects the event context to determine if it fulfills the objective's criteria and updates the Objective state accordingly. Complexity is dependent on the implementation logic. |

## Integration Patterns

### Standard Usage
A developer's primary interaction with this system is to *implement* the interface to create custom quest logic. The system will then invoke the implementation automatically.

```java
// A task for an objective to kill 10 Goblins
public class KillTenGoblinsTask implements KillTask {
    private static final String GOBLIN_PREFAB = "hytale:goblin";
    private static final int GOAL = 10;

    @Override
    public void checkKilledEntity(Store<EntityStore> killerStore, Ref<EntityStore> victimRef, Objective objective, NPCEntity npc, Damage lastDamage) {
        // Retrieve the entity that was killed
        EntityStore victimStore = victimRef.get();
        if (victimStore == null) {
            return; // Entity already despawned or invalid
        }

        // Check if the victim is a goblin
        if (GOBLIN_PREFAB.equals(victimStore.getPrefabName())) {
            // Get current progress from the objective's state
            int currentKills = objective.getInternalState().getInteger("goblins_killed", 0);

            // Increment and update state
            currentKills++;
            objective.getInternalState().setInteger("goblins_killed", currentKills);

            // Check for completion
            if (currentKills >= GOAL) {
                objective.complete();
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Never call the checkKilledEntity method directly. It is a system-level callback managed exclusively by the server's objective and event loop. Manually calling it can lead to objectives being advanced incorrectly and cause state desynchronization.
- **Storing Entity References:** Do not store direct references to NPCEntity or other entity objects as fields within your implementation. These objects are volatile and can be destroyed at any time. Instead, work with the provided Ref and Store objects within the scope of the method call or store persistent identifiers if necessary.

## Data Pipeline
The KillTask interface is a processor in a larger event-driven data flow. It is triggered deep within the server's entity lifecycle management.

> Flow:
> Entity receives fatal Damage -> Server DamageModule -> Entity death is confirmed -> **EntityDeathEvent** is published to the server event bus -> ObjectiveModule (listener) receives the event -> ObjectiveModule identifies the relevant player and their active Objective -> The Objective delegates to its configured **KillTask** implementation -> KillTask.checkKilledEntity is invoked -> The Objective's state is mutated -> State change is replicated to the client -> Client UI is updated.

