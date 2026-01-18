---
description: Architectural reference for UseNPCInteraction
---

# UseNPCInteraction

**Package:** com.hypixel.hytale.server.npc.interactions
**Type:** Transient Command Object

## Definition
```java
// Signature
public class UseNPCInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts

The UseNPCInteraction class is a concrete implementation of the Command design pattern, representing the specific action of a player attempting to interact with an NPC. It is not a long-lived service but a stateless, data-driven object that encapsulates a single, atomic operation within the server's interaction system.

Architecturally, this class serves as a crucial validation and dispatch layer. It is instantiated by the server's configuration loader via its static CODEC, meaning its existence and properties are defined in game data files, not hardcoded. This allows for flexible and data-driven NPC behavior.

Its primary responsibility is to perform a series of preconditions and state checks before delegating the actual interaction logic to the target NPC's internal state machine. It inherits from SimpleInstantInteraction, signifying that it is a non-persistent action that is expected to resolve completely within a single server tick. It does not manage its own state across multiple ticks.

## Lifecycle & Ownership

-   **Creation:** Instances are not created at runtime per-event. They are deserialized and instantiated once by the server's configuration and asset loading system during server startup or a configuration reload, using the provided static CODEC. The resulting object is a shared, reusable definition of the interaction.
-   **Scope:** The object's lifetime is tied to the server's loaded configuration. It persists as a stateless, singleton-like definition within an interaction registry for the entire server session.
-   **Destruction:** The object is garbage collected when the server shuts down or when the interaction configuration is reloaded, at which point the old registry is discarded.

## Internal State & Concurrency

-   **State:** **Immutable.** UseNPCInteraction is fundamentally stateless. It contains no mutable fields and all necessary context for its operation is provided via the InteractionContext parameter to its methods. Any state changes are performed on external systems like the CommandBuffer or the target NPCEntity.

-   **Thread Safety:** **Not thread-safe.** This class is designed to be executed exclusively by the server's main game loop thread. Its methods operate on shared, mutable state (e.g., CommandBuffer, Blackboard) without internal locking. The engine must guarantee that all interactions are processed sequentially within a given tick to prevent data corruption. The check against ReservationStatus is an explicit, application-level concurrency control mechanism to handle logical conflicts, such as two players trying to use the same NPC simultaneously.

## API Surface

The public contract is almost entirely defined by its superclass. The core logic is contained within the overridden firstRun method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | Executes the interaction. Validates that the initiator is a Player and the target is a valid, non-busy NPCEntity. On success, it delegates control to the NPC's role state. On failure, it updates the context state to Failed and may send a feedback message to the player. |

## Integration Patterns

### Standard Usage

This class is not intended to be invoked directly by most game logic or modding code. It is defined in data files and executed by the server's core InteractionModule in response to player actions. The engine resolves the appropriate interaction from its registry and executes it.

```java
// Conceptual engine-level code that executes the interaction.
// This is handled by the server's InteractionModule.

InteractionContext context = createInteractionContextFor(player, targetNPC);
RootInteraction rootConfig = getInteractionConfigFor(targetNPC); // e.g., UseNPCInteraction.DEFAULT_ROOT

// The registry contains the pre-loaded UseNPCInteraction instance
SimpleInstantInteraction interaction = interactionRegistry.get(rootConfig.getInteractionId());

// The engine invokes the command
interaction.firstRun(InteractionType.PRIMARY, context, cooldownHandler);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new UseNPCInteraction()`. The interaction system relies on instances loaded from configuration via the CODEC. Direct instantiation bypasses the registry and will likely fail.
-   **Asynchronous Execution:** Never invoke the firstRun method from an asynchronous task or a different thread. It must be called from within the server's main tick to ensure safe access to the CommandBuffer and other game state.
-   **Stateful Subclassing:** Avoid extending this class to add mutable state. The interaction system is designed around stateless, reusable command objects. Adding state can introduce severe concurrency issues and memory leaks.

## Data Pipeline

The UseNPCInteraction acts as a specific stage in a larger data processing flow that begins with player input and ends with a change in the NPC's state.

> Flow:
> Player Input Packet -> Server Network Handler -> InteractionModule (resolves target and action) -> **UseNPCInteraction.firstRun()** -> Blackboard (Reservation Check) -> NPCEntity.Role.StateSupport.addInteraction() -> NPC State Machine Update

