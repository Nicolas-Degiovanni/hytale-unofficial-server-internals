---
description: Architectural reference for MoonPhaseChangeEventSystem
---

# MoonPhaseChangeEventSystem

**Package:** com.hypixel.hytale.server.spawning.world.system
**Type:** Transient System

## Definition
```java
// Signature
public class MoonPhaseChangeEventSystem extends WorldEventSystem<EntityStore, MoonPhaseChangeEvent> {
```

## Architecture & Concepts
The MoonPhaseChangeEventSystem is a reactive, event-driven component within the server-side Entity-Component-System (ECS) architecture. Its sole responsibility is to act as a bridge between the world's celestial mechanics and the entity spawning logic.

By extending WorldEventSystem and subscribing to the MoonPhaseChangeEvent, this system decouples the time-of-day simulation from the complex rules governing mob spawning. When the world's moon phase changes, this system is automatically invoked to update the spawn weightings for various environments. This ensures that environmental factors dynamically influence which entities are likely to appear, without requiring the spawning system to constantly poll the world state.

This class is a classic example of a "pure system" in ECS design: it is stateless and exists only to transform data in response to a specific trigger.

### Lifecycle & Ownership
- **Creation:** An instance of MoonPhaseChangeEventSystem is created and registered by a higher-level system manager, typically during the initialization of a server world instance. It is not intended for manual instantiation.
- **Scope:** The system's lifecycle is tightly bound to the server world it belongs to. It persists as long as that world is loaded and active.
- **Destruction:** The instance is garbage collected when the corresponding server world is unloaded and the managing system registry is cleared.

## Internal State & Concurrency
- **State:** This system is **stateless**. It does not maintain any internal fields or cache data between invocations of its handle method. All state is read from the incoming event object or retrieved from shared world resources via the CommandBuffer.

- **Thread Safety:** The system itself is inherently thread-safe due to its stateless nature. However, it operates on shared, mutable state within the WorldSpawnData resource. Concurrency is managed by the parent ECS framework, which guarantees that system execution is serialized for a given world update tick. The use of a CommandBuffer further ensures that mutations are queued and applied in a deterministic order, preventing race conditions with other systems.

    **Warning:** Direct, multi-threaded invocation of the handle method would be catastrophic to world state. All interaction must be mediated by the engine's event bus.

## API Surface
The public API is minimal, consisting only of the event handler method invoked by the engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handle(store, commandBuffer, event) | void | O(N) | Handles the MoonPhaseChangeEvent. Recalculates spawn weights for all N configured environments. This is the system's primary entry point, called exclusively by the event bus. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. Instead, they trigger its logic by firing a MoonPhaseChangeEvent on the appropriate world event bus. The engine's system manager ensures the event is routed to this system's handle method.

```java
// Conceptual example of how another system would trigger this one.
// DO NOT call MoonPhaseChangeEventSystem directly.

// Somewhere in the world's time-of-day simulation code...
MoonPhase newPhase = world.celestialBody.calculateNewMoonPhase();
MoonPhaseChangeEvent event = new MoonPhaseChangeEvent(oldPhase, newPhase);

// The event bus will automatically find and invoke MoonPhaseChangeEventSystem
world.getEventBus().fire(event);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new MoonPhaseChangeEventSystem()`. The system must be managed by the world's central system registry to be correctly integrated into the event pipeline.
- **Manual Invocation:** Do not call the `handle` method directly. This bypasses the engine's concurrency model and command buffer, leading to unpredictable state corruption and race conditions. Always publish an event to the event bus.

## Data Pipeline
The system acts as a specific processor in a larger event-driven data flow. It translates a high-level world event into a low-level data modification within the spawning system's configuration.

> Flow:
> World Time Simulation -> Fires **MoonPhaseChangeEvent** -> World Event Bus -> **MoonPhaseChangeEventSystem.handle()** -> Retrieves WorldSpawnData Resource -> Modifies Spawn Weights In-Memory

