---
description: Architectural reference for ReputationAttitudeSystem
---

# ReputationAttitudeSystem

**Package:** com.hypixel.hytale.builtin.adventure.npcreputation
**Type:** System Component

## Definition
```java
// Signature
public class ReputationAttitudeSystem extends StoreSystem<EntityStore> {
```

## Architecture & Concepts
The ReputationAttitudeSystem is a server-side System within the Entity-Component-System (ECS) framework. Its sole architectural purpose is to act as a bridge, injecting game-specific reputation logic into the generic Non-Player Character (NPC) artificial intelligence engine.

It operates by registering a high-priority *provider* with the NPC's AI **Blackboard**. A Blackboard is a centralized data store for an AI agent, and its **AttitudeView** is the specific interface used to query how that agent feels about other entities.

When an NPC's behavior tree needs to make a decision based on its attitude towards a target (e.g., "Should I attack this entity?"), the AttitudeView queries all registered providers. This system's provider specifically handles the case where the target is a **Player**. It does not calculate attitude itself; instead, it delegates the query to the global **ReputationPlugin**, which manages the complex rules of player reputation.

In essence, this class ensures that an NPC's abstract concept of "attitude" is directly influenced by the concrete "reputation" a player has earned.

### Lifecycle & Ownership
The lifecycle of a ReputationAttitudeSystem is managed entirely by the server's ECS framework and is tightly coupled to the lifecycle of an individual NPC entity.

- **Creation:** An instance is created and attached by the engine when an NPC with a Blackboard component is initialized in the world. This process is automatic and driven by entity configuration.
- **Scope:** The instance persists only as long as its associated **Store** (representing the NPC) exists in the game world. It is not a global singleton; each NPC with the requisite components will have its own instance of this system.
- **Destruction:** The instance is marked for garbage collection when its parent Store is destroyed, which typically occurs when an NPC is killed or unloaded from a world chunk. The **onSystemRemovedFromStore** callback is invoked immediately prior to destruction, though it performs no cleanup in this implementation.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. The single instance field, resourceType, is an immutable definition handle. All operations are performed on the Store and entity references passed into its methods by the ECS framework. It performs no internal caching.

- **Thread Safety:** The class is inherently thread-safe due to its stateless design. However, all system methods are designed to be called synchronously from the main server thread responsible for the corresponding world tick. The ECS framework guarantees that systems operate on their Store in a controlled, non-concurrent manner.

    **WARNING:** Manually invoking its methods from an asynchronous thread will violate engine invariants and lead to severe concurrency issues, such as race conditions and data corruption.

## API Surface
The public API consists exclusively of lifecycle methods intended for invocation by the ECS framework. Direct calls by game logic are not supported and are considered an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onSystemAddedToStore(Store) | void | O(1) | Framework callback. Registers the reputation-based attitude provider with the NPC's AI Blackboard. |
| onSystemRemovedFromStore(Store) | void | O(1) | Framework callback. Currently a no-op; performs no cleanup. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. Its integration is declarative, defined within the asset files for a given NPC. The engine automatically instantiates and attaches the system to any entity that includes a **Blackboard** component.

The primary interaction point for developers is with the **ReputationPlugin**, which this system uses as its data source.

```java
// This system is used implicitly by the engine.
// A developer would modify reputation via the plugin,
// and this system would then use that data.

// Example of influencing the system indirectly:
ReputationPlugin.get().addReputation(player, faction, 100);

// The next time an NPC checks its attitude towards the player,
// ReputationAttitudeSystem will query the new value.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ReputationAttitudeSystem()`. The ECS framework is solely responsible for its creation and lifecycle management. Doing so will result in a non-functional object that is not registered with any entity.
- **Manual Invocation:** Do not call `onSystemAddedToStore` or `onSystemRemovedFromStore` manually. These are lifecycle hooks for the engine. Manual invocation will disrupt the AI's initialization state and may cause runtime exceptions or inconsistent behavior.

## Data Pipeline
This system participates in a request-response data flow initiated by an NPC's AI. It does not proactively push data.

> Flow:
> NPC Behavior Tree (e.g., "Evaluate Target") -> Query **Blackboard** for Attitude -> **AttitudeView** invokes providers -> **ReputationAttitudeSystem** provider is triggered -> System checks if target is a Player -> System calls **ReputationPlugin** -> Reputation value returned -> Value becomes AI's attitude -> Behavior Tree makes decision (e.g., Flee, Ignore, Attack)

