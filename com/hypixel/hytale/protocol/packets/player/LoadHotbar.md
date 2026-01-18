---
description: Architectural reference for LoadHotbar
---

# LoadHotbar

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class LoadHotbar implements Packet {
```

## Architecture & Concepts
The LoadHotbar class is a network packet definition, not a persistent game object. It serves as a simple, high-performance Data Transfer Object representing a server-to-client command. Its sole purpose is to instruct the client to change which of the player's inventory rows is currently active as the main hotbar.

As an implementation of the Packet interface, this class is a fundamental component of the Hytale network protocol layer. It contains both the data payload (the inventory row index) and the metadata required for the network engine to process it. The static fields, such as PACKET_ID and FIXED_BLOCK_SIZE, act as a schema that allows the protocol's packet dispatcher to identify, validate, and deserialize the raw byte stream from a Netty ByteBuf into a structured LoadHotbar object.

This class is designed for minimal overhead. Its public field and static codec methods (serialize, deserialize) bypass typical encapsulation patterns to achieve maximum performance within the tight constraints of the game's network event loop.

## Lifecycle & Ownership
-   **Creation:**
    -   **Server-Side:** Instantiated by a server-side game system when a player's active hotbar needs to change. For example, when a player enters a vehicle or a specific game state is triggered. The server's network layer then serializes this object for transmission.
    -   **Client-Side:** Instantiated by the client's network protocol pipeline. A central packet factory uses the static `deserialize` method to construct the object from an incoming network buffer when a packet with ID 106 is received.

-   **Scope:** Extremely short-lived and transient. A LoadHotbar object exists only for the brief moment it is being serialized for sending or after being deserialized for processing. It is a message, not a state container.

-   **Destruction:** The object becomes eligible for garbage collection immediately after its data has been consumed by a client-side handler, such as an inventory or UI manager. It holds no external resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** Mutable. The single state field, inventoryRow, is public for direct, low-overhead access. The object's state is intended to be set once upon creation and then treated as immutable during processing.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, serialized, deserialized, and processed within a single thread, typically a Netty I/O thread or the main game loop thread. All synchronization and thread-safe handoffs must be managed by the surrounding network and game engine architecture.

## API Surface
The primary contract is defined by the static codec methods and the Packet interface, not instance methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(1) | Encodes the object's state into a network buffer. For internal use by the protocol engine. |
| deserialize(ByteBuf, int) | static LoadHotbar | O(1) | Decodes a network buffer into a new LoadHotbar instance. For internal use by the protocol engine. |
| getId() | int | O(1) | Returns the unique network identifier (106) for this packet type. |
| computeSize() | int | O(1) | Returns the fixed size of the packet payload in bytes. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-deserialization check to ensure the buffer is large enough to contain the packet. |

## Integration Patterns

### Standard Usage
This packet is never directly instantiated or managed by typical game logic developers. It is handled entirely by the network layer and the systems that subscribe to player inventory events.

A client-side system would listen for this packet type via a central event bus or dispatcher.

```java
// Example of a client-side packet handler
public void onPacketReceived(LoadHotbar packet) {
    // Retrieve the player's inventory and UI systems from the game context
    PlayerInventory inventory = client.getPlayerInventory();
    HotbarUI hotbarUI = client.getUI().getHotbar();

    // Use the data from the packet to update the game state
    inventory.setActiveRow(packet.inventoryRow);
    hotbarUI.redraw();
}
```

### Anti-Patterns (Do NOT do this)
-   **State Caching:** Do not store a reference to a LoadHotbar packet after it has been processed. These objects are transient and should be discarded. Create new instances for new messages.
-   **Manual Codec Invocation:** Do not call `serialize` or `deserialize` directly from game logic. These methods are low-level hooks for the network protocol engine. Sending a packet should be done through a high-level `NetworkManager.sendPacket()` abstraction.
-   **Client-Side Instantiation:** Game clients should never create a LoadHotbar packet. It is a server-authoritative command.

## Data Pipeline
The flow for this packet is unidirectional from the server to a specific client.

> Flow:
> Server Game Logic -> **new LoadHotbar(row)** -> Server Network Engine (Serialization) -> TCP Stream -> Client Network Engine (Deserialization) -> Client Packet Handler -> UI/Inventory System Update

