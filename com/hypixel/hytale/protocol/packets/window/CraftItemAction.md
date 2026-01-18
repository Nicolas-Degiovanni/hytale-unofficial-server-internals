---
description: Architectural reference for CraftItemAction
---

# CraftItemAction

**Package:** com.hypixel.hytale.protocol.packets.window
**Type:** Transient

## Definition
```java
// Signature
public class CraftItemAction extends WindowAction {
```

## Architecture & Concepts
The CraftItemAction class is a Data Transfer Object (DTO) representing a specific network packet. It signifies a user's intent to execute a crafting operation within a game window, such as a crafting table or forge. As a subclass of WindowAction, it operates within the context of an open user interface container on the server.

This packet's primary function is to act as a command from the client to the server. The client constructs and sends this packet, and the server receives and interprets it to trigger the corresponding server-side game logic.

**Warning:** The current implementation of this class is a structural placeholder. It contains no data fields and its serialization/deserialization methods are no-ops. This implies that the server infers the full context of the crafting action (e.g., the recipe, the ingredients, the output slot) from the player's session state, specifically the state of the window they currently have open. The packet itself acts as a simple, stateless trigger.

## Lifecycle & Ownership
- **Creation:**
    - On the **client**, an instance is created by the UI system in response to a direct player interaction, such as clicking a "craft" button.
    - On the **server**, an instance is created by the network protocol layer's packet deserializer when an incoming byte stream is identified as a CraftItemAction. The static `deserialize` factory method is the designated entry point for this process.

- **Scope:** Extremely short-lived. An instance of CraftItemAction exists only for the duration of a single transaction. On the client, it is discarded after being serialized into the network buffer. On the server, it is discarded after the corresponding game logic has been executed.

- **Destruction:** The object is not explicitly managed or destroyed. It becomes eligible for garbage collection immediately after its use case is fulfilled.

## Internal State & Concurrency
- **State:** Immutable and stateless. This class contains no instance fields. Its identity and behavior are defined entirely by its type.

- **Thread Safety:** This class is inherently thread-safe. All methods, including the static factories, operate only on their arguments and do not access or modify any shared or static state. It can be safely processed by any thread, which is critical for use within a multi-threaded network environment like Netty.

## API Surface
The public API is designed for interaction with the Hytale protocol engine. Direct invocation by game feature developers is discouraged.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static CraftItemAction | O(1) | Factory method to construct an instance from a network buffer. |
| serialize(ByteBuf) | int | O(1) | Writes the packet's representation to a network buffer. Currently a no-op. |
| computeSize() | int | O(1) | Calculates the number of bytes required to serialize the packet. Returns 0. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Verifies if the data in a buffer can form a valid packet. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in gameplay code. It is created and consumed automatically by the client's network manager and the server's network pipeline.

**Client-Side (Conceptual)**
```java
// Executed by the UI framework when a player clicks the craft button
void onCraftButtonClick() {
    CraftItemAction action = new CraftItemAction();
    // The network system handles serialization and transmission
    client.getNetworkManager().sendPacket(action);
}
```

**Server-Side (Conceptual)**
```java
// The packet handler is invoked by the Netty pipeline
public class PlayerPacketHandler {
    public void handle(CraftItemAction action, Player player) {
        // Server infers context from the player's currently open window
        Window openWindow = player.getOpenWindow();
        if (openWindow instanceof CraftingWindow) {
            ((CraftingWindow) openWindow).executeCraft();
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Adding State:** Do not modify this class to add fields. If crafting data (like a recipe ID) needs to be sent, a new, dedicated packet class must be designed and integrated into the protocol. Modifying this placeholder will break protocol compatibility.
- **Direct Serialization:** Never call `serialize` or `deserialize` manually. These methods are tightly coupled to the engine's network buffer management and protocol state machine. Circumventing the engine will lead to corrupted data streams and connection errors.

## Data Pipeline
The flow of this object is unidirectional from client to server. It is a command, not a state synchronization object, and thus does not typically flow from server to client.

> Flow:
> Client UI Event (Player Clicks Craft) -> **new CraftItemAction()** -> Protocol Serialization Engine -> Netty Channel -> Server Network Interface -> Netty Pipeline -> Protocol Deserialization Engine -> **CraftItemAction.deserialize()** -> Server Game Logic Handler -> Player State Mutation (Inventory Update)

