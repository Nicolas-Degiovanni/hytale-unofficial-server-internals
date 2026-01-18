---
description: Architectural reference for SortItemsAction
---

# SortItemsAction

**Package:** com.hypixel.hytale.protocol.packets.window
**Type:** Packet Model

## Definition
```java
// Signature
public class SortItemsAction extends WindowAction {
```

## Architecture & Concepts
The SortItemsAction class is a specialized Data Transfer Object (DTO) that represents a discrete, user-initiated command within the client-server protocol. It inherits from WindowAction, indicating its role is to modify the state of a game UI window, such as an inventory or a storage container.

This class is not a service or a manager; it is a pure data container. Its primary function is to encapsulate the type of sorting a player wishes to perform. The design is heavily optimized for network performance and minimal overhead. This is evident through its direct serialization and deserialization logic that interacts with Netty's ByteBuf, and its compile-time size constants (e.g., FIXED_BLOCK_SIZE).

SortItemsAction acts as a message, flowing from the client's input handler, through the network layer, to the server's game logic for processing. It is a fundamental building block of the interactive inventory system.

## Lifecycle & Ownership
- **Creation:** An instance is created on the client when a player interacts with a sorting UI element. On the server, a new instance is created by the protocol's deserialization layer when the corresponding network packet is received.
- **Scope:** Extremely short-lived and transient. The object exists only for the duration of its serialization, network transit, and subsequent processing by a handler. It is a "fire-and-forget" data structure.
- **Destruction:** The object becomes eligible for garbage collection immediately after its data has been consumed by the relevant system (e.g., the server-side inventory manager). No system maintains a long-term reference to it.

## Internal State & Concurrency
- **State:** The class holds a single piece of mutable state: the public field *sortType*. While technically mutable, instances are intended to be treated as immutable after creation. The state is minimal, containing only the sorting criteria.
- **Thread Safety:** **This class is not thread-safe.** Its public, non-final field makes it unsafe for concurrent access. This is by design, as packet models are expected to be processed sequentially by a single network thread or game loop thread.

> **Warning:** Never share an instance of SortItemsAction across multiple threads without explicit, external synchronization. Doing so can lead to race conditions where the sortType is modified during serialization.

## API Surface
The public API is dominated by low-level serialization and validation methods intended for the core protocol engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf buf) | int | O(1) | Encodes the object's state into a network buffer. |
| deserialize(ByteBuf buf, int offset) | SortItemsAction | O(1) | Static factory method to construct an instance from a network buffer. |
| computeSize() | int | O(1) | Returns the constant size (1 byte) of the serialized data. |
| validateStructure(ByteBuf buffer, int offset) | ValidationResult | O(1) | Performs a pre-deserialization check on a buffer to prevent read errors. |

## Integration Patterns

### Standard Usage
Application code should never call serialization methods directly. Instead, it should create an instance and pass it to a higher-level system, such as a network manager or packet dispatcher.

```java
// Client-side: A UI handler responds to a button click
SortItemsAction sortByNameAction = new SortItemsAction(SortType.Name);

// The action is wrapped in a generic window packet and sent
WindowActionPacket packet = new WindowActionPacket(activeWindowId, sortByNameAction);
clientNetworkManager.sendPacket(packet);

// Server-side: A packet handler processes the incoming action
public void onWindowAction(Player player, WindowActionPacket packet) {
    if (packet.getAction() instanceof SortItemsAction sortAction) {
        player.getInventory().sort(sortAction.sortType);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and re-send the same SortItemsAction instance. This is not safe and breaks the assumption of transient, single-use packet objects. Always create a new instance for each distinct action.
- **Direct Deserialization:** Avoid calling the static deserialize method in application-level code. This is the responsibility of the protocol layer, which handles buffer management and error checking.
- **Long-Term Storage:** This object is not designed for persistence. Storing it in a cache or game state is incorrect, as it represents a point-in-time command, not a persistent state.

## Data Pipeline
The data flow for this action is unidirectional from client to server.

> Flow:
> Client UI Event (Sort Button Click) -> **SortItemsAction** (Instantiation) -> Protocol Serializer -> Network Stack (Client) -> Network Stack (Server) -> Protocol Deserializer -> **SortItemsAction** (Re-instantiation) -> Server Game Logic (Inventory System)

