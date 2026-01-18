---
description: Architectural reference for DoorInteraction
---

# DoorInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Configuration Object

## Definition
```java
// Signature
public class DoorInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The DoorInteraction class is a server-side, data-driven component responsible for the complete logic of opening and closing single and double doors. It operates within the engine's broader Interaction System, which maps player actions to specific world behaviors.

This class is not a persistent entity in the world but rather a stateless configuration object loaded from game assets. When a player interacts with a block configured to use this interaction type, the engine retrieves the corresponding DoorInteraction instance and executes its logic.

Its core responsibilities include:
- **State Calculation:** Determining the door's next state (e.g., OPENED_IN, OPENED_OUT, CLOSED) based on its current state and the interacting entity's position.
- **Collision & Obstruction Detection:** Verifying that the space the door will occupy is clear before allowing it to open. If the path is blocked, the interaction fails.
- **Double Door Coordination:** Detecting an adjacent, matching door and orchestrating a synchronized state change for both blocks.
- **World State Modification:** Applying the calculated state changes to the world by updating the target block's type and interaction state. This is achieved by queuing commands into a CommandBuffer, ensuring transactional integrity within a single game tick.
- **Feedback Generation:** Triggering sound events at the door's location to provide auditory feedback to the player.

DoorInteraction is a prime example of Hytale's component-based, data-driven design. The complex logic is encapsulated away from the BlockType definition, allowing it to be reused for any block that should behave like a door.

## Lifecycle & Ownership
- **Creation:** Instances of DoorInteraction are not created manually using the *new* keyword. They are deserialized and instantiated by the engine's asset loading system via the provided static CODEC field. This typically occurs once at server startup when game assets are loaded into memory.
- **Scope:** A loaded DoorInteraction instance is effectively a singleton for its specific asset definition. It is stored in a central asset registry (e.g., Interaction.getAssetMap) and persists for the entire server session. The same instance is reused for every interaction with any block of the corresponding type.
- **Destruction:** The object is garbage collected when the server shuts down and the asset registries are cleared.

## Internal State & Concurrency
- **State:** The class contains a single configuration field, *horizontal*, which is set upon deserialization from the asset file. This state is **immutable** after initialization. The methods themselves are stateless, operating exclusively on parameters passed during invocation (e.g., World, CommandBuffer, InteractionContext).
- **Thread Safety:** **This class is not thread-safe.** Its methods, particularly interactWithBlock, read and write directly to world state (via ChunkAccessor and CommandBuffer). All invocations **must** occur on the main server thread that owns the game world to prevent race conditions, chunk corruption, and other critical concurrency failures.

## API Surface
The public contract is defined by its parent class, SimpleBlockInteraction. The primary entry point is the overridden method `interactWithBlock`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | protected void | O(N) | Executes the core door logic. Complexity is dependent on the size of the door's bounding box (N) due to obstruction checks. Modifies world state via the provided CommandBuffer. |
| simulateInteractWithBlock(...) | protected void | O(1) | A no-op on the server. Intended for client-side prediction, which is not implemented for this interaction. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly in code. Instead, they define it within a block's asset file, associating it with the "Use" interaction type. The engine handles the invocation.

The following conceptual example illustrates how the Interaction System would trigger this logic:

```java
// Conceptual engine code that invokes an interaction
// This code would exist within the server's core interaction module.

Interaction interaction = Interaction.getAssetMap().getAsset(interactionId);

if (interaction instanceof DoorInteraction doorInteraction) {
    // The engine invokes the method with the current world context
    doorInteraction.interactWithBlock(
        world,
        commandBuffer,
        interactionType,
        interactionContext,
        itemInHand,
        targetBlock,
        cooldownHandler
    );
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new DoorInteraction()`. The object's lifecycle is managed entirely by the asset system. Attempting to create it manually will result in an unconfigured and non-functional object.
- **Asynchronous Invocation:** Never call `interactWithBlock` from a separate thread. Accessing or modifying the World or CommandBuffer outside the main server tick is strictly forbidden and will lead to server instability and data corruption.
- **State Mutation:** Do not attempt to modify the `horizontal` field via reflection after the object has been loaded. This violates the data-driven contract and can lead to unpredictable behavior.

## Data Pipeline
The flow of data and control for a door interaction event follows a clear, server-authoritative path.

> Flow:
> Player Input (Client) -> Network Packet -> Server Interaction Module -> **DoorInteraction.interactWithBlock** -> CommandBuffer -> World State Change -> Network Sync -> Client Visual Update

