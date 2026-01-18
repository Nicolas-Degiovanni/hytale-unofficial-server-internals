---
description: Architectural reference for BuilderToolPasteClipboard
---

# BuilderToolPasteClipboard

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Data Transfer Object (DTO) / Transient

## Definition
```java
// Signature
public class BuilderToolPasteClipboard implements Packet {
```

## Architecture & Concepts
The BuilderToolPasteClipboard class is a network packet definition within the Hytale Protocol Layer. It serves as a simple, high-performance Data Transfer Object designed for a single purpose: to transmit a "paste from clipboard" command from a game client to the server.

This packet encapsulates the 3D world coordinates where a build tool paste operation should occur. As a concrete implementation of the Packet interface, it is discoverable by the protocol's packet registry, which uses the static PACKET_ID field (407) to map incoming byte streams to this class for deserialization.

Its design prioritizes raw performance and minimal overhead. The structure is fixed at 12 bytes, uncompressed, and contains only the essential coordinate data. This makes it trivial for the network layer to validate, decode, and dispatch for processing by the relevant server-side game logic. It represents a command, not a state synchronization object.

## Lifecycle & Ownership
- **Creation:**
    - **Client-Side:** Instantiated by the client's Builder Tool systems in response to a direct player action (e.g., right-clicking with a paste tool). The object is populated with the target block coordinates.
    - **Server-Side:** Instantiated by the server's network protocol dispatcher. When a raw network buffer with packet ID 407 is received, the static `deserialize` factory method is invoked to construct the object from the byte stream.

- **Scope:** This object is highly transient and has an extremely short lifespan.
    - On the client, it exists only for the duration of the serialization process before being written to the network channel.
    - On the server, it exists from the moment of deserialization until the corresponding command handler has finished processing it.

- **Destruction:** The object holds no native resources and is managed entirely by the Java Garbage Collector. Once all references are dropped after processing, it is eligible for collection.

## Internal State & Concurrency
- **State:** The internal state is mutable and consists of three public integer fields: x, y, and z. This direct field access pattern is a deliberate design choice to eliminate method call overhead in performance-critical code, a common practice in network DTOs. The state is a pure data representation of a world coordinate.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and processed within a single-threaded context, such as a Netty event loop thread or a main game logic thread.
    - **WARNING:** Concurrent modification of its public fields from multiple threads will lead to data corruption and unpredictable behavior. Do not share instances of this packet across threads without external synchronization, which is itself an anti-pattern.

## API Surface
The public API is focused on serialization, deserialization, and metadata for the protocol layer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf buf) | void | O(1) | Serializes the x, y, and z coordinates into the provided buffer using Little Endian byte order. |
| deserialize(ByteBuf buf, int offset) | static BuilderToolPasteClipboard | O(1) | Factory method to construct a new instance by reading 12 bytes from a buffer at a given offset. |
| validateStructure(ByteBuf buffer, int offset) | static ValidationResult | O(1) | Performs a preliminary bounds check to ensure the buffer contains enough data for a valid packet. |
| computeSize() | int | O(1) | Returns the constant size of the packet in bytes, which is always 12. |
| getId() | int | O(1) | Returns the unique network identifier for this packet type, which is always 407. |

## Integration Patterns

### Standard Usage
This packet is not intended for direct use by most game feature developers. It is handled by the low-level client input systems and server-side command handlers. A typical server-side handler would receive the deserialized packet and execute the command.

```java
// Example of a server-side command handler consuming the packet
public class PasteCommandHandler implements PacketHandler<BuilderToolPasteClipboard> {

    @Override
    public void handle(PlayerConnection connection, BuilderToolPasteClipboard packet) {
        Player player = connection.getPlayer();
        World world = player.getWorld();

        // CRITICAL: The server must always validate the coordinates and player permissions
        // before modifying the world state.
        if (world.isWithinBounds(packet.x, packet.y, packet.z) && player.hasBuildPermission()) {
            ClipboardService clipboard = player.getClipboardService();
            clipboard.pasteAt(world, packet.x, packet.y, packet.z);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Object Re-use:** Do not modify and re-send the same packet instance. The network layer may operate on the object asynchronously. Always create a new instance for each distinct command.
- **Manual Serialization:** Do not attempt to write the integer fields to a buffer manually. Always use the `serialize` method to guarantee the correct byte order (Little Endian) and field layout expected by the server.
- **Stateful Logic:** Do not add business logic or stateful behavior to this class. It is strictly a data container. All processing logic belongs in dedicated handler or service classes.

## Data Pipeline
The data flow for this packet is unidirectional, from the client to the server, representing a user command.

> Flow:
> Client Input System -> **BuilderToolPasteClipboard (new instance)** -> Client Network Encoder -> Netty ByteBuf -> Server Network Decoder -> Packet Dispatcher (lookup by ID 407) -> **BuilderToolPasteClipboard.deserialize()** -> Server Command Handler -> World Modification Service

