---
description: Architectural reference for Blackboard
---

# Blackboard

**Package:** com.hypixel.hytale.server.npc.blackboard
**Type:** Singleton (per World Resource)

## Definition
```java
// Signature
public class Blackboard implements Resource<EntityStore> {
```

## Architecture & Concepts
The Blackboard is the central nervous system for the server-side Non-Player Character (NPC) AI. It implements the classic **Blackboard System** architectural pattern, acting as a shared, observable repository of world state information tailored for AI consumption.

Its primary function is to decouple sensory data producers (e.g., game event listeners) from AI data consumers (e.g., behavior trees, goal planners). Instead of AI logic directly subscribing to dozens of game events, the Blackboard subscribes, processes, and organizes this information into specialized, queryable data structures called **Views**.

Each View, such as AttitudeView or BlockTypeView, represents a specific facet of the game world relevant to AI decision-making. The Blackboard manages a collection of these views through corresponding IBlackboardViewManager instances, which handle the storage, lifecycle, and access patterns for each type of view. This design allows for a modular and extensible AI sensory system where new types of world-state awareness can be added by simply registering a new view type.

## Lifecycle & Ownership
The Blackboard's lifecycle is tightly coupled to the lifecycle of a server World. It is not a global singleton but rather a singleton resource within the context of a specific world instance.

- **Creation:** A Blackboard instance is not created directly by developers. It is instantiated by the server's Entity Component System (ECS) framework when the NPCPlugin is initialized for a new World. Its type is registered via the static getResourceType method.
- **Scope:** The instance persists for the entire duration of the World it is associated with. The init method is called early in the world's setup phase to register the core AI views.
- **Destruction:** The instance is marked for garbage collection when the corresponding World is unloaded. The onWorldRemoved method is invoked as a final cleanup hook, which cascades the shutdown signal to all registered view managers.

## Internal State & Concurrency
- **State:** The Blackboard's state is highly mutable, as it reflects the real-time state of the game world. Its primary state is the registry of view managers, stored in a ConcurrentHashMap. This map is populated during the init phase and is not expected to change frequently during runtime. The state of the individual views it manages is constantly changing in response to game events.

- **Thread Safety:** The class is designed to be thread-safe for its primary operations.
    - The use of ConcurrentHashMap ensures that registering and retrieving view managers can be done safely from multiple threads.
    - Event handling methods like onEntityDamageBlock iterate over the map's values, which is a safe operation on ConcurrentHashMap.
    - **WARNING:** While the Blackboard container is thread-safe, it provides no thread-safety guarantees for the IBlackboardView or IBlackboardViewManager implementations it holds. It is the responsibility of each view implementation to ensure its own internal concurrency-safety.

## API Surface
The public API is focused on two main capabilities: retrieving views and propagating events.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(World) | void | O(N) | Initializes the Blackboard, registering all default view managers. Must be called once per lifecycle. |
| getView(...) | View | O(1) | Retrieves a specific View instance based on its class and a context (e.g., entity, chunk). Throws NullPointerException if the view type is not registered. |
| onEntityDamageBlock(...) | void | O(M) | Receives a block damage event and broadcasts it to all registered view managers for processing. M is the number of registered managers. |
| onEntityBreakBlock(...) | void | O(M) | Receives a block break event and broadcasts it to all registered view managers. |
| cleanupViews() | void | O(M) | Triggers a cleanup cycle on all managed views. |

## Integration Patterns

### Standard Usage
The Blackboard should always be retrieved as a resource from the appropriate context, typically the World or a system that has access to world resources. From there, specific views are requested to query AI-relevant state.

```java
// Example: An AI system retrieving the AttitudeView to make a decision
// Note: 'world' is a pre-existing World object.

Blackboard blackboard = world.getResource(Blackboard.getResourceType());
AttitudeView attitudeView = blackboard.getView(AttitudeView.class);

// Now use the view to query state
Attitude towardsPlayer = attitudeView.getAttitude(npcEntityRef, playerEntityRef);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new Blackboard()`. The object is managed by the ECS resource system. Manual creation will result in an uninitialized and non-functional object that is not integrated with the game's event loop.

- **Access Before Initialization:** Do not attempt to retrieve views from the Blackboard before the World's `init` sequence is complete. This can lead to a NullPointerException, as the view managers will not have been registered yet.

- **Modification of Internal State:** Do not attempt to access or modify the internal `views` map via reflection. This can break the thread-safety guarantees and lead to unpredictable behavior.

## Data Pipeline
The Blackboard acts as a central hub in the AI data pipeline, transforming raw game events into structured, queryable AI knowledge.

> Flow:
> Game Event (e.g., BreakBlockEvent) -> World Event Bus -> **Blackboard** Event Handlers (e.g., onEntityBreakBlock) -> Fan-out to all IBlackboardViewManagers -> IBlackboardView State Update -> AI System (e.g., Behavior Tree) queries a specific IBlackboardView for decision-making data.

