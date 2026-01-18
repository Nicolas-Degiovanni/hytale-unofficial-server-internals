---
description: Architectural reference for VoidEventRefSystem
---

# VoidEventRefSystem

**Package:** com.hypixel.hytale.builtin.portals.systems.voidevent
**Type:** System (ECS)

## Definition
```java
// Signature
public final class VoidEventRefSystem extends RefSystem<EntityStore> {
```

## Architecture & Concepts
The VoidEventRefSystem is a reactive, event-driven system within Hytale's Entity Component System (ECS) framework. Its sole responsibility is to manage the lifecycle of entities that possess the **VoidEvent** component. It acts as a high-level orchestrator, translating the simple existence of a component into complex world state changes.

This system functions as a critical bridge between the ECS state and other engine subsystems. Specifically, it couples the addition and removal of a VoidEvent entity to the **Ambience** and **Void Event Stage** systems. When a VoidEvent begins, this system is responsible for initiating environmental effects like music. When it ends, it ensures a clean teardown of those effects and any active gameplay logic associated with the event's stages.

Architecturally, it embodies the "System" pattern in ECS: it is stateless, logic-focused, and operates on a specific subset of entities defined by its query.

### Lifecycle & Ownership
- **Creation:** Instantiated automatically by the server's core System Manager when a World is initialized. There is typically one instance of this system per running World.
- **Scope:** The lifecycle of this system is bound directly to the lifecycle of the World it operates on. It persists as long as the World exists.
- **Destruction:** The system is destroyed and garbage collected when its parent World is unloaded, for instance, during a server shutdown or when a dimension is no longer active.

## Internal State & Concurrency
- **State:** The VoidEventRefSystem is **entirely stateless**. It holds no internal fields or cached data. All state it reads or modifies is external, located in ECS Resources (like PortalWorld and AmbienceResource) or Components (like VoidEvent). This design makes the system highly predictable and robust.

- **Thread Safety:** This system is **not thread-safe** and must not be accessed from multiple threads. The Hytale ECS engine guarantees that its methods are invoked serially on a single, dedicated thread for the corresponding World update tick. Any direct, multi-threaded invocation from outside the engine's control will lead to race conditions and world corruption.

## API Surface
The public methods of this class are not a conventional API for developers. They represent a contract fulfilled for the ECS engine. Direct invocation is a critical anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(...) | void | O(1) | Engine callback. Triggers when an entity with a VoidEvent component is added. Sets the forced ambient music for the world. |
| onEntityRemove(...) | void | O(1) | Engine callback. Triggers when a VoidEvent entity is removed. Clears the forced music and invokes VoidEventStagesSystem to stop any active stage. |
| getQuery() | Query | O(1) | Engine callback. Defines the system's operational scope to entities with the VoidEvent component. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. Its logic is triggered implicitly by manipulating an entity's components via a CommandBuffer. To start a void event, add the component; to stop it, remove the component or destroy the entity.

```java
// Correctly triggering the system's logic
// 'commandBuffer' is provided by the engine in another system's update method.

// To START the void event and its music
Entity entity = ... // An entity to host the event
commandBuffer.addComponent(entity.getRef(), new VoidEvent());

// To STOP the void event and clean up its state
commandBuffer.removeComponent(entity.getRef(), VoidEvent.getComponentType());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new VoidEventRefSystem()`. The ECS engine is solely responsible for the lifecycle of its systems. Attempting to manage it manually will result in a non-functional system that is not registered to receive engine callbacks.

- **Manual Invocation:** Calling `onEntityAdded` or `onEntityRemove` directly is strictly forbidden. This bypasses the ECS framework's state tracking and will cause severe desynchronization between the component state and the world state. For example, calling `onEntityRemove` without actually removing the component would stop the event's music, but the event component would still exist, leading to an inconsistent and broken game state.

## Data Pipeline
The VoidEventRefSystem acts as a processing node in two distinct data flows, translating a component state change into a series of side effects.

> **Flow (Activation):**
> CommandBuffer.addComponent(entity, VoidEvent) → ECS State Change → **VoidEventRefSystem.onEntityAdded** → PortalWorld Resource Read → AmbienceResource.setForcedMusicAmbience(music)

> **Flow (Deactivation):**
> CommandBuffer.removeComponent(entity, VoidEvent) → ECS State Change → **VoidEventRefSystem.onEntityRemove** → AmbienceResource.setForcedMusicAmbience(null) → VoidEventStagesSystem.stopStage()

