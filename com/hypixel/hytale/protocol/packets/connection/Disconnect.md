---
description: Architectural reference for the Disconnect packet, a core component of the Hytale connection protocol.
---

# Disconnect

**Package:** com.hypixel.hytale.protocol.packets.connection
**Type:** Transient

## Definition
```java
// Signature
public class Disconnect implements Packet {
```

## Architecture & Concepts
The Disconnect class is a Data Transfer Object (DTO) representing a specific message within the Hytale network protocol. Its sole purpose is to encapsulate the data required to inform a remote peer that a connection is being terminated, along with a reason and a classification for the termination.

As an implementation of the Packet interface, it is a fundamental building block of the network layer. It does not contain any logic itself; instead, it serves as a structured data container that is created, serialized, transmitted, deserialized, and then processed by dedicated network handlers. Its design prioritizes wire-format efficiency, utilizing a bitmask for nullable fields and variable-length integers to minimize bandwidth consumption.

This class sits at the boundary between high-level game logic (e.g., a server deciding to kick a player) and the low-level network transport layer (e.g., a Netty pipeline).

### Lifecycle & Ownership
-   **Creation:** An instance is created under two distinct circumstances:
    1.  **Outbound:** Instantiated directly by game logic on the sending side (client or server) when a disconnection needs to be initiated. For example: `new Disconnect("Kicked for idling", DisconnectType.Kick)`.
    2.  **Inbound:** Instantiated by the protocol decoding pipeline via the static `deserialize` factory method when a corresponding byte stream (with Packet ID 1) is received from the network.
-   **Scope:** Extremely short-lived. An instance exists only for the brief period it takes to be serialized into a network buffer or to be processed by a packet handler after deserialization. It is a classic transient object.
-   **Destruction:** The object is eligible for garbage collection immediately after its use. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
-   **State:** The class is **mutable**. Its public fields, `reason` and `type`, can be modified after construction. This is a design choice for convenience, but it carries significant implications for concurrent access.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without explicit, external synchronization. It is designed to be created, processed, and discarded within the confines of a single thread, typically a Netty I/O worker thread.

    **WARNING:** Modifying a Disconnect instance from one thread while another thread is serializing or reading it will result in undefined behavior, data corruption, or runtime exceptions.

## API Surface
The public API is divided between instance methods for outbound packet construction and static methods for inbound packet decoding.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (1). |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided Netty byte buffer. N is the length of the reason string. |
| computeSize() | int | O(N) | Calculates the final serialized size in bytes without performing the write. |
| deserialize(ByteBuf, int) | static Disconnect | O(N) | Decodes bytes from a buffer into a new Disconnect instance. Throws ProtocolException on malformed data. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a read-only check to validate if the buffer contains a structurally sound Disconnect packet. |

## Integration Patterns

### Standard Usage
The class is used as part of a standard network event pipeline. The sending endpoint constructs and populates the object, then passes it to the network system for serialization. The receiving endpoint receives a fully-formed object from the deserialization layer.

```java
// Example: Server-side logic to kick a player
// NOTE: networkManager is a hypothetical service
Disconnect kickPacket = new Disconnect("You have been kicked.", DisconnectType.Kick);
networkManager.sendPacket(playerConnection, kickPacket);

// Example: Client-side packet handler
public void handleDisconnect(Disconnect packet) {
    // This packet is provided by the network layer, not created with 'new'
    uiManager.showErrorScreen("Disconnected", packet.reason);
    game.returnToMainMenu();
}
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not hold onto and reuse Disconnect instances. They are lightweight and should be created for each unique disconnection event to prevent state leakage.
-   **Manual Deserialization:** Do not attempt to parse the byte stream manually. Always use the provided static `deserialize` method, which correctly handles the protocol's specific encoding rules (e.g., null bit fields, VarInts).
-   **Cross-Thread Modification:** Never create a Disconnect packet on one thread and pass it to the network thread for sending if other threads might still hold a reference to it. This is a direct path to a race condition.

## Data Pipeline
The Disconnect object is a payload that travels through the network stack. Its representation changes from an in-memory object to a compact byte array and back.

> **Outbound Flow:**
> Game Logic → `new Disconnect(...)` → Network Encoder → **serialize()** → Raw Bytes → TCP/UDP Socket

> **Inbound Flow:**
> TCP/UDP Socket → Raw Bytes → Packet Decoder → **deserialize()** → `Disconnect` Instance → Packet Handler → Game Logic

