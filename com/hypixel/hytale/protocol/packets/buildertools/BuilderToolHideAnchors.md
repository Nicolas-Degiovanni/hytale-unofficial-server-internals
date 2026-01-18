---
description: Architectural reference for BuilderToolHideAnchors
---

# BuilderToolHideAnchors

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient

## Definition
```java
// Signature
public class BuilderToolHideAnchors implements Packet {
```

## Architecture & Concepts
The BuilderToolHideAnchors class represents a network packet that functions as a stateless command. Within the Hytale protocol, it serves as a signal to toggle the visibility of "anchors" in the game's builder tools interface.

This packet is an example of a **Signal Packet** or **Marker Packet**. It carries no payload or variable data; its very presence in the network stream constitutes the entire message. The server or client acts based on the receipt of this packet type, identified by its static PACKET_ID of 416.

As an implementation of the Packet interface, it is a first-class citizen of the network protocol layer. It is instantiated, serialized, deserialized, and dispatched by the core networking engine, which routes it to a registered handler for processing. Its zero-byte size makes it an extremely efficient mechanism for communicating simple, parameter-less user actions or state changes between the client and server.

### Lifecycle & Ownership
- **Creation:**
    - **Sending Peer:** Instantiated via its constructor (`new BuilderToolHideAnchors()`) immediately before being queued for transmission by the network manager.
    - **Receiving Peer:** Instantiated by the static `deserialize` factory method, which is invoked by the protocol's central packet dispatcher when a packet with ID 416 is read from the network buffer.
- **Scope:** Ephemeral. An instance exists only for the duration of a single network processing tick. It is created, passed to a handler, and then immediately becomes eligible for garbage collection.
- **Destruction:** Managed by the Java Garbage Collector. Once the packet handler completes execution, the object is no longer referenced and its memory is reclaimed.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. Instances contain no fields and their behavior is defined entirely by their type. The static constants define protocol metadata, not instance state.
- **Thread Safety:** Inherently thread-safe. Because instances are immutable, they can be safely shared across threads. However, in practice, a single instance is typically confined to the Netty I/O thread that processed its corresponding network buffer.

## API Surface
The public API is primarily for internal use by the protocol engine. Application-level code should not interact with these methods directly, but rather with the network manager that sends the packet or the handler that receives it.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (416). |
| deserialize(ByteBuf, int) | BuilderToolHideAnchors | O(1) | Static factory method. Creates a new instance. Consumes zero bytes. |
| serialize(ByteBuf) | void | O(1) | Writes nothing to the buffer. The packet's ID is handled by the protocol frame encoder. |
| computeSize() | int | O(1) | Returns 0, indicating this packet has no data payload. |
| clone() | BuilderToolHideAnchors | O(1) | Creates a new, identical instance. |

## Integration Patterns

### Standard Usage
This packet is not used directly. It is created and sent through a higher-level network service. The receiving end registers a handler to react to its arrival.

**Sending the command:**
```java
// Example: A UI event handler sends the packet
// This code would exist within a client-side system.
NetworkConnectionManager connection = context.getService(NetworkConnectionManager.class);
connection.sendPacket(new BuilderToolHideAnchors());
```

**Handling the command:**
```java
// Example: A packet handler registry on the server-side
// The dispatcher invokes this method upon receiving the packet.
public void handle(BuilderToolHideAnchors packet, PlayerConnection sender) {
    BuilderToolState state = sender.getBuilderToolState();
    state.setAnchorsVisible(false);
}
```

### Anti-Patterns (Do NOT do this)
- **Adding State:** Do not modify this class to add fields. If data needs to be transmitted, a new packet class must be defined. This packet's contract is to be a zero-byte signal.
- **Manual Serialization:** Never call `serialize` or `deserialize` directly. These methods are invoked exclusively by the Netty pipeline and packet dispatcher. Bypassing the protocol engine will lead to corrupted network streams.
- **Expecting a Reply:** This is a fire-and-forget command. Do not design logic that blocks or waits for a direct response to the transmission of this specific packet.

## Data Pipeline
The flow of this command involves user input on one peer triggering a state change on the remote peer.

> **Flow (Client to Server):**
> User Input (e.g., Key Press) -> Client-Side UI System -> `new BuilderToolHideAnchors()` -> Network Manager -> **Serialization Pipeline** -> TCP/IP Stack -> Server TCP/IP Stack -> **Deserialization Pipeline** -> Packet Dispatcher -> Server-Side Packet Handler -> Game State Update

