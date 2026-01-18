---
description: Architectural reference for BlockWindow
---

# BlockWindow

**Package:** com.hypixel.hytale.server.core.entity.entities.player.windows
**Type:** Transient State Object (Base Class)

## Definition
```java
// Signature
public abstract class BlockWindow extends Window implements ValidatedWindow {
```

## Architecture & Concepts
The BlockWindow is an abstract, server-side component that represents a player's stateful interaction with a specific block in the world. It serves as a foundational element for any UI or game system that is tied to a block's location and type, such as crafting tables, chests, or furnaces.

Its primary architectural role is to act as a server-authoritative validator. When a client requests to interact with a block, the server instantiates a concrete implementation of BlockWindow. This object encapsulates the *expected state* of the world at the time of the request. The core responsibility of this class, through its implementation of the ValidatedWindow interface, is to verify that the player's interaction is legitimate by checking against the *current* world state.

This validation prevents a wide class of exploits and race conditions, including:
*   Interacting with objects from an impossible distance.
*   Interacting with a block that has been destroyed or replaced since the client sent the request.
*   Attempting to interact with a block in an unloaded or non-existent chunk.

It forms a critical bridge between the network protocol layer, which receives player input, and the world simulation layer, which manages game state.

## Lifecycle & Ownership
- **Creation:** A concrete subclass of BlockWindow is instantiated by a server-side packet handler in response to a specific client request to interact with a block. The constructor is populated with coordinates and a block type derived from the incoming network packet.
- **Scope:** The object is short-lived and transaction-scoped. It exists only for the duration of a single player interaction validation sequence. It is owned by the player's session, typically managed by a higher-level WindowManager.
- **Destruction:** The instance is eligible for garbage collection as soon as the interaction is either completed or rejected, and the player's WindowManager closes or replaces it. There is no persistent state held within this object beyond the immediate transaction.

## Internal State & Concurrency
- **State:** The object is stateful but largely immutable after creation. The core state consists of the block coordinates (x, y, z) and the expected BlockType. While the maximum interaction distance can be mutated via setMaxDistance, the positional and type data are fixed upon instantiation. The object's purpose is to be a snapshot of an *intended* interaction.

- **Thread Safety:** **WARNING:** This class is not thread-safe and must not be shared across threads. It is designed to be created, used, and discarded within the context of a single server tick or a single network event processing thread associated with a specific player. Its methods, particularly validate, read from underlying world and entity systems (World, EntityStore) which have their own strict concurrency models. Unsynchronized, multi-threaded access will lead to severe data corruption and server instability.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| validate() | boolean | O(1) | Executes the core validation logic. Returns true if the interaction is valid according to server rules. |
| getX() | int | O(1) | Returns the target block's X coordinate. |
| getY() | int | O(1) | Returns the target block's Y coordinate. |
| getZ() | int | O(1) | Returns the target block's Z coordinate. |
| getBlockType() | BlockType | O(1) | Returns the BlockType the window was created to interact with. |
| setMaxDistance(double) | void | O(1) | Overrides the default maximum interaction distance for this specific window instance. |

## Integration Patterns

### Standard Usage
The intended use is within a network packet handler. The handler extracts interaction data from a packet, creates the appropriate BlockWindow subclass, and uses it to validate the request before modifying game state.

```java
// Example from a hypothetical PacketOpenContainerHandler
BlockWindow window = new ContainerWindow(packet.getX(), packet.getY(), packet.getZ(), ...);
player.getWindowManager().setActiveWindow(window);

// The validate() method is the gatekeeper for all subsequent logic
if (!window.validate()) {
    player.disconnect("Invalid interaction attempt");
    return;
}

// Proceed with opening the container...
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not hold a reference to a BlockWindow instance beyond a single interaction. Its validation is only meaningful at the moment it is checked.
- **Cross-Player Usage:** Never use a BlockWindow created for one player to validate an interaction for another. The internal validation logic is tied to the player reference obtained via getPlayerRef.
- **Client-Side Trust:** Do not create a BlockWindow with a BlockType that the client claims exists without first validating it against server-side asset registries. The BlockType passed to the constructor should be resolved on the server.

## Data Pipeline
The BlockWindow acts as a validation stage in the data flow of a player-world interaction. It ensures data integrity before the server commits to a state change.

> Flow:
> Client Interaction Packet -> Server Packet Handler -> **BlockWindow** Instantiation -> **BlockWindow**.validate() -> World State Mutation -> Server Response Packet<ctrl63>

