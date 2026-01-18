---
description: Architectural reference for BreakBlockInteraction
---

# BreakBlockInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Configuration Object

## Definition
```java
// Signature
public class BreakBlockInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The BreakBlockInteraction class is a server-side, data-driven configuration object that defines the logic for destroying or harvesting a block. It functions as a specific *strategy* within the broader Interaction system. It is not a service or a manager, but rather a template of behavior that is loaded from game asset files.

Its primary role is to encapsulate all server-side rules for a block-breaking action, including:
*   Distinguishing between gradual breaking (like mining stone) and instant harvesting (like picking a flower).
*   Enforcing tool requirements and game mode rules (e.g., creative mode versus survival mode).
*   Validating against world-level gameplay settings, such as whether block breaking is permitted at all.

The static CODEC field is the cornerstone of its design. The engine's asset loading system uses this codec to deserialize a configuration file (e.g., a JSON file) into a fully-formed BreakBlockInteraction instance. This allows game designers to define and modify complex block interaction behaviors without changing Java code.

This class operates on the world through a CommandBuffer, ensuring that all world modifications are queued and executed safely within the server's tick lifecycle.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly using the constructor. They are instantiated exclusively by the server's asset loading system via the static BuilderCodec named CODEC. This process occurs during server bootstrap or when new game content is loaded.
- **Scope:** An instance of BreakBlockInteraction persists for the entire server session. It is loaded once and serves as an immutable template for all interactions of its configured type.
- **Destruction:** The object is eligible for garbage collection only when the server shuts down or when its corresponding asset configuration is unloaded.

## Internal State & Concurrency
- **State:** The internal state of this class (harvest, toolId, matchTool) is defined in an asset file and is set upon creation. After initialization, this state should be considered **immutable**. The methods do not modify this internal state during gameplay; they use it as read-only configuration to process an interaction.
- **Thread Safety:** This class is **not thread-safe** if its methods are invoked from multiple threads concurrently. However, it is designed to be used exclusively within the server's main game loop thread. All world-mutating operations are delegated to a CommandBuffer, which is a standard engine pattern for deferring writes to a safe point in the game tick, thus preventing race conditions on world data.

## API Surface
The public contract is defined by methods overridden from its parent, SimpleBlockInteraction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick0(...) | void | O(1) | Called each tick while an interaction is ongoing. Updates synchronization data for the client. |
| interactWithBlock(...) | void | O(C) | Executes the core break/harvest logic. Complexity depends on chunk lookups. Throws exceptions if the context is invalid. |
| generatePacket() | Interaction | O(1) | Creates a network packet representing this interaction type. |
| configurePacket(Interaction) | void | O(1) | Populates a given network packet with the specific state of this interaction (e.g., the harvest flag). |

**WARNING:** The internal logic of the `interactWithBlock` method is complex and was not fully recoverable by the decompiler. Analysis of the bytecode indicates it performs the following sequence:
1.  Retrieves the Player component from the InteractionContext.
2.  Validates that the interacting entity is a Player.
3.  Retrieves the target WorldChunk and BlockChunk from the World.
4.  Checks global gameplay rules from GameplayConfig (e.g., isBlockBreakingAllowed).
5.  If `harvest` is true, it calls into BlockHarvestUtils to perform an instant pickup.
6.  If `harvest` is false, it checks the Player's GameMode. In creative modes, it performs an instant break via BlockHarvestUtils. In survival modes, it applies damage via BlockHarvestUtils, which may or may not result in the block breaking immediately.

## Integration Patterns

### Standard Usage
A developer or game designer does not interact with this class directly in Java code. Instead, they define its behavior in an asset file, which is then loaded by the engine. The InteractionModule is responsible for invoking the methods on the configured instance when a player initiates the action.

The conceptual flow within the engine is as follows:
```java
// Engine-level code, not for typical user implementation
InteractionContext context = createInteractionContextForPlayer(player, targetBlock);
InteractionType config = findInteractionConfigForAction("break_block"); // This would return a BreakBlockInteraction instance

// The engine would then drive the interaction over multiple ticks
if (config instanceof BreakBlockInteraction) {
    // On the first tick, the engine calls the core logic
    ((BreakBlockInteraction) config).interactWithBlock(world, commandBuffer, ..., context, ...);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BreakBlockInteraction()`. This bypasses the asset system and results in an unconfigured object that will fail at runtime. All instances must be created via the CODEC.
- **State Mutation:** Do not attempt to modify the `harvest` or `toolId` fields after the object has been loaded. These are intended to be read-only configuration values.
- **External Invocation:** Do not call `interactWithBlock` from outside the server's core InteractionModule. Doing so without a valid, engine-managed CommandBuffer and InteractionContext will break world state consistency and cause severe bugs.

## Data Pipeline
The flow of data for a block breaking event is managed by the server and involves several systems. This class is a key processing step in that pipeline.

> Flow:
> Client Input Packet -> Server Network Layer -> InteractionModule -> **BreakBlockInteraction** -> CommandBuffer -> World State Update -> Server Network Layer -> Client State Update Packet

