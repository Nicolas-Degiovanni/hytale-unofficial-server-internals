---
description: Architectural reference for BuilderToolInteraction
---

# BuilderToolInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Transient

## Definition
```java
// Signature
public class BuilderToolInteraction extends SimpleInteraction {
```

## Architecture & Concepts
BuilderToolInteraction is a server-side configuration class that defines the behavior for interactions involving client-authoritative "builder" tools. It is a specific implementation within the server's Interaction Module, designed for scenarios where the server must wait for precise input from the client before finalizing an action.

The key architectural pattern this class implements is a **Client-Authoritative Data Exchange**. Unlike purely server-driven interactions, this class instructs the server to pause its logic via the getWaitForDataFrom method, which returns WaitForDataFrom.Client. This signals the server to send an initial packet to the client, effectively delegating control for the interaction's data gathering phase. The client then performs its logic, such as presenting a UI or calculating a precise world position, and sends the result back. The server consumes this result in the tick0 method, synchronizing the client's state into the authoritative server context.

This model is essential for tools requiring responsive, real-time user feedback, where network latency would make a server-authoritative approach feel sluggish or imprecise.

## Lifecycle & Ownership
- **Creation:** Instances of BuilderToolInteraction are not created directly using the new keyword. They are deserialized from game data files by the server's configuration loading system at startup, using the provided static CODEC field. This class represents a reusable, data-driven behavior definition.
- **Scope:** An instance of this class is effectively a singleton for its specific interaction type. It persists for the entire lifetime of the server and is shared across all player sessions and threads that trigger the corresponding interaction.
- **Destruction:** The object is eligible for garbage collection only when the server shuts down or performs a full reload of its interaction configurations. There is no manual destruction or cleanup process.

## Internal State & Concurrency
- **State:** BuilderToolInteraction is **stateless and immutable**. It contains no instance fields and its behavior is entirely determined by its method implementations. All mutable state required for an interaction is passed into its methods via the InteractionContext parameter.
- **Thread Safety:** This class is inherently **thread-safe**. Because it is stateless, a single shared instance can be safely used to process interactions for multiple players concurrently on different server threads without risk of race conditions or data corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generatePacket() | Interaction | O(1) | Creates the protocol packet sent to the client to initiate the builder tool UI and logic. |
| needsRemoteSync() | boolean | O(1) | Returns true, signaling to the engine that this interaction's state must be synchronized. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | **CRITICAL:** Returns WaitForDataFrom.Client, defining the client-authoritative data flow for this interaction. |
| tick0(...) | void | O(1) | Executes the core server-side logic, primarily synchronizing the client-provided state into the server's InteractionContext. |

## Integration Patterns

### Standard Usage
A developer does not instantiate or call methods on this class directly. Instead, it is defined in a data file (e.g., JSON) associated with an in-game item or ability. The server's InteractionModule loads this configuration and orchestrates its execution when a player triggers the action.

The following conceptual example illustrates how the *engine* would use a loaded instance.

```java
// Engine-level code within the InteractionModule

// 1. An interaction is triggered for a player.
// 2. The engine retrieves the pre-loaded configuration for that interaction.
BuilderToolInteraction interactionLogic = interactionConfigManager.get("hytale:stone_hammer_build_mode");
InteractionContext context = ...; // The context for the specific player and world state.

// 3. The engine's state machine determines it must wait for client data.
//    It calls generatePacket() and sends it to the client.

// 4. Later, after receiving a data packet from the client...
//    The engine populates context.getClientState() with the received data.

// 5. The engine calls tick0 to finalize the server-side state change.
interactionLogic.tick0(false, 0.016f, InteractionType.PRIMARY, context, cooldownHandler);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BuilderToolInteraction()`. The server relies on the static CODEC for deserialization and management. Bypassing this will lead to an unmanaged object that is not integrated with the game's configuration system.
- **Stateful Subclassing:** Do not extend this class to add mutable instance fields. This violates its stateless design, breaks thread safety, and will cause unpredictable behavior when the single instance is used by multiple players. All state must be managed within the InteractionContext.

## Data Pipeline
The data flow for this interaction is a well-defined round-trip, initiated by the server but fulfilled by the client.

> Flow:
> Server receives initial interact event -> InteractionModule retrieves **BuilderToolInteraction** config -> Server sends `BuilderToolInteraction` packet to Client -> Client displays UI and gathers input -> Client sends data response packet to Server -> Server receives data -> **BuilderToolInteraction.tick0** synchronizes data to server state -> World state is updated.

