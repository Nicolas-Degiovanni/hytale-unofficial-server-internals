---
description: Architectural reference for SeatingInteraction
---

# SeatingInteraction

**Package:** com.hypixel.hytale.builtin.mounts.interactions
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class SeatingInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The SeatingInteraction class is a server-authoritative, data-driven implementation for handling a player's attempt to sit on a block. It is not a persistent service but rather a stateless behavior definition that is loaded from asset configuration files.

Its primary architectural role is to act as a **mediator** between the generic server interaction system and the specialized BlockMountAPI. When a player interacts with a block configured to use this class, the server's interaction module dispatches the event to an instance of SeatingInteraction. This class then translates the generic interaction context into a specific call to the mounting system, handles the results (success, failure, already mounted), and queues appropriate feedback, such as sound effects or chat messages.

This class is designed to be configured entirely through the asset system, identified by its static **CODEC** field. This allows designers to create "sittable" blocks without writing any new Java code, simply by referencing this interaction handler in a block's definition file.

The complete absence of logic in the simulateInteractWithBlock method is a critical design choice. It signifies that this interaction is **purely server-authoritative**. There is no client-side prediction; the client only learns of the outcome after the server has processed the action and sent back entity state updates and sound events.

## Lifecycle & Ownership
- **Creation:** Instances are not created manually with the *new* keyword. They are instantiated by the server's asset loading pipeline during startup. The static CODEC field is used to deserialize a configuration from an asset file into a SeatingInteraction object. This object is then registered and associated with one or more block types.
- **Scope:** The configured instance persists for the entire server session, held within the interaction registry. It is effectively a singleton for a given interaction definition. The object itself is stateless, so the same instance is used to handle all interactions of its type.
- **Destruction:** The instance is garbage collected when the server shuts down and the asset registries are cleared.

## Internal State & Concurrency
- **State:** SeatingInteraction is **stateless and immutable**. It contains no instance fields and its behavior depends entirely on the arguments passed to its methods during an interaction event. This design makes it inherently reusable and predictable.
- **Thread Safety:** This class is thread-safe by design due to its stateless nature. However, it is intended to be executed exclusively on the main server thread for a given world. The methods operate on shared, mutable state (World, EntityStore) via a CommandBuffer. This pattern ensures that all state mutations are queued and executed deterministically at the end of the current server tick, preventing race conditions.

**WARNING:** Calling methods on this class from an asynchronous thread will lead to world state corruption. All interactions must be processed within the server's main game loop.

## API Surface
The public contract is defined by its parent class, SimpleBlockInteraction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(1) | Server-side execution logic. Attempts to mount the interacting player onto the target block using the BlockMountAPI. Queues sound events or messages via the CommandBuffer based on the result. |
| simulateInteractWithBlock(...) | void | O(1) | Client-side prediction logic. Intentionally empty, indicating the interaction is not predicted on the client. |
| CODEC | BuilderCodec | O(1) | Static field used by the asset system to deserialize this class from configuration files. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by gameplay programmers. It is configured within a block's asset definition file and triggered automatically by the server's interaction module when a player right-clicks the associated block.

The following conceptual example shows how the **interaction system** would invoke this handler.

```java
// Conceptual example from within the server's InteractionModule
// A player has interacted with a block.

// 1. The system retrieves the pre-configured interaction handler for the block.
SimpleBlockInteraction handler = blockType.getInteractionHandler();

// 2. The system verifies it's the correct type and invokes it.
if (handler instanceof SeatingInteraction) {
    handler.interactWithBlock(world, commandBuffer, type, context, itemInHand, targetBlock, cooldownHandler);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SeatingInteraction()`. The class is designed to be configured and loaded via the asset system. Manual instantiation bypasses this and will result in a non-functional object.
- **Assuming Client-Side State:** Do not write client-side logic that assumes the seating action will succeed. The empty `simulateInteractWithBlock` method guarantees that the client will not know the result until a state packet arrives from the server.
- **Stateful Extension:** Do not extend this class to add state (member variables). This violates the stateless design principle of interaction handlers and can lead to concurrency issues and unpredictable behavior when the same instance is reused for multiple players.

## Data Pipeline
The flow of data for a seating interaction is unidirectional from client input to client feedback, with all logic arbitrated by the server.

> Flow:
> Client Input (Right Click) -> C2S Interaction Packet -> Server Network Layer -> Interaction Module -> **SeatingInteraction.interactWithBlock** -> BlockMountAPI -> CommandBuffer (Entity Mount, Sound Event) -> Server Tick Processor -> S2C Entity Update & PlaySound Packets -> Client World & Audio Engine

