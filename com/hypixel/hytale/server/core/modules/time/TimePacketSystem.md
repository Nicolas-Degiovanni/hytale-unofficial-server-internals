---
description: Architectural reference for TimePacketSystem
---

# TimePacketSystem

**Package:** com.hypixel.hytale.server.core.modules.time
**Type:** System Component

## Definition
```java
// Signature
public class TimePacketSystem extends DelayedSystem<EntityStore> {
```

## Architecture & Concepts
The TimePacketSystem is a server-side component within the Entity Component System (ECS) framework responsible for synchronizing game time with connected clients. It functions as a specialized, time-based trigger rather than a continuous process.

By extending DelayedSystem, it decouples its execution from the main game tick, operating on a fixed interval defined by BROADCAST_INTERVAL (1.0 second). This design choice is critical for performance, as it prevents network saturation by ensuring time synchronization packets are sent periodically, not on every single server frame.

This class embodies the principle of separation of concerns. It does not own or manipulate the world's time state. Instead, it acts as a scheduled invoker for the authoritative WorldTimeResource, which holds the actual time data. The TimePacketSystem's sole responsibility is to query this resource at the correct interval and initiate the broadcast process.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's central SystemManager or a similar dependency injection framework during world initialization. Its dependency on WorldTimeResource is resolved and injected at this stage.
- **Scope:** The lifecycle of a TimePacketSystem instance is strictly bound to its parent World. It remains active for the entire duration that the world is loaded and running on the server.
- **Destruction:** The instance is marked for garbage collection when the associated World is unloaded. Cleanup is managed implicitly by the ECS framework; there are no public destruction or teardown methods.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its only field, worldTimeResourceType, is an immutable handle injected during construction. All operative data, such as the current world time, is read directly from the external WorldTimeResource on each execution. The timing state is managed internally by the parent DelayedSystem.
- **Thread Safety:** This system is **not thread-safe** and is designed for single-threaded access. It must be executed exclusively by the main server thread that manages the ECS update loop. Accessing its methods or the underlying Store from a concurrent thread will result in race conditions and severe state corruption.

## API Surface
The public API is minimal and intended for framework integration, not direct developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| delayedTick(dt, systemIndex, store) | void | O(1) | Framework-invoked method that executes the time broadcast logic. Checks if time is paused and delegates to WorldTimeResource. |

## Integration Patterns

### Standard Usage
This system is not designed for direct developer interaction. It is automatically registered with the server's SystemManager during the bootstrap process and operates autonomously as part of the core game loop. Its presence ensures time is synchronized; no manual calls are necessary or supported.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new TimePacketSystem()`. The system's lifecycle and dependencies are managed by the engine's core framework. Manual creation will result in a non-functional object.
- **Manual Invocation:** Calling the delayedTick method directly circumvents the timing and scheduling logic of the parent DelayedSystem. This can lead to network flooding if called in a tight loop or time desynchronization if not called frequently enough.
- **Conditional Disabling:** Logic to pause time synchronization should be implemented by pausing the game time via the WorldConfig. The system correctly respects the `isGameTimePaused` flag. Do not attempt to manually remove this system from the scheduler at runtime.

## Data Pipeline
The TimePacketSystem acts as a periodic trigger in a larger data flow, initiating the process of sending time state from the server to clients.

> Flow:
> ECS Scheduler Timer -> **TimePacketSystem.delayedTick()** -> WorldTimeResource.broadcastTimePacket() -> Server Network Layer -> Client Game State

