---
description: Architectural reference for ClearDebugShapes
---

# ClearDebugShapes

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class ClearDebugShapes implements Packet {
```

## Architecture & Concepts
The ClearDebugShapes class is a protocol-level Data Transfer Object representing a stateless command. It is a "signal packet" whose entire meaning is conveyed by its presence in the network stream, as it carries no payload. Its sole function is to instruct the receiving endpoint, typically the game client, to remove all currently rendered on-screen debugging visualizations, such as bounding boxes, pathing lines, or other developer-centric overlays.

Within the Hytale network protocol, this packet is identified by the static ID 115. The protocol engine uses this ID to map raw network data to an instance of this class. Due to its zero-byte nature, it is an extremely lightweight and efficient mechanism for broadcasting a simple, unambiguous command.

## Lifecycle & Ownership
- **Creation:**
    - **Outbound (Sending):** An instance is created via `new ClearDebugShapes()` on the server when a developer or system command triggers the need to clear a client's debug visuals. The new object is immediately passed to the network layer for serialization.
    - **Inbound (Receiving):** An instance is created by the protocol's central deserialization factory. After reading the packet ID (115) from the network buffer, the factory invokes the static `ClearDebugShapes.deserialize` method, which returns a new instance.
- **Scope:** The lifecycle of a ClearDebugShapes object is exceptionally brief. It is a transient object that exists only for the duration of a single network event processing tick. It is not retained, cached, or referenced after being handled.
- **Destruction:** The object becomes eligible for garbage collection immediately after it has been processed by the target system (e.g., a client-side debug rendering manager).

## Internal State & Concurrency
- **State:** This class is fundamentally stateless and immutable. An instance of ClearDebugShapes contains no fields and its behavior is defined entirely by its type. The static constants within the class define its protocol-level metadata, not instance state.
- **Thread Safety:** The class is inherently thread-safe due to its immutability. However, in practice, packet processing is almost always confined to a single network thread (e.g., a Netty event loop) to prevent race conditions in downstream game logic.

## API Surface
The public API is designed for consumption by the network protocol framework, not for direct invocation by game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier (115) for this packet type. |
| serialize(ByteBuf) | void | O(1) | Writes zero bytes to the buffer. This is a no-operation method. |
| deserialize(ByteBuf, int) | ClearDebugShapes | O(1) | Constructs a new instance. Consumes zero bytes from the buffer. |

## Integration Patterns

### Standard Usage
A developer will typically interact with this packet by either sending it or receiving it through a higher-level abstraction, such as a network connection manager or an event bus.

**Sending a Packet (Server-Side)**
```java
// Obtain the player's network connection and send the command
PlayerConnection connection = player.getConnection();
connection.sendPacket(new ClearDebugShapes());
```

**Receiving a Packet (Client-Side)**
```java
// In a network event listener or handler
public void onPacketReceived(Packet packet) {
    if (packet instanceof ClearDebugShapes) {
        // The presence of the packet is the signal
        DebugRenderManager.getInstance().clearAllShapes();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Adding State:** Do not modify this class to include fields or data. Its contract as a zero-byte signal packet is critical to protocol stability. Adding data would constitute a breaking protocol change.
- **Manual Serialization:** Never call `serialize` or `deserialize` directly. The network pipeline is exclusively responsible for managing the byte-level representation of packets. Manual calls will corrupt the network stream.

## Data Pipeline
The data flow for this packet is a one-way command from a sender to a receiver.

> **Outbound Flow (e.g., Server to Client):**
> Server-Side Command -> `new ClearDebugShapes()` -> Network Pipeline -> **Serializer (No-Op)** -> TCP/UDP Stream

> **Inbound Flow (e.g., Client Receiving):**
> TCP/UDP Stream -> Protocol Demultiplexer (reads ID 115) -> **ClearDebugShapes.deserialize()** -> Packet Event Bus -> Debug Rendering System Handler

