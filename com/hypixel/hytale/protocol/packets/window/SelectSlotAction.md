---
description: Architectural reference for SelectSlotAction
---

# SelectSlotAction

**Package:** com.hypixel.hytale.protocol.packets.window
**Type:** Transient Data Object

## Definition
```java
// Signature
public class SelectSlotAction extends WindowAction {
```

## Architecture & Concepts
The SelectSlotAction class is a data transfer object (DTO) that represents a discrete user action within the Hytale protocol layer. Specifically, it encapsulates the event of a player selecting a slot within a game window, such as an inventory or a crafting table.

This class is a concrete implementation of the Command pattern. It translates a high-level user interaction into a structured, serializable message that can be transmitted between the client and server. Its primary responsibility is to define the precise binary layout—the wire format—for this specific action.

As a subclass of WindowAction, it participates in a polymorphic system where various UI-related actions can be handled through a common interface. The static methods, particularly deserialize and validateStructure, are critical hooks for the low-level network pipeline (powered by Netty), allowing the engine to decode raw byte streams into meaningful game events.

## Lifecycle & Ownership
- **Creation:**
    - **Client-Side:** Instantiated by UI event handlers when a player interacts with a window slot. The handler populates the slot field and passes the object to the network layer for transmission.
    - **Server-Side:** Instantiated exclusively by the protocol's deserialization pipeline when an incoming network buffer is identified as a SelectSlotAction.
- **Scope:** The object's lifetime is extremely short and transactional. It exists only for the duration of a single processing step: from creation to serialization on the client, or from deserialization to handling on the server.
- **Destruction:** The object holds no external resources and is eligible for garbage collection immediately after its internal data (the slot index) has been read or written. Instances are not pooled or reused.

## Internal State & Concurrency
- **State:** The class holds a single piece of mutable state: the integer field named slot. The object's entire purpose is to transport this value. The static constants define the fixed size and structure for serialization, ensuring protocol consistency.
- **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access. It is expected to be created, populated, and processed within the context of a single network or game-logic thread. Concurrent modification of the slot field will lead to race conditions and unpredictable behavior.

## API Surface
The public API is divided between instance methods for data representation and static methods for protocol integration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SelectSlotAction(int slot) | Constructor | O(1) | Creates a new action for the specified slot index. |
| serialize(ByteBuf buf) | int | O(1) | Writes the object's state into the provided Netty ByteBuf using little-endian byte order. |
| deserialize(ByteBuf buf, int offset) | static SelectSlotAction | O(1) | Constructs a new SelectSlotAction by reading from a ByteBuf at a given offset. |
| validateStructure(ByteBuf buffer, int offset) | static ValidationResult | O(1) | Performs a low-level check to ensure the buffer contains enough bytes to decode the message. |
| computeSize() | int | O(1) | Returns the constant size (4 bytes) of the serialized object. |

## Integration Patterns

### Standard Usage
This object is intended to be used as a fire-and-forget message. The client creates it, sends it, and discards it. The server receives it, processes it, and discards it.

```java
// Client-side: A UI handler sends the action
int clickedSlotIndex = 10;
SelectSlotAction action = new SelectSlotAction(clickedSlotIndex);
clientNetworkManager.sendPacket(action);

// Server-side: A packet handler processes the action
public void handleSelectSlot(PlayerConnection connection, SelectSlotAction action) {
    PlayerInventory inventory = connection.getPlayer().getInventory();
    inventory.setSelectedSlot(action.slot);
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not hold references to SelectSlotAction instances after they have been processed or sent. They are transient and should not be cached.
- **Server-Side Instantiation:** Do not use `new SelectSlotAction()` on the server for any game logic. Server-side instances must originate exclusively from the network deserializer.
- **Modification After Send:** Modifying a SelectSlotAction object after it has been passed to the network layer for serialization can cause undefined behavior, as the serialization may occur on a different thread.

## Data Pipeline
The flow of this data object is linear and unidirectional for a single client-to-server interaction.

> **Flow:**
> Client UI Click -> Event Handler -> **new SelectSlotAction()** -> Network Subsystem -> `serialize()` -> Netty ByteBuf -> Server Network Listener -> Packet Decoder -> `deserialize()` -> **SelectSlotAction Instance** -> Packet Handler -> Game State Update

