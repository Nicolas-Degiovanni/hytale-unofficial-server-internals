---
description: Architectural reference for StateEvaluatorSystem
---

# StateEvaluatorSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Transient

## Definition
```java
// Signature
public class StateEvaluatorSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The StateEvaluatorSystem is a foundational component of the server-side NPC artificial intelligence engine. It operates within Hytale's Entity Component System (ECS) framework and is singularly responsible for driving state transitions in Non-Player Characters.

This system acts as the executor for the decision-making logic encapsulated within an entity's **StateEvaluator** component. On each processing cycle, it queries for all entities that possess both an **NPCEntity** component and a **StateEvaluator** component. For each matched entity, it invokes the evaluator to determine if the NPC should transition to a new behavioral state, such as changing from *Idle* to *Aggressive*.

Crucially, this system's execution is strictly ordered relative to other AI systems. Its dependencies ensure it runs *after* preliminary AI data is prepared (**PreBehaviourSupportTickSystem**) and *before* the NPC's primary actions are executed (**BehaviourTickSystem**). This ordering guarantees that an NPC's state is definitively resolved for the current tick before it attempts to perform any state-dependent behaviors.

The system is designed for high-performance, parallel execution, processing batches of entities in isolated Archetype Chunks.

## Lifecycle & Ownership
- **Creation:** Instantiated automatically by the server's central ECS runner during world initialization. The necessary ComponentType dependencies are injected by the framework's component registry. Developers do not and should not create instances of this class manually.
- **Scope:** The instance persists for the entire lifetime of the server session or the world it is associated with.
- **Destruction:** The instance is destroyed and garbage collected when the server shuts down or the corresponding world is unloaded.

## Internal State & Concurrency
- **State:** The StateEvaluatorSystem is effectively stateless. Its internal fields are final references to component definitions and dependency configurations, established at construction and never modified. All stateful data it operates on (e.g., an NPC's current state, evaluation timers) is stored externally in the components of the entities it processes.
- **Thread Safety:** This system is thread-safe and designed for parallel execution. The `isParallel` method allows the ECS runner to distribute the workload of processing different entity chunks across multiple threads. The ECS framework guarantees that each `tick` invocation operates on a distinct set of entities, preventing data races. While the system reads component data directly, any state-modifying side effects triggered by the evaluation logic are expected to be queued via the provided thread-safe **CommandBuffer**. The final state transition, however, is a direct mutation on the **StateSupport** component, which is safe within the context of a single entity being processed by one thread at a time.

## API Surface
The public API is exclusively for consumption by the ECS framework runner.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmdBuffer) | void | O(N) | The main entry point called by the ECS runner for each entity matching the query. N is the number of entities. The per-entity cost is determined by the complexity of the associated StateEvaluator logic. |
| getQuery() | Query | O(1) | Returns the ECS query used to identify which entities this system should process. |
| getDependencies() | Set | O(1) | Defines the execution order relative to other systems. Critical for AI logic stability. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. Instead, they enable its functionality by composing an entity with the correct components. The system will automatically discover and process the entity.

```java
// Conceptual example of creating an NPC entity
// This entity will now be processed by the StateEvaluatorSystem each tick.

Entity entity = world.createEntity();
CommandBuffer commands = world.getCommandBuffer();

// Attach the core NPC component
NPCEntity npc = new NPCEntity();
npc.setRole(new MyCustomRole());
commands.addComponent(entity, npc);

// Attach the decision-making logic component
StateEvaluator evaluator = new StateEvaluator(createMyDecisionTree());
commands.addComponent(entity, evaluator);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new StateEvaluatorSystem()`. The system's lifecycle and dependencies are managed exclusively by the ECS runner. Manual instantiation will result in a non-functional, disconnected system.
- **Manual Invocation:** Do not acquire a reference to the system to call its `tick` method. This bypasses the dependency sorting, parallel execution, and scheduling managed by the ECS runner, which can lead to severe race conditions, corrupted AI state, and unpredictable behavior.
- **External State Transitions:** Avoid creating other systems that change an NPC's primary behavioral state. This system is the sole authority for this task. Centralizing state transitions here prevents conflicting logic and ensures the strict execution order is respected.

## Data Pipeline
The system processes data for each entity in a well-defined flow, transforming sensory input and internal state into a potential state change.

> Flow:
> ECS Runner identifies a matching entity -> **StateEvaluatorSystem.tick()** is invoked with the entity's data -> Reads **NPCEntity** and **StateEvaluator** components -> Checks if evaluation is permitted (active and timer elapsed) -> Invokes **StateEvaluator.evaluate()** -> The external evaluation logic runs, returning a chosen **StateOption** -> If a new state is chosen, the system calls **StateSupport.setState()** -> The entity's state component is mutated directly for the current tick.

