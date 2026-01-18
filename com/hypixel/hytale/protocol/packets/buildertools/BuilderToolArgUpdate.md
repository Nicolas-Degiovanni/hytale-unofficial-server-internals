---
description: Architectural reference for BuilderToolArgUpdate
---

# BuilderToolArgUpdate

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient Data Model

## Definition
```java
// Signature
public class BuilderToolArgUpdate implements Packet {
```

## Architecture & Concepts
The BuilderToolArgUpdate class is a Data Transfer Object (DTO) that represents a network message within the Hytale protocol. Its primary function is to encapsulate a state change for a single argument within the in-game builder tools. This packet is sent from the client to the server when a user modifies a setting, such as changing a color, selecting a material, or adjusting a brush size.

Architecturally, this class sits at the boundary between the game logic and the low-level network layer. It provides a structured, object-oriented representation of what is fundamentally a binary message format.

The binary layout is highly optimized for performance and size, employing a hybrid structure:
1.  **Fixed-Size Block:** A 22-byte header containing metadata like the token, section, slot, and a bitmask for nullable fields. This allows for extremely fast initial parsing.
2.  **Variable-Size Block:** A subsequent data region for variable-length fields, such as the string-based id and value. The fixed-size block contains offsets pointing to the start of each variable field within this region.

This design allows the receiver to quickly validate the packet structure and even access fixed-size fields without needing to parse the entire variable-data payload.

### Lifecycle & Ownership
-   **Creation:**
    -   **Sending Peer (Client):** Instantiated directly via its constructor (`new BuilderToolArgUpdate(...)`) within the UI or game logic layer in response to a user action. The object is then populated with the relevant data.
    -   **Receiving Peer (Server):** Never instantiated with its constructor. Instead, it is created by the protocol's packet dispatcher, which invokes the static `deserialize` factory method on an incoming Netty ByteBuf.

-   **Scope:** The lifecycle of a BuilderToolArgUpdate instance is exceptionally brief. It is designed to be a short-lived message container. On the sending side, it exists only long enough to be serialized into a buffer. On the receiving side, it exists only until its data has been consumed by a packet handler, after which it becomes immediately eligible for garbage collection.

-   **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or manual cleanup procedures required.

## Internal State & Concurrency
-   **State:** The class is **mutable**, with public fields that can be modified after instantiation. This is a deliberate design choice for DTOs, allowing them to be constructed and populated efficiently before serialization. It is a pure data container and does not cache any information or maintain connections to other systems.

-   **Thread Safety:** This class is **not thread-safe**. It is intended for use within a single-threaded context, such as a Netty event loop or a main game thread. Sharing an instance across multiple threads without external synchronization mechanisms will result in data corruption and undefined behavior. The serialization and deserialization methods operate directly on non-thread-safe ByteBuf instances and must not be invoked concurrently on the same buffer.

## API Surface
The public contract is dominated by serialization and deserialization logic, which is the core responsibility of a packet class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static BuilderToolArgUpdate | O(N) | Constructs a new instance by reading and parsing binary data from the provided buffer. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into a binary format in the provided buffer. This is the primary method for outbound packets. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a security and integrity check on the buffer's data without full deserialization. Crucial for rejecting invalid packets early. |
| computeSize() | int | O(N) | Calculates the total number of bytes required to serialize the object. Useful for pre-allocating buffers. |
| getId() | int | O(1) | Returns the static packet identifier (400), used by the protocol dispatcher to route incoming data to the correct deserializer. |

## Integration Patterns

### Standard Usage
The class is used exclusively by the network protocol layer and the game logic responsible for builder tools. Application-level code should never interact with the serialization methods directly.

**Sending a Packet (Client-Side Logic):**
```java
// 1. A user interacts with a builder tool UI element.
// 2. The game logic creates and populates the packet.
BuilderToolArgUpdate packet = new BuilderToolArgUpdate(
    currentSessionToken,
    TOOL_SECTION_BRUSH,
    SLOT_PRIMARY_COLOR,
    BuilderToolArgGroup.Tool,
    "color_hex",
    "#FF0000"
);

// 3. The packet is handed to the network service to be sent.
networkChannel.sendPacket(packet); // The service will call packet.serialize() internally.
```

**Receiving a Packet (Server-Side Handler):**
```java
// The network layer has already called BuilderToolArgUpdate.deserialize()
// and is now routing the object to a registered handler.
public void handleBuilderToolUpdate(BuilderToolArgUpdate packet) {
    Player player = getPlayerByToken(packet.token);
    if (player != null) {
        player.getBuilderToolState().updateArgument(
            packet.section,
            packet.slot,
            packet.id,
            packet.value
        );
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage:** Do not retain instances of BuilderToolArgUpdate in caches or as part of a long-lived game state. They represent a point-in-time event, not a persistent entity. Copy its data into your own state objects instead.
-   **Manual Serialization/Deserialization:** Application logic should never call `serialize` or `deserialize`. This is the sole responsibility of the network protocol engine. Doing so violates the separation of concerns and couples game logic to the binary protocol format.
-   **Object Reuse:** Do not modify and resend the same packet instance. While technically possible, it is not idiomatic and can lead to subtle bugs if a reference is held elsewhere. Always create a new instance for each new message.

## Data Pipeline
The flow of data encapsulated by this class is unidirectional from client to server.

> **Flow: Client to Server**
> User Input (e.g., clicks a color) -> Client Game Logic -> **new BuilderToolArgUpdate()** -> Network Layer calls **serialize()** -> TCP/IP Stream -> Server Network Layer -> Packet Dispatcher calls **deserialize()** -> **BuilderToolArgUpdate instance** -> Server Game Logic Handler -> Server-side Player State Updated
---

