---
description: Architectural reference for InteractionSimulationHandler
---

# InteractionSimulationHandler

**Package:** com.hypixel.hytale.server.core.modules.interaction
**Type:** Transient

## Definition
```java
// Signature
public class InteractionSimulationHandler implements IInteractionSimulationHandler {
```

## Architecture & Concepts
The InteractionSimulationHandler is a foundational, stateful component within the server's interaction module. It serves as the default, and most basic, implementation of the IInteractionSimulationHandler interface. Its sole responsibility is to track the raw binary state (pressed or not pressed) of various interaction types for a single entity, typically a player.

This class acts as the initial entry point for player input into the server-side simulation. It does not contain complex game logic such as cooldowns, charge modifiers, or contextual validation. Instead, it provides a simple, direct representation of the player's controller state, which is then consumed by higher-level systems that orchestrate the complete interaction logic. It is a pure state machine, driven externally by network events and polled internally by the game loop.

### Lifecycle & Ownership
- **Creation:** An instance of InteractionSimulationHandler is typically created by a parent module responsible for a specific entity's lifecycle, such as a Player or an InteractionComponent. It is not a globally shared service.
- **Scope:** The lifetime of an InteractionSimulationHandler is tightly coupled to the entity it represents. It persists as long as the entity is being actively simulated on the server.
- **Destruction:** The object is eligible for garbage collection when its owning entity is removed from the world or its session is terminated. It requires no explicit cleanup.

## Internal State & Concurrency
- **State:** This component is **Mutable**. Its core state is the private final boolean array named isDown, which tracks the current state for every possible InteractionType. This array is directly modified via the setState method.
- **Thread Safety:** This class is **Not Thread-Safe**. All methods perform unsynchronized reads and writes on the isDown array. Concurrent access from multiple threads will lead to race conditions and undefined behavior.

**WARNING:** This handler must be confined to a single thread, which is expected to be the main server game thread responsible for ticking the entity. State changes (via setState) and state queries (via isCharging) must be serialized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setState(type, state) | void | O(1) | Updates the pressed state for the given interaction type. This is the primary entry point for mutating the handler's state. |
| isCharging(...) | boolean | O(1) | Returns true if the specified interaction type is currently in a "down" state. Ignores all other parameters. |
| shouldCancelCharging(...) | boolean | O(1) | Always returns false. This default implementation never initiates a charge cancellation. |
| getChargeValue(...) | float | O(1) | Returns the raw time value passed into the method, representing a linear charge progression. |

## Integration Patterns

### Standard Usage
The InteractionSimulationHandler is designed to be held and managed by a higher-level interaction processing system. The processor updates its state based on network input and then polls it during the server tick to drive game logic.

```java
// Within a hypothetical InteractionProcessor class during a server tick

// Assume 'handler' is an instance of InteractionSimulationHandler
// Assume 'packet' is a decoded network packet with player input
handler.setState(packet.getInteractionType(), packet.isPressed());

// Later in the tick, the simulation queries the state
boolean charging = handler.isCharging(
    isFirstTick,
    chargeTime,
    InteractionType.PRIMARY,
    context,
    entityStoreRef,
    cooldownHandler
);

if (charging) {
    // Proceed with interaction logic...
}
```

### Anti-Patterns (Do NOT do this)
- **Multi-threaded Access:** Do not call setState from a network thread while the main game thread is calling isCharging. All interactions with a single instance must be synchronized or queued onto the main game thread.
- **Assuming Complex Logic:** Do not rely on this class to manage cooldowns or other complex mechanics. It is a simple state tracker; the CooldownHandler and other context objects passed into its methods are what a *more advanced* implementation would use. This implementation ignores them.

## Data Pipeline
The handler sits between the network layer and the core game simulation, translating raw input events into a queryable state for the game logic to consume on each tick.

> Flow:
> Player Input Packet -> Server Network Decoder -> **InteractionSimulationHandler.setState()** -> Server Tick -> Interaction Processor polls **isCharging()** -> World State Update

