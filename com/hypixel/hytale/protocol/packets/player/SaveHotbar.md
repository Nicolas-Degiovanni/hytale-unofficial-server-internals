---
description: Architectural reference for the SaveHotbar network packet.
---

# SaveHotbar

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class SaveHotbar implements Packet {
```

## Architecture & Concepts
The SaveHotbar class is a network packet definition within Hytale's client-server protocol. It serves a singular, specific purpose: to communicate a change in the player's currently selected hotbar row from the client to the server.

As an implementation of the Packet interface, this class is not a service or a long-lived component. It is a simple, structured data container designed for serialization into a byte stream. Its existence is ephemeral, acting as a message that is created, sent, and immediately discarded. Architecturally, it sits at the boundary between the game logic (player input and inventory management) and the low-level network transport layer, providing a clean, type-safe contract for a specific game event.

The static fields like PACKET_ID, FIXED_BLOCK_SIZE, and validation methods are part of a larger protocol framework. This framework uses these metadata constants to efficiently dispatch incoming byte streams to the correct deserializer without reflection, ensuring high performance in the server's network processing pipeline.

## Lifecycle & Ownership
- **Creation:**
    - **Client-Side:** Instantiated by the client's input or inventory management system in direct response to a player action, such as scrolling the mouse wheel or pressing a number key to switch the active hotbar.
    - **Server-Side:** Instantiated by the server's network protocol decoder when a raw network buffer with Packet ID 107 is received. The static deserialize method is invoked to construct the object from the buffer.
- **Scope:** Extremely short-lived. A SaveHotbar object exists only for the duration of a single transaction. On the client, it lives from creation until it is passed to the network layer for serialization. On the server, it lives from deserialization until the corresponding packet handler has finished processing it.
- **Destruction:** The object becomes eligible for garbage collection immediately after its data has been serialized to a network buffer (client) or its state has been processed by a handler (server). No system should retain a reference to a packet instance after processing.

## Internal State & Concurrency
- **State:** Mutable, containing a single byte field, inventoryRow. While technically mutable, it is intended to be treated as a write-once object. Its state is set at creation and should not be modified thereafter. It holds no caches or complex internal data structures.
- **Thread Safety:** **This class is not thread-safe.** It is a plain data object with no internal synchronization. It is expected to be created, populated, and processed within a single thread context, such as the main game thread or a dedicated Netty event loop thread. Passing an instance across thread boundaries without explicit synchronization is a severe anti-pattern and will lead to race conditions.

## API Surface
The public contract is primarily for the network protocol framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SaveHotbar(byte) | constructor | O(1) | Creates a new packet with the specified hotbar row index. |
| serialize(ByteBuf) | void | O(1) | Writes the single inventoryRow byte into the provided Netty buffer. |
| deserialize(ByteBuf, int) | static SaveHotbar | O(1) | Reads one byte from the buffer at the given offset and constructs a new SaveHotbar instance. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a bounds check to ensure the buffer contains enough data for a valid packet. |
| getId() | int | O(1) | Returns the static network identifier (107) for this packet type. |

## Integration Patterns

### Standard Usage
The class is intended to be instantiated, populated, and immediately dispatched to the network system.

**Client-Side Example:**
```java
// In an input handler or inventory controller
byte newHotbarIndex = 4;
SaveHotbar packet = new SaveHotbar(newHotbarIndex);
networkManager.sendPacket(packet);
```

**Server-Side Handler (Conceptual):**
```java
// In a packet handler that receives a deserialized SaveHotbar object
public void handleSaveHotbar(PlayerConnection connection, SaveHotbar packet) {
    Player player = connection.getPlayer();
    player.getInventory().setActiveHotbarRow(packet.inventoryRow);
}
```

### Anti-Patterns (Do NOT do this)
- **Object Re-use:** Do not cache and re-use SaveHotbar instances. They are extremely lightweight, and creating a new instance for each event prevents difficult-to-diagnose state bugs.
- **Long-Term Storage:** Do not store references to SaveHotbar packets in services, game components, or collections. They represent a point-in-time event, not persistent state.
- **Cross-Thread Modification:** Do not create a packet on one thread and modify its contents on another before sending. This will lead to unpredictable data being serialized.

## Data Pipeline
The SaveHotbar packet is a key element in the data flow for player inventory controls.

> **Flow (Client to Server):**
> Player Input (Scroll Wheel) -> InputManager -> InventoryController -> `new SaveHotbar(row)` -> NetworkSystem.send() -> **SaveHotbar.serialize()** -> Netty Channel -> TCP/IP Stack -> Server Network Layer -> Packet Dispatcher (ID 107) -> **SaveHotbar.deserialize()** -> Server Packet Handler -> Player State Update

