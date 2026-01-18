---
description: Architectural reference for NPCInteractionSimulationHandler
---

# NPCInteractionSimulationHandler

**Package:** com.hypixel.hytale.server.npc.interactions
**Type:** Stateful Component

## Definition
```java
// Signature
public class NPCInteractionSimulationHandler implements IInteractionSimulationHandler {
```

## Architecture & Concepts
The NPCInteractionSimulationHandler is a server-side component that implements the IInteractionSimulationHandler interface. It provides a concrete, stateful strategy for managing time-based "charge-up" mechanics for entity interactions.

Architecturally, this class serves as a pluggable behavior module for the core interaction system. Instead of embedding complex timing logic directly into every NPC or behavior, the system delegates the responsibility of tracking charge state to an instance of this handler. Its primary function is to receive a target charge time and then report back to the simulation loop whether the interaction is still in its charging phase based on the elapsed time.

This design decouples the high-level NPC logic (which decides *that* an interaction should have a charge time) from the low-level interaction simulation (which ticks every frame and needs to query the charge state).

### Lifecycle & Ownership
- **Creation:** This is a Plain Old Java Object (POJO) and is not managed by a dependency injection framework. It is intended to be instantiated directly by higher-level game logic, such as an NPC Behavior Tree node or a server-side script, at the beginning of an interaction that requires a charge-up period.
- **Scope:** The lifetime of an NPCInteractionSimulationHandler instance is ephemeral and should be strictly tied to a single interaction instance. It holds the state for one specific charge-up event.
- **Destruction:** The object is not explicitly destroyed. It is cleaned up by the Java garbage collector once the owning system (e.g., the Behavior Tree node) releases its reference, which typically occurs when the interaction completes, is cancelled, or fails.

## Internal State & Concurrency
- **State:** This component is **mutable**. It contains a single floating-point field, requestedChargeTime, which represents the total duration required for the interaction to complete its charge. This state is set once via the requestChargeTime method.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms. All methods, particularly the state-mutating requestChargeTime and the state-reading isCharging, must be called from the same thread. In practice, this must be the main server thread responsible for ticking the world and its entities.

**WARNING:** Accessing an instance of this handler from multiple threads without external locking will lead to race conditions and undefined behavior.

## API Surface
The public API is defined by the IInteractionSimulationHandler interface, but this implementation only provides meaningful logic for a subset of the contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| requestChargeTime(float chargeTime) | void | O(1) | Sets the total required charge time. This is the primary configuration method. |
| isCharging(...) | boolean | O(1) | Returns true if the elapsed time is less than the requested charge time. Polled by the simulation. |
| getChargeValue(...) | float | O(1) | Returns the configured requestedChargeTime. |
| setState(...) | void | O(1) | No-op. This implementation does not track binary on/off states. |
| shouldCancelCharging(...) | boolean | O(1) | Always returns false. This implementation provides no cancellation logic. |

## Integration Patterns

### Standard Usage
The handler is created and configured by NPC-specific logic, then passed to the generic interaction system which polls it during the simulation tick.

```java
// In an NPC Behavior or Script:

// 1. An interaction is initiated.
// 2. A handler is created for this specific interaction.
NPCInteractionSimulationHandler chargeHandler = new NPCInteractionSimulationHandler();

// 3. The charge duration is configured.
chargeHandler.requestChargeTime(2.5f); // Request a 2.5 second charge time

// 4. The handler is passed to the core system to manage the interaction.
player.getInteractionModule().startInteraction(targetNPC, InteractionType.USE, chargeHandler);
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not reuse a single handler instance for multiple, concurrent interactions. Each interaction must have its own dedicated handler to prevent state corruption.
- **Direct State Polling:** Do not call isCharging from your own game logic. The core interaction system is responsible for ticking the interaction and calling this method; relying on it elsewhere can lead to desynchronization.
- **Multi-threaded Access:** Do not create the handler on one thread and pass it to a simulation running on another thread without explicit and robust external synchronization.

## Data Pipeline
The flow of control and state is unidirectional. The NPC logic configures the state, and the core simulation reads the state to make decisions.

> Flow:
> NPC Behavior Script -> `requestChargeTime(2.5f)` -> **NPCInteractionSimulationHandler** (State: requestedChargeTime = 2.5) -> Core Interaction System polls `isCharging(time)` -> Simulation Result (Continue Charging / Complete)

