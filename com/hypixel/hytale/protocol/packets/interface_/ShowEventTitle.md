---
description: Architectural reference for ShowEventTitle
---

# ShowEventTitle

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient

## Definition
```java
// Signature
public class ShowEventTitle implements Packet {
```

## Architecture & Concepts

The ShowEventTitle class is a Data Transfer Object (DTO) that represents a specific network message within the Hytale protocol. It is not a service or a manager; its sole purpose is to model the data required for a server to instruct a client to display a prominent, animated title on the user's screen. This is typically used for significant in-game events, such as completing a quest or entering a new zone.

Architecturally, this class is a key component of the client-server communication layer, acting as a structured contract for a specific UI command. Its design is heavily optimized for network performance, utilizing a custom binary serialization format.

The serialization strategy is notable for its efficiency. A packet is divided into two main sections:
1.  **Fixed-Size Block:** A 26-byte header containing primitive data types like floats, booleans, and a crucial one-byte bitmask named nullBits. This block also contains integer offsets pointing to the location of variable-sized data.
2.  **Variable-Size Block:** This section follows the fixed block and contains data like strings and nested complex objects (FormattedMessage). Its layout is determined by the offsets in the fixed block.

This design allows for extremely fast reads of fixed-size data and provides a lookup mechanism for variable data without needing to parse the entire packet sequentially. The nullBits field acts as a compact way to signal the presence or absence of optional fields, saving network bandwidth.

## Lifecycle & Ownership

-   **Creation:**
    -   **Server-Side (Sending):** Instantiated via its constructor (`new ShowEventTitle(...)`) by game logic that needs to trigger a title display for a player. The object is then passed to the network layer for serialization.
    -   **Client-Side (Receiving):** Instantiated by the protocol decoding pipeline. When a raw network buffer is identified with Packet ID 214, the static `deserialize` factory method is invoked to construct a ShowEventTitle object from the binary data.

-   **Scope:** The lifecycle of a ShowEventTitle instance is extremely brief and transactional. It exists only to be serialized into a buffer or to be deserialized from one and immediately processed by a packet handler. It does not hold state beyond the scope of a single network event.

-   **Destruction:** The object is eligible for garbage collection as soon as it has been written to the network buffer (server) or after its data has been consumed by a client-side handler. There are no managed resources requiring explicit cleanup.

## Internal State & Concurrency

-   **State:** The class is a mutable container for data. All of its fields are public and directly accessible, which is characteristic of a DTO designed for high-performance, internal-only use within the protocol layer. It holds no references to external services or systems.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and passed to the network layer from a single thread (e.g., the main server tick loop). On the client, it is deserialized and processed on a single Netty I/O thread.

    **WARNING:** Any attempt to read or modify a ShowEventTitle instance from multiple threads without external synchronization will result in race conditions and unpredictable behavior, potentially leading to corrupted network data or client-side rendering errors.

## API Surface

The primary interface is composed of static methods for validation and deserialization, and instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ShowEventTitle | O(N) | Constructs a new object by reading from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary format. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a security and integrity check on the buffer without full deserialization. Critical for preventing protocol exploits. |
| computeSize() | int | O(N) | Calculates the number of bytes required to serialize the current object state. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Measures the size of a packet within a buffer without deserializing its contents. |

## Integration Patterns

### Standard Usage

The primary integration point is the network protocol handler. A central dispatcher routes incoming buffers based on Packet ID to the appropriate deserializer.

**Server-Side (Sending a Packet):**
```java
// Executed within server game logic
FormattedMessage title = new FormattedMessage("Zone Discovered");
FormattedMessage subtitle = new FormattedMessage("The Whispering Valley");

ShowEventTitle packet = new ShowEventTitle(
    0.5f, // fadeInDuration
    0.5f, // fadeOutDuration
    3.0f, // duration
    "ui.icons.discovery", // icon
    true, // isMajor
    title,
    subtitle
);

// The network system will later call packet.serialize(buffer)
player.getConnection().sendPacket(packet);
```

**Client-Side (Handling a Packet):**
```java
// Within a client-side packet handler, after the packet has been deserialized
public void handle(ShowEventTitle packet) {
    // The UI system consumes the packet data to render the title
    uiManager.getEventTitleSystem().displayTitle(
        packet.primaryTitle,
        packet.secondaryTitle,
        packet.icon,
        packet.fadeInDuration,
        packet.duration,
        packet.fadeOutDuration
    );
}
```

### Anti-Patterns (Do NOT do this)

-   **State Reuse:** Do not retain and modify a ShowEventTitle instance to send multiple, slightly different messages. This is error-prone. Always construct a new, clean instance for each distinct message to ensure data integrity.
-   **Client-Side Instantiation:** Clients should never create instances of this packet using `new`. Its purpose is to receive and interpret commands from the server, not to send them.
-   **Unsafe Deserialization:** Never call `deserialize` on a buffer that has not first been cleared by `validateStructure`. Doing so exposes the application to potential crashes or exploits from malformed packets sent by a malicious actor.

## Data Pipeline

The flow of ShowEventTitle data is unidirectional from server to client.

> **Server Flow:**
> Game Event Trigger -> **new ShowEventTitle()** -> Network Channel -> serialize() -> Raw TCP/UDP Stream

> **Client Flow:**
> Raw TCP/UDP Stream -> Netty ByteBuf -> Packet Decoder (identifies ID 214) -> **ShowEventTitle.deserialize()** -> Packet Handler -> UI Rendering System -> Screen Display

