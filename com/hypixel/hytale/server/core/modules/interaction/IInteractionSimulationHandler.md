---
description: Architectural reference for IInteractionSimulationHandler
---

# IInteractionSimulationHandler

**Package:** com.hypixel.hytale.server.core.modules.interaction
**Type:** Contract Interface

## Definition
```java
// Signature
public interface IInteractionSimulationHandler {
```

## Architecture & Concepts
The IInteractionSimulationHandler interface defines a strict contract for server-side simulation of continuous or charged player interactions. It serves as a strategic abstraction layer within the server's core interaction module, decoupling the central interaction processing loop from the specific state logic of any given interaction, such as charging a bow, casting a spell, or using a tool.

This component is central to validating and processing player actions that occur over a duration rather than instantaneously. Implementations of this interface are responsible for managing the state machine of a single, specific type of interaction (e.g., PrimaryAttack, SecondaryUse). The core game loop queries these handlers to determine the current state, progression, and validity of an ongoing player action, ensuring that game mechanics are applied consistently and authoritatively on the server.

## Lifecycle & Ownership
As an interface, IInteractionSimulationHandler has no lifecycle itself. The lifecycle described here pertains to its concrete implementations.

- **Creation:** Implementations are typically instantiated by a higher-level manager, such as an InteractionModule or an EntityComponent factory, when an entity that can perform the interaction is created or equipped with a relevant item. They are not intended for direct, ad-hoc instantiation.
- **Scope:** The lifetime of a handler instance is tightly coupled to the entity and the specific interaction it manages. It may persist for the life of the entity or be created and destroyed as items are equipped and unequipped.
- **Destruction:** An instance is eligible for garbage collection when the owning entity is destroyed or the associated interaction capability is removed. There is no explicit public destruction method.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, any non-trivial implementation is expected to be highly stateful. It must maintain internal state variables to track the progress of an interaction, such as the start time, charge level, and active status.
- **Thread Safety:** Implementations are **not thread-safe** and are designed to be operated exclusively by the main server thread responsible for ticking the entity. All methods on this interface must be called from within the context of the world's update loop.

**WARNING:** Accessing a handler instance from a network thread, worker thread, or any context other than the owning entity's designated update thread will lead to race conditions, inconsistent state, and server instability. Synchronization is not implemented and is not the responsibility of the handler.

## API Surface
The API provides a query-based contract for the server's game loop to poll the status of an ongoing interaction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setState(type, active) | void | O(1) | Directly sets the active state of a specific interaction type. This is the primary entry point for initiating or terminating an interaction from a controlling system. |
| isCharging(...) | boolean | O(1) | Determines if the interaction is currently in a "charging" or "active" phase. This is the main predicate used by the game loop each tick. |
| shouldCancelCharging(...) | boolean | O(1) | Evaluates conditions to determine if an ongoing charge should be prematurely cancelled (e.g., player moved, was interrupted). |
| getChargeValue(...) | float | O(1) | Calculates and returns the current progress of the charge, typically a normalized value between 0.0 and 1.0. Returns a non-meaningful value if not currently charging. |

## Integration Patterns

### Standard Usage
The handler is not used directly by most game logic. Instead, it is managed by the server's core InteractionModule. This module receives player input packets, selects the appropriate handler for the player's current context (e.g., held item), and drives its state forward during the server tick.

```java
// Simplified conceptual example from a hypothetical InteractionModule
// Assume 'handler' is the correct, stateful implementation for the current player action.

// On player input to start an action:
handler.setState(InteractionType.PRIMARY_ATTACK, true);

// During the server's per-player tick:
if (handler.isCharging(...)) {
    if (handler.shouldCancelCharging(...)) {
        handler.setState(InteractionType.PRIMARY_ATTACK, false);
        // ... trigger cancellation logic
    } else {
        float charge = handler.getChargeValue(...);
        // ... update entity state, send charge feedback to client
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Mismanagement:** Do not assume a handler is in a valid state. Always query with isCharging before attempting to get a charge value or check for cancellation.
- **Cross-Thread Invocation:** Never call methods on a handler from outside the main server tick for the associated entity. This will break game state.
- **External State Modification:** Do not attempt to manage the charge state from outside the handler. The implementation should be the sole authority on its internal timer and state progression. Use setState to initiate and terminate, but let the handler's internal logic manage the rest.

## Data Pipeline
This interface acts as a stateful processor within the server-side player action data pipeline.

> Flow:
> Client Input Packet -> Network Layer -> Player Action System -> **IInteractionSimulationHandler (Instance)** -> Entity State Update -> World Simulation Tick -> State Replication to Clients

