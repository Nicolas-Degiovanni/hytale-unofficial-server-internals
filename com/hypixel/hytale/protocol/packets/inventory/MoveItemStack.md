---
description: Architectural reference for MoveItemStack
---

# MoveItemStack

**Package:** com.hypixel.hytale.protocol.packets.inventory
**Type:** Transient

## Definition
```java
// Signature
public class MoveItemStack implements Packet {
```

## Architecture & Concepts
The MoveItemStack class is a Data Transfer Object (DTO) that represents a single, atomic inventory operation within the Hytale network protocol. It is not a service or manager; it is a *message* encapsulating the user's intent to move a quantity of items from one inventory location to another.

As an implementation of the Packet interface, this class serves as a strict data contract between the client and server. Its primary role is to be serialized into a predictable binary format for network transmission and deserialized back into an object on the receiving end. The protocol relies on the static PACKET_ID of 175 to identify and route raw network data to the correct deserializer for this type.

The structure is defined by a fixed-size block of 20 bytes, containing five 4-byte little-endian integers. This fixed-size nature makes it highly efficient for the protocol engine to parse, as it avoids complex logic for variable-length fields.

## Lifecycle & Ownership
- **Creation:**
    - **Outbound (Sending):** Instantiated directly by high-level game logic, typically in response to a user interface event like dragging and dropping an item. The game code populates its fields and submits it to the network layer.
    - **Inbound (Receiving):** Instantiated by the protocol's central packet factory or dispatcher. When a data frame with packet ID 175 arrives, the dispatcher invokes the static `deserialize` method, which allocates and returns a new MoveItemStack instance.
- **Scope:** Extremely short-lived and ephemeral. An instance of MoveItemStack exists only for the brief period between its creation and its final processing.
- **Destruction:** The object becomes eligible for garbage collection immediately after its data is written to the network buffer (outbound) or after it has been processed by a packet handler (inbound). There is no manual resource management associated with this class.

## Internal State & Concurrency
- **State:** Mutable. MoveItemStack is a simple data container with public fields. It holds no references to other engine systems and does not cache any data. Its state is entirely defined by the five integer properties representing the move operation.
- **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** It is designed to be created, populated, and processed within a single, well-defined execution context, such as the main game thread or a Netty network thread. Accessing an instance from multiple threads without external locking will lead to race conditions and undefined behavior.

## API Surface
The primary contract is defined by the static serialization and validation methods, not by typical instance methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | MoveItemStack | O(1) | **Static Factory.** Constructs a new MoveItemStack by reading 20 bytes from the provided network buffer at a given offset. This is the primary entry point for inbound data. |
| serialize(ByteBuf) | void | O(1) | Writes the object's five integer fields into the provided network buffer using little-endian byte order. This is the primary exit point for outbound data. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static Pre-check.** Verifies that the buffer contains at least 20 readable bytes before attempting deserialization. This prevents buffer underflow exceptions. |

## Integration Patterns

### Standard Usage
The following demonstrates how client-side game logic would create and dispatch a MoveItemStack packet to the server after a user performs an inventory action.

```java
// Executed within the game's UI or inventory management system
void onPlayerMoveItem(int fromSection, int fromSlot, int toSection, int toSlot, int amount) {
    // 1. Create the packet with the action's data
    MoveItemStack packet = new MoveItemStack(fromSection, fromSlot, amount, toSection, toSlot);

    // 2. Submit the packet to the network service for serialization and transmission
    // WARNING: The 'networkService' is a hypothetical representation.
    networkService.sendPacket(packet);
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification After Dispatch:** Do not modify a MoveItemStack object after passing it to the network layer. The serialization may occur on a separate thread or at a later time, leading to data corruption. Always treat the object as immutable after it has been sent.
- **Reusing Instances:** Do not reuse a packet instance for multiple network messages. This is a false optimization that can cause severe bugs if the object is still referenced elsewhere. Always create a new instance for each distinct action.
- **Manual Deserialization:** Never call the static `deserialize` method directly from game logic. The network protocol engine is solely responsible for parsing inbound byte streams and constructing packet objects.

## Data Pipeline
The flow of MoveItemStack data is linear and unidirectional for any single operation.

> **Outbound Flow (Client to Server):**
> UI Input (e.g., Mouse Drag) -> InventoryController -> `new MoveItemStack(...)` -> NetworkService.send() -> **MoveItemStack.serialize(ByteBuf)** -> TCP/IP Stack

> **Inbound Flow (Server to Client):**
> TCP/IP Stack -> Netty Channel -> Raw ByteBuf -> Packet Dispatcher (reads ID 175) -> **MoveItemStack.deserialize(ByteBuf)** -> Packet Handler -> Game State Update

