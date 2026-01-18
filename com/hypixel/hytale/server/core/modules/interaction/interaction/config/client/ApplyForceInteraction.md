---
description: Architectural reference for ApplyForceInteraction
---

# ApplyForceInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Data-Driven Component

## Definition
```java
// Signature
public class ApplyForceInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The ApplyForceInteraction is a server-side, data-driven component that defines a single, atomic operation within the broader Interaction System: applying a physical force to an entity. It functions as a state within a larger, configurable state machine, where game behaviors like abilities, knockback, or environmental effects are composed of chained Interactions.

Its architecture is defined by three core principles:

1.  **Declarative Configuration:** Behavior is not hardcoded. Each ApplyForceInteraction is defined as a data asset, deserialized at runtime by a static **CODEC**. This allows designers to create complex physical behaviors—such as dashes, jumps, or knockbacks—by simply authoring data files. Properties like force direction, duration, and conditional transitions are all configured declaratively.

2.  **Separation of Configuration and State:** An instance of ApplyForceInteraction is a stateless template. All runtime state for an active interaction (e.g., elapsed time, completion status) is stored externally within an **InteractionContext** object, which is unique to each entity executing the interaction. This design ensures that a single ApplyForceInteraction asset can be safely reused by thousands of entities concurrently.

3.  **Hybrid Server-Client Authority:** This interaction employs a sophisticated synchronization model. The server executes a predictive simulation via **simulateTick0** to apply velocity changes and check for conditions like collisions or ground contact. Concurrently, it serializes its configuration into a network packet and sends it to the client. The server then waits for a response from the client via **tick0**, which processes the client's authoritative state (**ApplyForceState**). This hybrid model allows for responsive client-side prediction while letting the server make final decisions based on client feedback, which is critical for interactions that depend on client-detected states like landing on the ground.

The **compile** method translates the declarative configuration into a sequence of low-level operations and control flow jumps for the interaction engine, effectively building a small program from the asset data.

## Lifecycle & Ownership
-   **Creation:** ApplyForceInteraction instances are not created using the *new* keyword in game logic. They are instantiated by the engine's asset loading system during server startup or asset hot-reloading. The static **CODEC** field is used to deserialize the object from a data file (e.g., JSON). These instances are then stored in a central registry, accessible via an asset path.

-   **Scope:** The configured object is a long-lived, shared resource. It persists for the entire server session and is treated as a stateless blueprint. The *execution* of the interaction is transient and scoped to an **InteractionContext**, which is created when an entity begins the interaction and destroyed when it completes.

-   **Destruction:** The ApplyForceInteraction object is garbage collected only when its asset is unloaded by the server, typically during a full shutdown or a resource reload. The associated runtime state within the **InteractionContext** is cleaned up immediately upon the interaction's completion or cancellation.

## Internal State & Concurrency
-   **State:** The ApplyForceInteraction object is effectively **immutable** after its initial creation and deserialization. Its fields, such as **duration** and **waitForGround**, represent static configuration and are never modified during gameplay. All mutable runtime state is managed externally in the **InteractionContext**.

-   **Thread Safety:** The object is inherently **thread-safe** for read operations due to its immutable nature. Multiple threads can safely access its configuration properties. State modifications to game world components (e.g., an entity's **Velocity**) are not performed directly. Instead, they are submitted as instructions to a **CommandBuffer** within the **InteractionContext**. This command buffer pattern ensures that all state changes are deferred and executed in a controlled, thread-safe manner during the appropriate phase of the main game loop.

## API Surface
The public methods of this class are primarily callbacks for the interaction engine. They should not be invoked directly by game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick0(...) | void | O(1) | Engine callback. Processes authoritative state received from the client (**InteractionSyncData**). Determines if the interaction has finished based on client feedback and jumps to the next interaction. |
| simulateTick0(...) | void | O(N) | Engine callback. Runs the server-side simulation. Applies forces to the entity's **Velocity** component and checks for server-side completion conditions (timer, collision). N is the number of nearby entities for collision checks. |
| compile(...) | void | O(1) | Engine callback. Translates the interaction's configuration and its potential next steps (**groundInteraction**, **collisionInteraction**) into a list of operations and labels for the interaction execution engine. |
| generatePacket() | Interaction | O(1) | Engine callback. Creates the network packet to be sent to the client, synchronizing the interaction's parameters. |

## Integration Patterns

### Standard Usage
This component is not used directly in code. It is configured in a data file (e.g., a `.json` asset) and referenced by its asset path from other game systems, such as an item's ability definition or as the next step in another Interaction.

A conceptual data asset might look like this:

```json
{
  "type": "ApplyForceInteraction",
  "Force": 15.0,
  "Direction": [0, 1, 0],
  "AdjustVertical": true,
  "Duration": 0.2,
  "WaitForGround": true,
  "GroundNext": "asset:player_interactions/land_roll",
  "Next": "asset:player_interactions/ability_cooldown"
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call **new ApplyForceInteraction()**. The object must be created via the engine's asset pipeline to be correctly configured and registered.
-   **Direct Method Invocation:** Do not call **tick0**, **simulateTick0**, or **compile** from game logic. These are strictly for internal engine use. Triggering an interaction should be done through high-level APIs like an **InteractionModule**.
-   **State Modification:** Do not attempt to modify the fields of a shared ApplyForceInteraction instance at runtime. This will cause unpredictable behavior for all entities using that interaction and is not thread-safe.

## Data Pipeline
The execution of an ApplyForceInteraction follows a complex client-server data flow. The server simulates the action but relies on client feedback for final state transitions.

> **Flow: Server Simulation & Client Synchronization**
>
> 1.  **Server: Trigger** -> An entity's **InteractionContext** begins executing the **ApplyForceInteraction**.
> 2.  **Server: Packet Generation** -> **generatePacket** and **configurePacket** are called to create a `com.hypixel.hytale.protocol.ApplyForceInteraction` packet.
> 3.  **Server -> Client: Network Sync** -> The packet is sent to the client, containing all parameters (force, duration, conditions).
> 4.  **Server: Simulation** -> Simultaneously, **simulateTick0** runs on the server. It reads the entity's **HeadRotation**, calculates the force vector, and adds an instruction to the entity's **Velocity** component via a **CommandBuffer**.
> 5.  **Client: Prediction** -> The client receives the packet and immediately applies the force to its predicted entity, providing responsive feedback.
> 6.  **Client: State Check** -> The client continuously checks for conditions like ground contact or collision. When a condition is met, it sends an **InteractionSyncData** packet back to the server with the result (e.g., **ApplyForceState.Ground**).
> 7.  **Client -> Server: Network Sync** -> The client's state is sent to the server.
> 8.  **Server: State Processing** -> The server's next call to **tick0** receives this data. It reads the **ApplyForceState** from the client.
> 9.  **Server: Transition** -> Based on the client's state, **tick0** performs a `context.jump` to the appropriate next interaction (**groundInteraction**, **collisionInteraction**, or **next**), completing the data pipeline.

