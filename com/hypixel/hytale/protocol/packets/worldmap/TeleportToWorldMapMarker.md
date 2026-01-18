---
description: Architectural reference for TeleportToWorldMapMarker
---

# TeleportToWorldMapMarker

**Package:** com.hypixel.hytale.protocol.packets.worldmap
**Type:** Transient Data Object

## Definition
```java
// Signature
public class TeleportToWorldMapMarker implements Packet {
```

## Architecture & Concepts
The TeleportToWorldMapMarker class is a Data Transfer Object (DTO) that represents a specific, concrete message within the Hytale network protocol. It is not a service or manager; it is a pure data container whose sole purpose is to encapsulate a client's request to be teleported to a named location on the world map.

This class embodies a common pattern in high-performance networking stacks where the data structure and its binary serialization/deserialization logic are tightly coupled. By including static methods like `deserialize` and `validateStructure`, the class allows the network layer to operate directly on raw byte streams without needing to instantiate objects prematurely. This design is critical for performance, reducing garbage collection pressure, and for security, as it enables pre-validation of untrusted data before it enters the main game logic.

Its role is to act as a formal contract for message ID 244, ensuring that both the client and server agree on the exact binary layout for this specific network command.

### Lifecycle & Ownership
-   **Creation:**
    -   **Sending Peer (Client):** Instantiated directly via its constructor (`new TeleportToWorldMapMarker(...)`) by game logic, typically in response to a user interface event like clicking a map icon.
    -   **Receiving Peer (Server):** Instantiated by the protocol's packet decoder. The static `deserialize` factory method is invoked when the network layer reads a packet with ID 244 from the incoming byte buffer.
-   **Scope:** Extremely short-lived and transient. An instance of this packet exists only for the brief moment it is being processed. On the sending side, it lives until it is serialized into a network buffer. On the receiving side, it lives from deserialization until the corresponding packet handler has finished processing the request.
-   **Destruction:** The object becomes eligible for garbage collection immediately after use. No manual resource management is required.

## Internal State & Concurrency
-   **State:** The internal state consists of a single mutable `String id`. While technically mutable, instances of this class should be treated as immutable after their initial creation. Modifying a packet after it has been queued for sending is a severe anti-pattern.
-   **Thread Safety:** This class is **not thread-safe**. It is a simple data holder with no internal locking or synchronization. It is designed to be created on one thread, and then safely transferred to another (e.g., from a UI thread to a network thread) via a message queue. Concurrent modification will lead to race conditions and undefined behavior.

## API Surface
The public API is primarily for use by the network protocol framework, not for general game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(N) | Serializes the packet's state into the provided Netty ByteBuf. N is the length of the ID string. |
| deserialize(ByteBuf, int) | static TeleportToWorldMapMarker | O(N) | Factory method to construct a packet by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| computeSize() | int | O(N) | Calculates the exact number of bytes the packet will occupy when serialized. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a security and integrity check on the raw buffer data without full deserialization. Critical for preventing buffer overflows or parsing errors. |
| getId() | int | O(1) | Returns the static network identifier for this packet type, which is 244. |

## Integration Patterns

### Standard Usage
The standard pattern is to create an instance, populate its data, and hand it off to a network connection manager for dispatch. The framework handles the serialization and network transport.

```java
// Example: Client-side code sending the request to the server.
// Assumes 'clientConnection' is an existing object managing the network session.

String destinationMarkerId = "capital-city-spawn";
TeleportToWorldMapMarker teleportRequest = new TeleportToWorldMapMarker(destinationMarkerId);

clientConnection.sendPacket(teleportRequest);
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not hold onto a packet instance to modify and resend it. This is not safe, especially in an asynchronous networking environment. Always create a new packet object for each distinct request.
-   **Manual Serialization:** Avoid calling `serialize` directly. The network pipeline contains specialized encoders (e.g., Netty handlers) that are designed to manage the lifecycle of the ByteBuf and correctly frame the packet with its ID and length. Manual serialization can easily lead to corrupted network streams.
-   **Bypassing Validation:** A server implementation must never call `deserialize` on data from a client without first calling `validateStructure`. Bypassing this check exposes the server to denial-of-service attacks via maliciously crafted packets designed to trigger exceptions or resource exhaustion.

## Data Pipeline
This packet follows a unidirectional client-to-server data flow.

> Flow:
> Client UI Event -> Game Logic creates **TeleportToWorldMapMarker** -> Network Encoder calls `serialize` -> TCP Stream -> Server Network Decoder identifies Packet ID 244 -> Decoder calls `validateStructure` -> Decoder calls `deserialize` -> **TeleportToWorldMapMarker** object passed to Packet Handler -> Server Game Logic executes teleport.

