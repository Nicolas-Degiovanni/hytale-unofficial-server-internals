---
description: Architectural reference for MouseInteraction
---

# MouseInteraction

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Transient

## Definition
```java
// Signature
public class MouseInteraction implements Packet {
```

## Architecture & Concepts
The MouseInteraction class is a Data Transfer Object (DTO) that represents a single, discrete input event from the client's mouse. It is a fundamental component of the client-to-server (C2S) network protocol, encapsulating all possible data related to a mouse action, such as button clicks, movement, and interactions with the game world.

This class is not a service or manager; it is a pure data container, a *message*. Its design is heavily optimized for network efficiency and low-latency processing, eschewing standard Java serialization in favor of a custom, high-performance binary format managed directly on Netty's ByteBuf.

Key architectural features include:
*   **Custom Binary Layout:** The packet structure is rigidly defined by constants like FIXED_BLOCK_SIZE and VARIABLE_BLOCK_START. This allows for extremely fast, zero-reflection serialization and deserialization.
*   **Null Field Optimization:** A bitmask, `nullBits`, is used as the first byte of the payload. Each bit corresponds to a nullable field, indicating its presence or absence. This minimizes payload size by omitting data for optional fields.
*   **Offset-Based Pointers:** For variable-length data like `itemInHandId`, the fixed block of the packet contains integer offsets pointing to the data's location in the variable-data section. This is a common and highly effective pattern in game networking protocols for combining predictable and unpredictable data sizes.

## Lifecycle & Ownership
- **Creation:** An instance of MouseInteraction is created by the client-side input handling system whenever the player performs a relevant mouse action. The client's game loop populates a new instance with the current state (e.g., cursor position, button state, targeted block).
- **Scope:** The object's lifetime is exceptionally short.
    - On the **client**, it exists only long enough to be created, populated, and passed to the network layer for serialization.
    - On the **server**, it is instantiated by the network protocol decoder, processed by a single packet handler, and then immediately becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. Due to its transient nature, it places very low pressure on the GC.

## Internal State & Concurrency
- **State:** MouseInteraction is a mutable container of plain data. Its fields are public and can be modified after instantiation. However, by convention, it should be treated as immutable once it has been submitted to a processing queue or network channel.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, serialized, and deserialized within a single thread, typically a client's main thread or a server's Netty event loop thread. Concurrent modification from multiple threads will result in a corrupted or invalid network packet. All synchronization must be handled externally.

## API Surface
The primary contract is not for general application use but for the network protocol layer itself.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static MouseInteraction | O(N) | Constructs a MouseInteraction object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the custom binary protocol. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a security and integrity check on the raw buffer before deserialization. **CRITICAL** for preventing buffer overflows or DoS attacks from malformed packets. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this object will occupy when serialized. |

## Integration Patterns

### Standard Usage
This class is almost exclusively used by the network layer. A server-side packet handler would receive a fully-formed object from the network decoder.

```java
// Hypothetical server-side packet handler
public class MouseInteractionHandler implements PacketHandler<MouseInteraction> {

    @Override
    public void handle(PlayerConnection connection, MouseInteraction packet) {
        // Validate timestamp to prevent replay attacks
        if (isTimestampInvalid(packet.clientTimestamp)) {
            return;
        }

        // Process the specific interaction type
        if (packet.worldInteraction != null) {
            // Apply block placement or destruction logic
            connection.getPlayer().getWorldInteractor().process(packet.worldInteraction);
        } else if (packet.mouseButton != null) {
            // Handle UI clicks or other non-world interactions
            connection.getPlayer().getInventory().handleClick(packet.mouseButton, packet.activeSlot);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Object Re-use:** Do not attempt to cache and re-use MouseInteraction instances. Their mutable state makes this pattern highly error-prone. Always create a new instance for each distinct user action.
- **Server-Side Instantiation:** Never use `new MouseInteraction()` on the server for game logic. These packets represent client commands and must originate from the `deserialize` method via the network pipeline.
- **Ignoring Validation:** Bypassing `validateStructure` in a custom network layer is a severe security risk. An attacker could craft a packet with invalid offsets or lengths, causing server exceptions or crashes.
- **Asynchronous Modification:** Modifying a MouseInteraction object after it has been passed to the network thread for sending will cause a race condition, leading to a partially updated or corrupted packet being sent.

## Data Pipeline
The MouseInteraction object is a payload that travels through the client-server network stack.

> **Client Flow:**
> User Input (Mouse Click) -> Input Polling System -> **new MouseInteraction()** -> Network Manager Queue -> Netty Event Loop -> `serialize()` -> TCP Socket

> **Server Flow:**
> TCP Socket -> Netty Event Loop -> Packet Framer/Decoder -> `deserialize()` -> **MouseInteraction instance** -> Packet Handler -> Game World Simulation Update

