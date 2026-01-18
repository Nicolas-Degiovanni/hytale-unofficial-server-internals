---
description: Architectural reference for BuilderToolSetTransformationModeState
---

# BuilderToolSetTransformationModeState

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class BuilderToolSetTransformationModeState implements Packet {
```

## Architecture & Concepts
The BuilderToolSetTransformationModeState class is a network Packet Data Transfer Object (DTO). It represents a discrete, low-level network message used to synchronize the state of a specific builder tool feature between the client and server.

As an implementation of the Packet interface, this class is a fundamental component of the Hytale network protocol layer. Its primary role is to encapsulate a single boolean state: whether the "Transformation Mode" for the builder tools is enabled or disabled. The class itself contains no logic; it is a pure data container.

The network engine relies on the static metadata fields within this class, such as PACKET_ID, FIXED_BLOCK_SIZE, and MAX_SIZE. This metadata allows the protocol's serialization and deserialization systems to operate without reflection, enabling high-performance buffer processing. This packet is designed for minimal overhead, consuming exactly one byte on the wire.

## Lifecycle & Ownership
- **Creation:** An instance is created under two circumstances:
    1.  **Client-Side (Sending):** Instantiated by a client-side system, such as a UI controller or input handler, when the player toggles the transformation mode. The newly created object is then passed to the client's network connection manager for serialization and transmission.
    2.  **Server-Side (Receiving):** Instantiated by the server's packet deserialization pipeline when a raw network buffer with the corresponding PACKET_ID (408) is received from a client. The static `deserialize` method is invoked to construct the object from the buffer.

- **Scope:** Extremely short-lived and transient. The object exists only for the brief moment it takes to be serialized into a network buffer or deserialized and passed to a packet handler. It is not intended to be stored or referenced long-term.

- **Destruction:** The object becomes eligible for garbage collection immediately after its data has been processed by a packet handler or written to a ByteBuf. There is no manual destruction or cleanup mechanism.

## Internal State & Concurrency
- **State:** Mutable. The single state field, `enabled`, is a public boolean. This design prioritizes simplicity and performance for a simple data container over encapsulation.

- **Thread Safety:** This class is **not thread-safe**. It contains no locks or other concurrency controls. It is designed to be created, populated, and processed within a single thread context, such as a Netty event loop thread or the main game thread. Concurrent access to an instance from multiple threads will result in unpredictable behavior and is a critical anti-pattern.

## API Surface
The public API is designed for interaction with the network protocol engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier (408) for this packet type. |
| serialize(ByteBuf) | void | O(1) | Writes the boolean `enabled` state as a single byte into the provided buffer. |
| deserialize(ByteBuf, int) | BuilderToolSetTransformationModeState | O(1) | **Static Factory.** Reads one byte from the buffer at the given offset and constructs a new instance. |
| computeSize() | int | O(1) | Returns the fixed size of the packet payload in bytes, which is always 1. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static Validator.** Checks if the buffer contains at least 1 readable byte at the offset. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to create an instance, set its state, and immediately pass it to a network service for dispatch. The receiving end gets a fully-formed object and reads its state.

```java
// Client-side: Sending the packet to the server
// Assume 'networkService' is an injected dependency that handles the connection.
BuilderToolSetTransformationModeState packet = new BuilderToolSetTransformationModeState(true);
networkService.sendPacket(packet);
```

```java
// Server-side: Handling the received packet
// This code would exist within a packet handler method.
public void handleTransformationModeState(BuilderToolSetTransformationModeState packet) {
    Player aPlayer = getPlayerFromContext();
    aPlayer.getBuilderTools().setTransformationMode(packet.enabled);
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not store instances of this packet in caches or as member variables of long-lived services. It represents a transient event, not a persistent state. Storing it can lead to sending stale data.
- **Instance Re-use:** Do not modify and re-send the same packet instance. This is especially dangerous in a multi-threaded network environment. Always create a new instance for each message to ensure thread-safety and data integrity.
- **Direct Serialization:** Application-level code should never call `serialize` or `deserialize` directly. These methods are the contract with the low-level network protocol engine. Interacting with them directly bypasses the network pipeline's logic for compression, encryption, and batching.

## Data Pipeline
The flow of data encapsulated by this packet is unidirectional, typically from client to server to signal a user action.

> **Flow (Client to Server):**
> Player Input (UI Click) -> InputHandler -> **new BuilderToolSetTransformationModeState(state)** -> ClientNetworkManager.send() -> ProtocolSerializer -> Netty Channel -> Server

> **Flow (Server-Side Processing):**
> Netty Channel -> ProtocolDeserializer -> **BuilderToolSetTransformationModeState instance** -> PacketDispatcher -> PlayerSession.onPacket(packet) -> Game Logic Update

