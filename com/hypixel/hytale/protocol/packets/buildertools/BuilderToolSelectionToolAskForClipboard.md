---
description: Architectural reference for BuilderToolSelectionToolAskForClipboard
---

# BuilderToolSelectionToolAskForClipboard

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient

## Definition
```java
// Signature
public class BuilderToolSelectionToolAskForClipboard implements Packet {
```

## Architecture & Concepts
The BuilderToolSelectionToolAskForClipboard class is a network packet that functions as a zero-payload signal within the Hytale protocol. It represents a specific, stateless command sent from a game client to the game server.

Its sole purpose is to request the contents of the server-authoritative builder tool clipboard for the sending player. Unlike packets that transfer data like player position or block updates, this packet's significance is its type, not its content. The server identifies the message by its unique PACKET_ID (410) and understands it as a request to transmit clipboard data back to the client.

This design is highly efficient for simple commands, as it consumes minimal network bandwidth (zero bytes for the payload) and requires trivial processing for serialization and deserialization. It is a fundamental component of the client-server interaction model for the in-game building system.

### Lifecycle & Ownership
- **Creation:**
    - **Client-Side:** Instantiated by the client's input or UI layer when a player executes an action to paste from the builder clipboard (e.g., a key combination). The newly created object is immediately passed to the network layer for serialization and transmission.
    - **Server-Side:** Instantiated by the server's network protocol decoder. When an incoming data stream contains the packet ID 410, the static `deserialize` factory method is invoked to create an instance, which is then passed to a registered packet handler.
- **Scope:** Extremely short-lived. An instance of this packet exists only for the brief moment between its creation and its processing. On the client, it is eligible for garbage collection after being written to the network buffer. On the server, it is discarded after the corresponding packet handler has executed.
- **Destruction:** Handled automatically by the Java Garbage Collector. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields, and its behavior is defined entirely by its type. All constants, such as PACKET_ID, are static and final.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutability, an instance can be safely shared, published, and processed across multiple threads without any risk of data corruption or race conditions. It can be safely moved from a network I/O thread (like Netty's event loop) to a main game logic thread.

## API Surface
The public API is designed for interaction with the network protocol framework, not for direct manipulation by game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (410). |
| serialize(ByteBuf) | void | O(1) | Writes the packet to a network buffer. This is a no-op as the packet has no payload. |
| deserialize(ByteBuf, int) | BuilderToolSelectionToolAskForClipboard | O(1) | Static factory method to construct an instance from a network buffer. |
| computeSize() | int | O(1) | Returns the serialized size of the packet's payload, which is always zero. |

## Integration Patterns

### Standard Usage
This packet is not intended to be used in isolation. It is part of a request-response pattern. The client sends this packet and expects a corresponding response from the server containing the clipboard data.

**Client-Side Sending Logic:**
```java
// Example: A client-side input handler sends the request.
// The networkManager handles serialization and transmission.
networkManager.send(new BuilderToolSelectionToolAskForClipboard());
```

**Server-Side Handling Logic (Conceptual):**
```java
// A packet handler, registered to ID 410, receives the packet.
public void handleAskForClipboard(PlayerConnection connection, BuilderToolSelectionToolAskForClipboard packet) {
    // Retrieve the player-specific clipboard data from the game state.
    ClipboardData clipboard = player.getBuilderTools().getClipboard();

    // Construct and send a response packet containing the data.
    Packet response = new BuilderToolClipboardContentsPacket(clipboard);
    connection.send(response);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation on Server:** The server should never create an instance of this packet to send to a client. It is exclusively a client-to-server message. Doing so would have no effect and violates the protocol's design.
- **Adding State:** Do not modify this class to include data fields. Its contract is to be a zero-byte signal. If a request needs to include parameters, a new, distinct packet class must be defined.
- **Assuming Synchronous Response:** The client must not block or wait for an immediate response after sending this packet. Network communication is asynchronous. The client should listen for the server's response packet (e.g., `BuilderToolClipboardContentsPacket`) and update its state when it arrives.

## Data Pipeline
The flow of this packet represents a simple command from the client to the server, triggering a data-bearing response in the opposite direction.

> **Client to Server Flow:**
> Player Input -> Input System -> **new BuilderToolSelectionToolAskForClipboard()** -> Network Encoder -> Server
>
> **Server-Side Processing:**
> Network Decoder -> **BuilderToolSelectionToolAskForClipboard Instance** -> Packet Handler -> Game Logic -> Response Packet (e.g., BuilderToolClipboardContentsPacket) -> Network Encoder -> Client

