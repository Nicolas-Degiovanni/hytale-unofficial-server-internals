---
description: Architectural reference for SmartMoveItemStack
---

# SmartMoveItemStack

**Package:** com.hypixel.hytale.protocol.packets.inventory
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class SmartMoveItemStack implements Packet {
```

## Architecture & Concepts
The SmartMoveItemStack class is a Data Transfer Object (DTO) that represents a specific client-to-server network command within the Hytale protocol. It encapsulates a player's intent to perform a "smart" inventory action, such as shift-clicking an item to quickly move it to a corresponding equipment slot or merge it with an existing stack.

Unlike a simple move packet that would specify both a source and a destination, this packet only defines the source item stack and a high-level *intent*, represented by the SmartMoveType enum. This design offloads the complex logic of determining the appropriate destination slot to the server. This centralizes game rules, reduces client-side complexity, and prevents cheating by disallowing clients from specifying arbitrary destination slots.

This class is a fundamental component of the client-server inventory synchronization model. It is designed to be lightweight, with a fixed binary layout for efficient serialization and deserialization by the Netty-based network layer.

## Lifecycle & Ownership
- **Creation:**
    - **Client-Side:** Instantiated by the client's UI or input handling systems in direct response to a player action (e.g., shift-clicking an item in an inventory grid).
    - **Server-Side:** Instantiated by the protocol's packet deserialization layer when a raw byte buffer with Packet ID 176 is received from a client. The static `deserialize` method is the designated factory.
- **Scope:** This object has an extremely short and transient lifecycle. On the client, it exists only for the duration of the serialization and network dispatch process. On the server, it exists only for the duration of a single packet processing tick.
- **Destruction:** The object is eligible for garbage collection immediately after being processed by the server-side game logic or after being written to the network buffer on the client. There are no manual cleanup or `close` operations required.

## Internal State & Concurrency
- **State:** The class is a mutable container for inventory action data. Its public fields (`fromSectionId`, `fromSlotId`, `quantity`, `moveType`) are directly accessible and modifiable after construction. This facilitates easy population of the packet's data before serialization.

- **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization primitives.

    > **Warning:** Instances of SmartMoveItemStack must be confined to a single thread. On the server, this is typically a Netty I/O worker thread or a dedicated game logic thread. Do not share or modify an instance across multiple threads without implementing external synchronization, as this will lead to race conditions and unpredictable behavior.

## API Surface
The public API is designed for network protocol integration, focusing on serialization, deserialization, and metadata.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier (176) for this packet type. |
| serialize(ByteBuf) | void | O(1) | Encodes the object's state into the provided Netty ByteBuf using Little Endian byte order. |
| deserialize(ByteBuf, int) | SmartMoveItemStack | O(1) | A static factory method that decodes a new instance from a ByteBuf at a given offset. |
| computeSize() | int | O(1) | Returns the fixed size of the packet's payload in bytes (13). |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a preliminary check to ensure the buffer contains enough data to read the packet. |

## Integration Patterns

### Standard Usage
This packet is created by the client and consumed by the server. The server's packet handler receives the deserialized object and forwards it to the appropriate game system for processing.

```java
// Example: Server-side packet handler logic
public void handleSmartMove(PlayerConnection connection, SmartMoveItemStack packet) {
    Player player = connection.getPlayer();
    InventoryManager inventoryManager = player.getInventoryManager();

    // The server's game logic interprets the "smart move"
    inventoryManager.processSmartMove(
        packet.fromSectionId,
        packet.fromSlotId,
        packet.quantity,
        packet.moveType
    );
}
```

### Anti-Patterns (Do NOT do this)
- **Object Reuse:** Do not modify and resend the same SmartMoveItemStack instance. Its mutable nature makes this practice highly error-prone. Always create a new packet for each distinct user action.
- **Manual Serialization:** Do not attempt to write the fields to a ByteBuf manually. Always use the provided `serialize` method to guarantee correctness, especially regarding byte order (Little Endian) and field layout.
- **Client-Side Logic:** Do not implement complex destination-finding logic on the client. The purpose of this packet is to delegate that responsibility to the server. The client should only report the user's intent.

## Data Pipeline
The flow of data for this packet is unidirectional from the client to the server.

> **Flow:**
> Client User Input (Shift-Click) -> UI Event Handler -> **new SmartMoveItemStack(...)** -> Client Network Layer (Serialization) -> Server Network Layer (Deserialization) -> **SmartMoveItemStack Instance** -> Server Packet Handler -> Inventory Game Logic -> World State Update

