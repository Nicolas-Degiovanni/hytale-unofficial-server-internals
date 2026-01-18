---
description: Architectural reference for SetMovementStates
---

# SetMovementStates

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class SetMovementStates implements Packet {
```

## Architecture & Concepts
The SetMovementStates class is a network protocol Data Transfer Object. It serves a single, critical purpose: to communicate a change in a player's fundamental movement state (such as sprinting, sneaking, or flying) between the client and the server. As an implementation of the Packet interface, it is a foundational component of the Hytale network layer.

This class is not a service or a manager; it is a pure data container. Its structure is highly optimized for network efficiency, designed to be serialized into a minimal binary representation. The design explicitly avoids complex logic, focusing instead on the precise definition of a network message. The use of a bitmask for null fields and fixed-size computations underscores its role in a performance-sensitive real-time environment.

It represents a single, atomic update from the client's input controller to the server's authoritative player entity simulation.

### Lifecycle & Ownership
- **Creation:**
    - **Client-Side:** Instantiated by the client's input handling system whenever a player's movement state changes. For example, when the sprint key is pressed, a new SetMovementStates object is created with the updated state.
    - **Server-Side:** Instantiated by the network protocol decoder when a corresponding byte stream (with Packet ID 102) is received from a client. The static deserialize method is the designated factory.

- **Scope:** Transient and extremely short-lived. An instance exists only for the duration of a single network transaction. On the client, it is created, passed to the network layer for serialization, and then becomes eligible for garbage collection. On the server, it is created during deserialization, processed by a packet handler, and immediately becomes eligible for garbage collection.

- **Destruction:** Managed entirely by the Java Garbage Collector. No manual resource cleanup is required.

## Internal State & Concurrency
- **State:** The class holds mutable state through its single public field, movementStates. This field is an instance of SavedMovementStates, which encapsulates the specific flags for movement (e.g., isSprinting). The state is simple, directly mapping to the data payload of the network packet.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and processed within a single thread, typically a Netty I/O worker thread or the main game thread. Accessing or modifying an instance from multiple threads without external synchronization will result in undefined behavior and data corruption.

## API Surface
The public API is minimal, focusing exclusively on serialization, deserialization, and metadata for the network protocol.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (102). |
| serialize(ByteBuf) | void | O(1) | Encodes the object's state into the provided Netty byte buffer. |
| deserialize(ByteBuf, int) | SetMovementStates | O(1) | Static factory method. Decodes a new instance from a byte buffer at a given offset. |
| computeSize() | int | O(1) | Returns the exact number of bytes this packet will consume on the wire (2 bytes). |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a low-cost check to ensure the buffer is large enough to contain this packet. |

## Integration Patterns

### Standard Usage
This packet is typically created on the client and sent to the server. The server then receives it and updates the player's state.

```java
// Client-side: Creating and sending the packet
SavedMovementStates currentStates = player.getMovementController().getStates();
SetMovementStates packet = new SetMovementStates(currentStates);

// The network manager handles the actual serialization and transmission
clientNetworkManager.sendPacket(packet);
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not modify and resend the same SetMovementStates instance. Packets are ephemeral; create a new instance for each state change to avoid race conditions and unexpected behavior in the network pipeline.
- **Manual Serialization:** Do not attempt to read or write the packet's data to a ByteBuf manually. Always use the provided serialize and deserialize methods to ensure protocol compliance. The internal binary layout is an implementation detail.
- **Cross-Thread Modification:** Do not create a packet on the main thread and pass a reference to a network thread for modification. The object should be fully constructed before being handed off for serialization.

## Data Pipeline
The flow for this packet is unidirectional from the client to the server, representing a command to change state.

> Flow:
> Client Input (e.g., Key Press) -> Player Controller -> **SetMovementStates** (Instantiation) -> Client Network Manager -> Serialization -> TCP/IP Stack -> Server Network Listener -> Packet Decoder -> **SetMovementStates** (Deserialization) -> Server Packet Handler -> Player Entity State Update

