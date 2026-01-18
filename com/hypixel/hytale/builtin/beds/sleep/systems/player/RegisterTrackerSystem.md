---
description: Architectural reference for RegisterTrackerSystem
---

# RegisterTrackerSystem

**Package:** com.hypixel.hytale.builtin.beds.sleep.systems.player
**Type:** System / Transient

## Definition
```java
// Signature
public class RegisterTrackerSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
The RegisterTrackerSystem is a foundational component within Hytale's Entity Component System (ECS) framework, specifically designed to bootstrap the sleep mechanic for players. Its role is purely reactive and structural, acting as an automated initialization hook.

This system subscribes to entity creation events within the game world. By implementing the getQuery method to filter for entities possessing a PlayerRef component, it ensures its logic only targets player entities.

Upon the detection of a new player entity entering the world, the system's sole responsibility is to attach a SleepTracker component to that entity. This enforces a critical design contract: **every player entity must be trackable for sleep-related state**. By guaranteeing the presence of the SleepTracker component, it enables other, more complex systems—such as those managing the sleep process, time-skipping, or wake-up events—to operate reliably without needing to perform redundant existence checks. It is a classic example of a "setup" system that prepares entities for more complex gameplay logic.

## Lifecycle & Ownership
- **Creation:** Instantiated automatically by the server's ECS SystemManager during world or game mode initialization. This system is discovered and loaded as part of a predefined set of systems for the sleep module. Manual instantiation is not supported.
- **Scope:** The instance persists for the entire lifetime of the server world it is registered with. Its lifecycle is directly managed by the parent ECS framework.
- **Destruction:** The instance is destroyed and garbage collected when the world is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** The RegisterTrackerSystem is **stateless**. It contains no member variables, caches no data, and its operations are idempotent. Its behavior is exclusively determined by the entity data passed into its callback methods. This design makes it highly predictable and robust.
- **Thread Safety:** This class is inherently **thread-safe** due to its stateless nature. However, it is designed to be operated by a single-threaded game loop or a world tick scheduler. All method invocations are expected to be serialized by the parent ECS framework. Unsynchronized, concurrent calls from external threads will lead to world state corruption and are strictly forbidden.

## API Surface
The public API consists of framework-level callbacks, which are not intended for direct invocation by game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | **Framework Callback.** Invoked when a new entity matching the query is added to the world. Attaches a SleepTracker component. |
| onEntityRemoved(holder, reason, store) | void | O(1) | **Framework Callback.** A no-op in this implementation. The system takes no action when a player entity is removed. |
| getQuery() | Query | O(1) | **Framework Callback.** Provides the entity filter, selecting only entities that have a PlayerRef component. |

## Integration Patterns

### Standard Usage
This system is not used by calling its methods directly. Instead, it is registered with the engine's SystemManager, typically as part of a game mode's definition. The framework handles the rest.

```java
// Example of how a SystemManager might be configured
// This code is conceptual and does not represent a direct API call.

SystemRegistry registry = server.getSystemRegistry();
registry.register(RegisterTrackerSystem.class);

// The engine will now automatically invoke the system for new players.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new RegisterTrackerSystem()`. The ECS framework is responsible for its lifecycle. Doing so will result in a non-functional system that is not registered to receive entity events.
- **Manual Invocation:** Do not call `onEntityAdd` directly. This bypasses the ECS framework's authority and can lead to race conditions or an inconsistent world state.
- **Extending for Other Logic:** Avoid subclassing this system to add unrelated logic. Its purpose is narrowly defined to component initialization. Complex behaviors should be implemented in separate, dedicated systems.

## Data Pipeline
The system acts as a simple, event-driven processor in the entity creation pipeline.

> Flow:
> Player Entity Added to World -> ECS System Manager Dispatches Event -> **RegisterTrackerSystem** (Query Match) -> `onEntityAdd` Triggered -> `holder.ensureComponent` -> Player Entity now has a `SleepTracker` component

