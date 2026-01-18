---
description: Architectural reference for CustomUICommand
---

# CustomUICommand

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class CustomUICommand {
```

## Architecture & Concepts

The CustomUICommand class is a Data Transfer Object (DTO) that represents a single, atomic instruction for manipulating a client's user interface. It is a fundamental component of the server-driven UI framework, allowing the server to dynamically modify UI elements on the client without requiring a client-side code update.

This class does not contain any logic itself; it is a pure data container. Its primary role is to define the in-memory representation of a UI command. The associated static methods, such as *serialize* and *deserialize*, form the codec responsible for translating this object to and from its on-the-wire binary format.

The binary serialization format is highly optimized for network performance. It employs a structure consisting of a fixed-size header and a variable-size data block. The header contains a bitmask for null fields and a table of offsets pointing to the location of variable-length string data. This design allows the network layer to perform quick validations and determine the total packet size without needing to parse the entire payload, which is critical for preventing denial-of-service attacks and managing buffer allocation efficiently.

## Lifecycle & Ownership

-   **Creation:**
    -   **Server-Side (Sending):** Instantiated directly via its constructor (`new CustomUICommand(...)`) by server-side game logic responsible for orchestrating UI changes for a client.
    -   **Client-Side (Receiving):** Instantiated by the network protocol layer, which calls the static `deserialize` factory method when an incoming byte stream is identified as a UI command packet.

-   **Scope:** The lifetime of a CustomUICommand instance is extremely short and ephemeral. It exists only long enough to be either serialized into a network buffer for transmission or processed by the client's UI system upon receipt. It is a message, not a persistent entity.

-   **Destruction:** The object becomes eligible for garbage collection immediately after its data has been used. On the server, this is after serialization. On the client, this is after the UI system has processed the command. Ownership is never transferred; its data is read, and the object is discarded.

## Internal State & Concurrency

-   **State:** The object's state is fully mutable through its public fields. It is a simple data aggregate with no internal caching or complex state management.

-   **Thread Safety:** **This class is not thread-safe.** It provides no internal synchronization mechanisms. Instances are intended to be created, populated, and read within a single thread, such as a Netty I/O thread or the main game logic thread. Concurrent modification from multiple threads will result in unpredictable behavior and data corruption. Any cross-thread usage must be managed with external synchronization.

## API Surface

The primary API is the static codec for serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static CustomUICommand | O(N) | Constructs a new CustomUICommand by reading from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary protocol. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs a lightweight check on a buffer to validate offsets and lengths without full deserialization. Critical for security. |
| computeSize() | int | O(N) | Calculates the total byte size the object will occupy when serialized. |

*N = total length of string data*

## Integration Patterns

### Standard Usage

A server-side system constructs and populates the command, which is then embedded in a larger packet and sent. The client-side network layer deserializes it and dispatches it to the UI system.

```java
// Server-side: Creating and preparing a command to send
CustomUICommand command = new CustomUICommand(
    CustomUICommandType.Append,
    "#main_hud.chat_window",
    "<p>Welcome!</p>",
    null
);

// The 'command' object is then passed to the network layer for serialization.
// Client-side code would receive this object after deserialization.
```

### Anti-Patterns (Do NOT do this)

-   **Object Re-use:** Do not attempt to pool or reuse CustomUICommand instances. They are lightweight objects, and re-using them can lead to subtle bugs from stale data. Always create a new instance for each command.
-   **Client-Side Instantiation:** Clients should never create instances of this class with `new`. These commands are authoritative instructions *from* the server. Client-side creation has no effect and violates the server-driven UI architecture.
-   **Multi-Threaded Modification:** Do not pass a CustomUICommand instance between threads for modification without explicit locking. It is unsafe and will lead to race conditions.
-   **Skipping Validation:** Never call `deserialize` on a buffer received from a remote connection without first calling `validateStructure`. Bypassing this check exposes the client to malformed packets that could cause crashes or exploits.

## Data Pipeline

The flow of this data object is linear, from server-side creation to client-side consumption.

> Flow:
> Server Game Logic → `new CustomUICommand()` → Packet Serializer → **Network Byte Stream** → Packet Deserializer → `CustomUICommand.deserialize()` → Client UI System

