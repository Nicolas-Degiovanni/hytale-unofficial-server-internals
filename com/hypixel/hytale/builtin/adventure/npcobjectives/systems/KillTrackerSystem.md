---
description: Architectural reference for KillTrackerSystem
---

# KillTrackerSystem

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.systems
**Type:** System

## Definition
```java
// Signature
public class KillTrackerSystem extends DeathSystems.OnDeathSystem {
```

## Architecture & Concepts
The KillTrackerSystem is a reactive, stateless system operating within the server-side Entity-Component-System (ECS) framework. Its sole responsibility is to monitor the deaths of NPC entities and determine if those deaths satisfy the conditions of any active "kill-based" quest objectives.

This system acts as a critical bridge between the generic, engine-level **Death System** and the game-specific **Adventure Objectives Module**. By extending `DeathSystems.OnDeathSystem`, it subscribes to a specific event in an entity's lifecycle: the moment it is marked for death by the addition of a `DeathComponent`.

Its core design function is to decouple quest logic from combat mechanics. The core damage and death systems do not need any knowledge of quests. Instead, this system observes death events and applies the relevant objective logic after the fact, promoting modularity and separation of concerns. The `getQuery` method ensures this system's logic only executes for entities that are explicitly designated as NPCs, preventing unnecessary processing for other entity types.

### Lifecycle & Ownership
- **Creation:** Instantiated automatically by the server's `SystemGraph` during world initialization. Systems are part of the core world definition and are not created manually by game logic developers.
- **Scope:** Singleton per world instance. It persists for the entire lifecycle of a game world.
- **Destruction:** The system is destroyed and its memory is reclaimed when the corresponding game world is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** The KillTrackerSystem is entirely **stateless**. It holds no internal fields or cached data. All state it operates on, such as the list of active quests, is retrieved from a global `KillTrackerResource` via the world `Store` during method execution. This is a fundamental pattern for ECS systems to ensure predictable, data-driven behavior.
- **Thread Safety:** This system is **not thread-safe** and must not be invoked from outside the main server thread. The ECS engine guarantees that all systems for a given world are executed sequentially within a single tick, preventing race conditions. Any attempt to access this system from a worker thread will lead to world state corruption.

## API Surface
The public contract is implicitly defined by its parent class, `DeathSystems.OnDeathSystem`. Direct invocation is an anti-pattern; the engine's scheduler is the sole caller.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | **Engine Callback.** Returns a query that filters for entities with an NPCEntity component. |
| onComponentAdded(...) | void | O(N) | **Engine Callback.** Triggered when a DeathComponent is added to a matching entity. N is the number of active kill tasks. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. To leverage its functionality, a developer creates and registers a quest objective that results in a `KillTaskTransaction` being added to the global `KillTrackerResource`. The system will then automatically detect and process relevant NPC deaths.

```java
// PSEUDOCODE: How to create a task this system will process

// 1. Get the world's central resource for kill quests
KillTrackerResource tracker = world.getResource(KillTrackerResource.class);

// 2. Create a new transaction representing the quest objective
// (The actual creation logic is handled by the objective system)
KillTaskTransaction newKillQuest = createQuestObjective(...);

// 3. Add the transaction to the global resource
// The KillTrackerSystem will now monitor this task.
tracker.getKillTasks().add(newKillQuest);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new KillTrackerSystem()`. An unmanaged instance will not be registered with the engine's scheduler and will never execute.
- **Manual Invocation:** Do not call the `onComponentAdded` method directly. This bypasses the ECS scheduler and can cause severe state desynchronization, as it would be operating outside the context of a valid world tick.

## Data Pipeline
The system is triggered as the final step in a chain of events originating from entity damage. It reads from a global resource and delegates the final logic check to the transaction object itself.

> Flow:
> NPCEntity receives fatal damage -> Core Damage System adds `DeathComponent` -> ECS Scheduler dispatches event to subscribers -> **KillTrackerSystem.onComponentAdded** is invoked -> System reads `KillTrackerResource` -> System iterates `KillTaskTransaction` list -> `KillTaskTransaction.checkKilledEntity` is called for each task.

