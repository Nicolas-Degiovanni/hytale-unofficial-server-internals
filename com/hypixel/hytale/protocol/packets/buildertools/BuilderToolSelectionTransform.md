---
description: Architectural reference for BuilderToolSelectionTransform
---

# BuilderToolSelectionTransform

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Data Transfer Object (Packet)

## Definition
```java
// Signature
public class BuilderToolSelectionTransform implements Packet {
```

## Architecture & Concepts
The BuilderToolSelectionTransform class is a network packet, a specialized Data Transfer Object designed for client-server communication. It does not contain business logic; its sole purpose is to encapsulate the state of a complex in-game building operation and transmit it from the game client to the server for execution.

This packet represents a single, atomic "transform selection" command initiated by a player using the builder tools. It contains all necessary parameters for the server to replicate the operation, including a 4x4 transformation matrix for rotation and scaling, the initial selection volume, and a set of boolean flags that dictate the command's behavior (e.g., cut versus copy).

The design prioritizes network efficiency and protocol robustness. Serialization relies on a compact bitmask to represent nullable fields, minimizing payload size. Furthermore, the class includes static validation and size-limiting constants (MAX_SIZE) to protect the server from malformed or malicious packets that could lead to resource exhaustion or buffer overflow vulnerabilities.

## Lifecycle & Ownership
- **Creation:** An instance is created on the client-side in response to a player finalizing a builder tool action. The client's input and UI systems gather the relevant state (selection bounds, transformation matrix) and construct the packet. On the server, a new instance is created by the protocol decoding pipeline when it receives the corresponding raw data from a client's network stream.

- **Scope:** This object is **transient** and extremely short-lived. Its lifecycle is confined to the duration of a single network transaction. On the client, it exists only long enough to be serialized and sent. On the server, it exists only from the moment of deserialization until it is consumed by a packet handler.

- **Destruction:** The object becomes eligible for garbage collection immediately after its data has been processed by the server-side game logic. It holds no persistent state and is not registered with any service locator or registry.

## Internal State & Concurrency
- **State:** The object's state is fully **mutable** via its public fields. This design is intentional for a DTO, allowing for easy construction and population before serialization. The state represents a complete snapshot of a single builder tool operation.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and read within a single-threaded context. On the server, a packet is typically deserialized on a Netty I/O thread and then passed to the main game-loop thread via a queue for safe processing.

    **Warning:** Concurrent modification or access from multiple threads will lead to race conditions and undefined behavior. All interactions with an instance of this packet on the server must be synchronized with the main game tick.

## API Surface
The public contract is dominated by serialization and protocol-related utility methods. Direct field access is used for data manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided Netty buffer for network transmission. Throws ProtocolException on validation failure. |
| deserialize(ByteBuf, int) | static BuilderToolSelectionTransform | O(N) | Decodes a new instance from a raw Netty buffer. This is the primary entry point for server-side creation. Throws ProtocolException on validation failure. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a lightweight, pre-deserialization check on a raw buffer to ensure it is well-formed and within size limits. Critical for server security. |
| computeSize() | int | O(1) | Calculates the exact number of bytes this object will consume when serialized. |

*Complexity O(N) refers to the number of elements in the transformationMatrix array.*

## Integration Patterns

### Standard Usage
This packet is never manually instantiated by game-logic developers. It is created by the client's input system and consumed by the server's packet handling system. A typical server-side handler would process it as follows.

```java
// Example of a server-side PacketHandler
public class BuilderToolPacketHandler {

    public void handleSelectionTransform(BuilderToolSelectionTransform packet) {
        // This method is assumed to be called on the main game thread.
        World world = GameContext.getWorld();
        Player player = GameContext.getPlayerFromPacket(packet);

        // Validate player permissions before proceeding.
        if (!player.hasPermission("world.edit")) {
            return; // Discard packet
        }

        // Pass the packet data to the world modification service.
        // The service will execute the actual block transformation.
        world.getBuilderService().applyTransform(
            player,
            packet.initialSelectionMin,
            packet.initialSelectionMax,
            packet.transformationMatrix,
            packet.cutOriginal
        );
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not hold a reference to a packet instance after it has been processed. These objects are single-use messages. Reusing them can lead to processing stale or incorrect data.

- **Manual Serialization:** Never attempt to read or write the packet's data to a buffer manually. The serialization format is complex, involving a null-field bitmask and variable-length integers. Always use the provided `serialize` and `deserialize` methods to ensure protocol compliance.

- **Cross-Thread Processing:** Do not access the packet's fields from a thread other than the one designated for packet processing (typically the main game thread). If data is needed elsewhere, copy it to a thread-safe structure first.

## Data Pipeline
The flow of this data object is unidirectional from client to server. It represents a command, not a query.

> **Flow:**
> Client Input System -> **BuilderToolSelectionTransform (Instance created)** -> Client Network Encoder -> TCP/IP Stream -> Server Network Decoder -> **BuilderToolSelectionTransform (Instance re-created)** -> Server Packet Handler -> World Edit Service

