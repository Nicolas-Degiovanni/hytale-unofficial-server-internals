---
description: Architectural reference for SetActiveSlot
---

# SetActiveSlot

**Package:** com.hypixel.hytale.protocol.packets.inventory
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class SetActiveSlot implements Packet {
```

## Architecture & Concepts
The SetActiveSlot class is a network packet definition within Hytale's client-server protocol. It serves as a pure data container, representing a command sent from the client to the server to change the player's currently selected inventory slot, typically within the hotbar.

As a DTO implementing the Packet interface, its sole responsibility is to encapsulate the state required for this action and provide the logic for its own serialization and deserialization. It contains no business logic. The processing of this command is handled by dedicated packet processors on the server, which receive the deserialized object and apply the state change to the corresponding player's game state.

This class is a fundamental building block of the player interaction model, translating a direct user input (like scrolling a mouse wheel or pressing a number key) into a structured, network-transmissible message.

### Lifecycle & Ownership
- **Creation:**
    - **Client-Side:** Instantiated by a client-side system, such as an InputManager or PlayerController, in direct response to a user action that changes the selected hotbar slot.
    - **Server-Side:** Instantiated by the network layer's packet deserialization pipeline when a raw byte buffer with Packet ID 177 is received from a client.
- **Scope:** Ephemeral and extremely short-lived. An instance exists only for the brief period between its creation and its processing. On the client, this is the time it takes to be serialized and queued for network transmission. On the server, it is the time from deserialization until its corresponding handler has finished execution.
- **Destruction:** The object is eligible for garbage collection immediately after being processed. There is no long-term ownership or state retention.

## Internal State & Concurrency
- **State:** Mutable. The class holds two integer fields, inventorySectionId and activeSlot. These are public and can be modified after instantiation, though this is not the intended pattern. The object is designed to be populated once and then treated as immutable data for the remainder of its lifecycle. It does not cache any data.
- **Thread Safety:** **Not thread-safe.** This object is designed for single-threaded access within the context of a network processing loop (e.g., a Netty event loop thread). Concurrent modification from multiple threads will lead to race conditions and unpredictable behavior. All interaction with a SetActiveSlot instance must be externally synchronized or confined to a single thread.

## API Surface
The public contract is primarily defined by the Packet interface and the static deserialization factory.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier (177) for this packet type. |
| serialize(ByteBuf) | void | O(1) | Writes the object's state into the provided Netty ByteBuf using little-endian byte order. |
| deserialize(ByteBuf, int) | static SetActiveSlot | O(1) | Static factory method. Reads 8 bytes from the buffer at the given offset and constructs a new SetActiveSlot instance. |
| computeSize() | int | O(1) | Returns the fixed size of the packet's payload in bytes, which is always 8. |

## Integration Patterns

### Standard Usage
This packet is created and dispatched by client-side game logic and consumed by server-side packet handlers. A developer should never interact with its serialization methods directly.

```java
// Client-side: A player controller sends the packet
int newSlotIndex = 4;
int hotbarSectionId = 0; // Assuming 0 is the main hotbar

SetActiveSlot packet = new SetActiveSlot(hotbarSectionId, newSlotIndex);
networkManager.sendPacket(packet);

// Server-side: A packet handler processes the data
public void handleSetActiveSlot(PlayerConnection connection, SetActiveSlot packet) {
    PlayerInventory inventory = connection.getPlayer().getInventory();
    inventory.setActiveSlot(packet.inventorySectionId, packet.activeSlot);
}
```

### Anti-Patterns (Do NOT do this)
- **Object Re-use:** Do not attempt to pool or re-use SetActiveSlot instances. They are lightweight objects, and the cost of creation is negligible. Re-using instances can lead to subtle bugs if state is not properly reset.
- **Manual Serialization:** Never call serialize or deserialize directly. These methods are strictly for internal use by the protocol's serialization engine. Interacting with them manually bypasses the engine's pipeline management.
- **State Modification After Queuing:** Do not modify the packet's fields after it has been passed to a network manager for sending. The serialization may happen on a different thread, leading to a race condition.

## Data Pipeline
The flow of data encapsulated by this packet is unidirectional from client to server.

> **Flow:**
> Client User Input -> InputManager -> **new SetActiveSlot()** -> NetworkManager -> PacketSerializer -> TCP Stream -> Server Network Layer -> PacketDeserializer -> **SetActiveSlot.deserialize()** -> PacketProcessor -> Game State Update

