---
description: Architectural reference for InventoryAction
---

# InventoryAction

**Package:** com.hypixel.hytale.protocol.packets.inventory
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class InventoryAction implements Packet {
```

## Architecture & Concepts
The InventoryAction class is a network packet, a specialized Data Transfer Object (DTO) designed for client-server communication. It represents a single, discrete action a player attempts to perform on an inventory, such as moving, dropping, or splitting an item stack.

This packet is a fundamental component of the client-authoritative input model for inventory management. The client user interface generates an InventoryAction in response to player input. This action is then serialized and sent to the server as a command. The server receives this command, validates it against the authoritative game state, and, if valid, applies the change. The resulting state modification is then broadcast to all relevant clients, ensuring synchronization.

It is not a service or a manager; it is a pure data container whose structure is strictly defined by the network protocol. Its primary role is to carry state from the client's input layer to the server's game logic layer.

## Lifecycle & Ownership
- **Creation:** An InventoryAction instance is created under two distinct circumstances:
    1. **Client-Side:** Instantiated by UI event handlers when a player interacts with an inventory screen. The object is populated with data corresponding to the specific click, drag, or keypress.
    2. **Server-Side:** Instantiated by the network protocol layer's deserializer, specifically the static *deserialize* method, when an incoming byte stream with packet ID 179 is processed.

- **Scope:** The object's lifetime is exceptionally short and bound to a single transaction. On the client, it exists only long enough to be passed to the network sender for serialization. On the server, it exists only for the duration of the packet handling logic within a single game tick.

- **Destruction:** The object is managed by the Java Garbage Collector. As it is a simple, short-lived POJO with no external resources, it is eligible for collection as soon as it falls out of the scope of the method that created or received it.

## Internal State & Concurrency
- **State:** The class holds a small, mutable state consisting of the inventory section, the action type, and associated data. The public fields allow for direct modification, which is a common and acceptable pattern for internal DTOs that are populated by deserializers or constructed on-the-fly. The state is defined by the fixed-size block of 6 bytes specified in the protocol.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and processed within a single thread's context, such as a client's UI thread or a server's network processing thread (e.g., a Netty event loop).

    **WARNING:** Sharing an InventoryAction instance across multiple threads will lead to unpredictable behavior and data corruption. Do not store instances in shared collections or modify them from a different thread than the one that created them.

## API Surface
The public API is designed for serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| InventoryAction() | Constructor | O(1) | Creates a default instance. |
| InventoryAction(int, InventoryActionType, byte) | Constructor | O(1) | Creates a fully populated instance. |
| getId() | int | O(1) | Returns the static network ID (179) for this packet type. |
| serialize(ByteBuf) | void | O(1) | Writes the object's state into a Netty byte buffer. |
| computeSize() | int | O(1) | Returns the fixed size of the packet payload, which is 6 bytes. |
| deserialize(ByteBuf, int) | static InventoryAction | O(1) | Creates a new InventoryAction by reading from a byte buffer at an offset. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check to ensure a buffer contains enough data to deserialize. |
| clone() | InventoryAction | O(1) | Creates a shallow copy of the instance. |

## Integration Patterns

### Standard Usage
An InventoryAction is created by a client-side system, sent to the server, and processed. It is never managed by a dependency injection framework or service locator.

```java
// Client-side: Responding to a UI event to take all items from slot 5
// in the main player inventory (section 0).
InventoryAction takeAction = new InventoryAction(
    0, // inventorySectionId for the main grid
    InventoryActionType.TakeAll,
    (byte) 5 // actionData representing the slot index
);

// The network system would then take this object and serialize it.
networkManager.sendPacket(takeAction);
```

### Anti-Patterns (Do NOT do this)
- **Object Re-use:** Do not attempt to cache and re-use InventoryAction instances. They are lightweight and should be created for each distinct user action to prevent state leakage between operations.
- **Post-Serialization Modification:** Modifying an InventoryAction object after it has been queued for sending to the network layer is a severe race condition. The serialization may occur on a different thread, leading to a partially updated or corrupted packet being sent.
- **Server-Side Instantiation:** Do not use *new InventoryAction()* on the server for any purpose other than testing. All incoming actions must originate from the *deserialize* method to ensure they reflect actual data from the client.

## Data Pipeline
The flow of this data object is linear and unidirectional from client to server.

> **Client Flow:**
> Player Input (e.g., Mouse Click) -> UI Event Handler -> **InventoryAction (Creation)** -> Network System -> Serialization -> Outbound ByteBuf

> **Server Flow:**
> Inbound ByteBuf -> Packet Demultiplexer (reads ID 179) -> **InventoryAction.deserialize()** -> Game Logic (Inventory System) -> World State Update -> (Optional) State Sync Packet to Clients

