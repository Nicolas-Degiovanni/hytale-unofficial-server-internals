---
description: Architectural reference for VoidEventStagesSystem
---

# VoidEventStagesSystem

**Package:** com.hypixel.hytale.builtin.portals.systems.voidevent
**Type:** System Component

## Definition
```java
// Signature
public class VoidEventStagesSystem extends DelayedEntitySystem<EntityStore> {
```

## Architecture & Concepts
The VoidEventStagesSystem is a time-driven state machine manager operating within the server's Entity Component System (ECS) framework. Its primary responsibility is to progress a running *Void Event* through its predefined stages based on elapsed time.

This system acts as the orchestrator for the event's timeline. It does not contain the logic for the stages themselves; rather, it reads a VoidEventConfig data structure, determines which stage should be active at a given moment, and triggers the transition. The effects of a stage transition, such as changing the world's weather, are executed via static utility methods, decoupling the timeline logic from the gameplay effects.

By extending DelayedEntitySystem, this system is designed for low-frequency execution (every 1.5 seconds by default), making it efficient for long-running background events where sub-second precision is not required. It operates reactively on data present in shared *Resources* (PortalWorld, WeatherResource) and *Components* (VoidEvent).

### Lifecycle & Ownership
- **Creation:** Instantiated automatically by the server's ECS System Manager during world initialization. Developers **must not** create instances of this class manually.
- **Scope:** The system is scoped to a running server world. A single instance exists for the lifetime of the world, processing any and all VoidEvent entities.
- **Destruction:** The instance is destroyed and garbage collected when the server world is unloaded.

## Internal State & Concurrency
- **State:** The VoidEventStagesSystem is fundamentally **stateless**. It holds no mutable instance fields and derives all necessary information from the ECS Store and CommandBuffer during each tick. Its behavior is a pure function of the current world state (specifically, the PortalWorld resource and VoidEvent component) and the passage of time.

- **Thread Safety:** This system is **not thread-safe** and is designed for exclusive execution on the main server thread. The ECS framework guarantees that the tick method is called sequentially and not concurrently.
    - **WARNING:** Calling any of its methods from an asynchronous task or a different thread will lead to severe concurrency violations, including data corruption and ConcurrentModificationExceptions, as it accesses non-thread-safe ECS containers like Store and CommandBuffer.

## API Surface
The primary entry point is the tick method, which is invoked by the ECS scheduler. The startStage and stopStage methods are exposed as public static utilities to allow other systems to potentially trigger stage effects outside the normal timeline, though this is an advanced use case.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, buffer) | void | O(S) | Called by the ECS scheduler. Computes the desired stage and applies changes. Complexity is linear to S, the number of stages in the event config. |
| startStage(stage, world, store, buffer) | static void | O(1) | Applies the starting effects of a given stage, such as forcing a specific weather pattern. |
| stopStage(stage, world, store, buffer) | static void | O(1) | Reverts the effects of a given stage, such as clearing a forced weather pattern. |

## Integration Patterns

### Standard Usage
Direct interaction with this system is not a standard pattern. Instead, developers configure the data that the system consumes. The system activates automatically when a PortalWorld resource exists and an entity with a VoidEvent component is present.

The following example demonstrates how to set up the prerequisite data that would cause the VoidEventStagesSystem to begin its work.

```java
// In another system or setup logic:
// 1. Get access to the world's resources and command buffer.
PortalWorld portalWorld = store.getResource(PortalWorld.getResourceType());
WeatherResource weather = store.getResource(WeatherResource.getResourceType());

// 2. Ensure the PortalWorld is configured and a VoidEvent entity exists.
// This is typically done when a player enters a portal zone.
if (portalWorld.exists() && portalWorld.getVoidEventRef() != null) {
    // 3. The VoidEventStagesSystem will now automatically run its 'tick' logic
    // on its next scheduled update, reading from PortalWorld and modifying
    // the VoidEvent component and WeatherResource as needed.
    // No direct call to the system is necessary.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new VoidEventStagesSystem()`. The ECS framework is responsible for the lifecycle of all systems. Manual instantiation will result in a non-functional object that is not registered with the game loop.
- **Manual Ticking:** Never call the `tick` method directly. This bypasses the engine's scheduler and can lead to unpredictable behavior, especially concerning the `DelayedEntitySystem` timing logic.
- **State Storage:** Do not modify the system to store state in instance fields. This breaks the stateless design and can cause issues with world serialization or system reloads. All state must be stored in ECS Components or Resources.

## Data Pipeline
The system functions as a processor in a larger data flow. It reads from multiple sources, computes a new state, and writes its results back into the ECS for other systems to consume.

> Flow:
> Game Clock Tick -> PortalWorld Resource (Time updated) -> **VoidEventStagesSystem** (Reads time, compares against VoidEventConfig) -> CommandBuffer (Writes new active stage to VoidEvent component) -> WeatherResource (Modified by start/stopStage) -> Weather System (Reacts to forced weather change)

