---
description: Architectural reference for CombatActionEvaluatorSystems.EvaluatorTick
---

# CombatActionEvaluatorSystems.EvaluatorTick

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator
**Type:** ECS System

## Definition
```java
// Signature
public static class EvaluatorTick extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The **EvaluatorTick** system is the core decision-making engine for Non-Player Character (NPC) combat AI. It operates within the server-side Entity Component System (ECS) framework, executing once per game tick for every entity that possesses the required combat components.

This system is not a monolithic AI brain; rather, it is a specialized processor that translates high-level game state into concrete combat actions. It functions as a state-driven evaluator, constantly checking if an NPC is in its designated combat state. When active, it manages a complex lifecycle of target selection, action evaluation, cooldowns, and timeouts.

Its primary responsibilities include:
- Driving the selection of "basic attacks" during lulls in combat.
- Triggering the evaluation of more complex, utility-scored "combat actions" based on cooldowns and environmental context.
- Maintaining the NPC's focus on a primary target and communicating this target to other systems via the **MarkedEntitySupport** mechanism.
- Terminating actions if a target is lost, dies, or if the action times out.

This system is the central hub that connects an NPC's configured combat behaviors (**CombatBalanceAsset**) with its real-time state (**TransformComponent**, **TargetMemory**) to produce tangible actions.

### Lifecycle & Ownership
- **Creation:** A single instance of **EvaluatorTick** is created and registered with the world's system scheduler during the server's bootstrap sequence, likely by the **NPCCombatActionEvaluatorPlugin**. It is not instantiated per-entity.
- **Scope:** The system is a long-lived, singleton-like object that persists for the entire server session. It is fundamentally stateless itself; all operational state is stored within the components of the entities it processes.
- **Destruction:** The system is destroyed and de-registered during server shutdown.

## Internal State & Concurrency
- **State:** **EvaluatorTick** is designed to be stateless. It holds no mutable fields that persist between ticks or across different entities. All data, such as cooldown timers, current targets, or action state, is read from and written to components like **CombatActionEvaluator** and **ValueStore** on a per-entity basis.
- **Thread Safety:** This system is explicitly non-parallel. The **isParallel** method returns false, which is a strict contract with the ECS scheduler. This guarantees that the **tick** method will be executed serially for all applicable entities. This design choice eliminates the need for internal locking or synchronization primitives, as it operates within a single-threaded context during its execution phase. All cross-system communication and structural changes (adding/removing components) are safely deferred via the **CommandBuffer**.

## API Surface
The public API is consumed exclusively by the ECS scheduler, not by gameplay developers directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmd) | void | O(N) | The main entry point, executed by the engine for each entity matching the query. Contains the entire combat decision logic for one entity for one frame. |
| getQuery() | Query | O(1) | Defines the component signature of entities this system will operate on. Critical for ECS scheduling. |
| getDependencies() | Set | O(1) | Declares execution order dependencies. Guarantees that **TargetMemorySystems.Ticking** runs before this system, ensuring target data is fresh. |

## Integration Patterns

### Standard Usage
A developer does not call this system's methods directly. Instead, they enable it by composing an entity with the correct set of components. The system is configured through data, not code.

1.  An entity archetype is defined to include **CombatConstructionData**.
2.  The **OnAdded** system processes this, adding the **CombatActionEvaluator**, **TargetMemory**, and other required components based on a **CombatBalanceAsset**.
3.  Once the components are present, the ECS scheduler automatically includes the entity in the **EvaluatorTick** system's processing loop on every subsequent tick.

```java
// This system is not used directly in code.
// It is enabled by adding the correct components to an entity archetype.
// Example: In a JSON or asset definition for an NPC
"components": {
  "hytale:npc_entity": { ... },
  "hytale:combat_construction_data": {
    "combatState": "COMBAT",
    "markedTargetSlot": 1,
    ...
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Never instantiate or call the **tick** method manually. Doing so bypasses the ECS scheduler, dependency ordering, and the **CommandBuffer**, which will lead to race conditions, state corruption, and unpredictable crashes.
- **Incorrect Component State:** Adding a **CombatActionEvaluator** component without the other components defined in **getQuery** (like **TargetMemory** or **ValueStore**) will cause the system to ignore the entity, or worse, throw assertions if it does process it. Always use the intended **OnAdded** setup system.

## Data Pipeline
This system processes component data to update other components, driving the NPC's behavior.

> Flow:
> **TargetMemory** (updated by another system) -> **EvaluatorTick** reads target data -> Evaluates options from **CombatActionEvaluator** config -> Selects action -> Updates **CombatActionEvaluator** state (current action, cooldowns) -> Writes target reference to **Role.MarkedEntitySupport** -> Other systems (Movement, Animation) react to the marked target and action state.

---
---
description: Architectural reference for CombatActionEvaluatorSystems.OnAdded
---

# CombatActionEvaluatorSystems.OnAdded

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator
**Type:** ECS Lifecycle System

## Definition
```java
// Signature
public static class OnAdded extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
The **OnAdded** system is a specialized bootstrap mechanism within the ECS framework. It follows the "setup system" pattern, where its sole responsibility is to react to the creation of new entities and configure them for a specific purpose. In this case, it initializes an NPC for participation in the combat evaluation system.

It listens for any entity that is created with the temporary **CombatConstructionData** component. Upon detection, it performs a one-time setup routine:
1.  It retrieves the NPC's **Role** and associated **CombatBalanceAsset**.
2.  It uses the configuration from the balance asset and the construction data to instantiate and configure the core **CombatActionEvaluator** component.
3.  It adds other necessary components, such as **TargetMemory**, which is required for the main **EvaluatorTick** system to function.
4.  Finally, it removes the **CombatConstructionData** component, as its purpose is complete.

This pattern decouples the definition of an NPC archetype from the complex construction of its runtime components, allowing for a data-driven and centralized initialization process.

### Lifecycle & Ownership
- **Creation:** A single instance of **OnAdded** is created and registered with the world's system scheduler during server startup, typically by the **NPCCombatActionEvaluatorPlugin**.
- **Scope:** This system is a long-lived, singleton-like object that persists for the entire server session.
- **Destruction:** The system is destroyed and de-registered during server shutdown.

## Internal State & Concurrency
- **State:** **OnAdded** is entirely stateless. It does not store any information between invocations.
- **Thread Safety:** As a **HolderSystem**, its callbacks (**onEntityAdd**, **onEntityRemoved**) are invoked by the ECS world during entity lifecycle events. These events are managed within a controlled, typically serial, context, ensuring that the component additions and removals are atomic and safe from race conditions. The explicit dependency on **BalancingInitialisationSystem** is critical, as it prevents this system from running before the required asset data has been loaded, avoiding null pointer exceptions.

## API Surface
The public API is a contract with the ECS framework and is not intended for direct developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Callback executed by the engine when an entity matching the query is added to the world. Contains the component setup logic. |
| getQuery() | Query | O(1) | Defines the component signature that triggers this system: any entity with **CombatConstructionData**. |
| getDependencies() | Set | O(1) | Declares that this system must run *after* **BalancingInitialisationSystem**. |

## Integration Patterns

### Standard Usage
This system is used implicitly by defining an NPC archetype with the **CombatConstructionData** component. When an entity is spawned from this archetype, the system automatically executes its setup logic.

```java
// In an NPC archetype definition (e.g., JSON)
// Adding this component ensures the OnAdded system will run for this entity upon creation.
"components": {
  "hytale:combat_construction_data": {
    "combatState": "COMBAT",
    "markedTargetSlot": 1,
    "minRangeSlot": 10,
    "maxRangeSlot": 11,
    "positioningAngleSlot": 12
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Component Addition:** Do not manually add the **CombatActionEvaluator** or **TargetMemory** components to an entity. This bypasses the critical, data-driven configuration performed by this system, which will result in an uninitialized or incorrectly configured NPC that will fail to behave correctly in combat.
- **Persistent Construction Data:** Do not design logic that relies on the **CombatConstructionData** component existing after initialization. This system is designed to remove it, and its presence post-setup is an indicator of a fault.

## Data Pipeline
This system transforms a temporary data component into a set of permanent, functional runtime components.

> Flow:
> Entity Spawn with **CombatConstructionData** -> **OnAdded.onEntityAdd** is triggered -> Reads **CombatBalanceAsset** -> Instantiates **CombatActionEvaluator** and **TargetMemory** -> Adds new components to the entity -> Removes **CombatConstructionData** from the entity -> Entity is now ready for processing by **EvaluatorTick**.

