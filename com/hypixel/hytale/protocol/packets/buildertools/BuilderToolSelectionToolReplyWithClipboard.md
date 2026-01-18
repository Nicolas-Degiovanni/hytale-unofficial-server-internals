---
description: Architectural reference for BuilderToolSelectionToolReplyWithClipboard
---

# BuilderToolSelectionToolReplyWithClipboard

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class BuilderToolSelectionToolReplyWithClipboard implements Packet {
```

## Architecture & Concepts
The BuilderToolSelectionToolReplyWithClipboard is a network packet that functions as a server-to-client data container. Its primary role is to transmit a large volume of world data, specifically a "clipboard" of block and fluid states, to a client. This is typically sent in response to a player action using an in-game builder tool, such as a copy or selection command.

This packet is a critical component of the creative and building systems. It is designed for high performance and network efficiency, not for long-term state management. The binary layout is heavily optimized, employing a custom serialization format that includes:
- A **null bit field**: A single byte at the start of the payload acts as a bitmask to indicate which of the nullable array fields (`blocksChange`, `fluidsChange`) are present in the data stream. This avoids wasting bytes for null data.
- **Relative offsets**: The packet header contains integer offsets pointing to the start of each variable-sized data block within the payload. This allows the deserializer to jump directly to the data without sequential parsing.
- **VarInt encoding**: Array lengths are encoded using the variable-length VarInt format to save space for smaller selections.

This class is a pure data structure. It contains no logic related to game state, rendering, or player interaction. Its sole responsibility is to accurately represent a collection of world changes for network transport.

## Lifecycle & Ownership
- **Creation:**
    - **Server-side:** Instantiated by a game logic handler when a player's builder tool action necessitates sending a clipboard. The server populates the `blocksChange` and `fluidsChange` arrays from its authoritative world state.
    - **Client-side:** Instantiated exclusively by the network protocol layer when a raw byte buffer with Packet ID 411 is received. The static `deserialize` method is the designated factory for client-side instances.

- **Scope:** This object is ephemeral and has an extremely short lifespan.
    - On the server, it exists only for the duration of the serialization process before being discarded.
    - On the client, it exists from the moment of deserialization until its data is consumed by the relevant game system (e.g., the Builder Tool UI). It is not cached or persisted.

- **Destruction:** The object becomes eligible for garbage collection as soon as its contents have been processed. The underlying Netty ByteBuf is managed by the network framework's memory pool.

## Internal State & Concurrency
- **State:** The object's state is defined entirely by its two public fields: `blocksChange` and `fluidsChange`. It is a mutable container designed to be populated once and then read. It holds no references to external services or systems.

- **Thread Safety:** **This class is not thread-safe.** It is a simple data holder and provides no internal synchronization. It must be created, serialized, deserialized, and processed within a single, well-defined thread context, such as a Netty I/O thread or the main client game thread. Concurrent access to its arrays from multiple threads will result in data corruption and undefined behavior.

## API Surface
The public contract is dominated by static methods for serialization, deserialization, and validation, reflecting its role as a network DTO.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | BuilderToolSelectionToolReplyWithClipboard | O(N) | **[Static]** Deserializes the packet from a ByteBuf. Throws ProtocolException on malformed data. This is the primary entry point on the client. |
| serialize(buf) | void | O(N) | Serializes the object's state into a ByteBuf. This is the primary entry point on the server. |
| validateStructure(buffer, offset) | ValidationResult | O(N) | **[Static]** Performs a low-cost check on a buffer to validate offsets and array bounds without full deserialization. Critical for preventing DoS attacks. |
| computeBytesConsumed(buf, offset) | int | O(N) | **[Static]** Calculates the total size of a serialized packet within a buffer by following its internal offsets. |
| computeSize() | int | O(N) | Calculates the required buffer size for serialization based on the current state of the internal arrays. |

*N = Total number of BlockChange and FluidChange elements.*

## Integration Patterns

### Standard Usage
The packet is handled by the client's network layer. A handler identifies the packet by its ID, deserializes it, and dispatches the resulting object to the appropriate game system for processing.

```java
// Example client-side network handler
// byteBuf contains the raw network data for this packet

BuilderToolSelectionToolReplyWithClipboard clipboardData = BuilderToolSelectionToolReplyWithClipboard.deserialize(byteBuf, 0);

// Pass the data to the builder tool system
BuilderToolSystem builderTool = game.getBuilderToolSystem();
builderTool.loadClipboard(clipboardData.blocksChange, clipboardData.fluidsChange);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation on Client:** Never use `new BuilderToolSelectionToolReplyWithClipboard()` on the client. Packets from the server must only be created via the `deserialize` method to ensure they correctly reflect the network data.
- **State Mutation After Deserialization:** Do not modify the `blocksChange` or `fluidsChange` arrays after the packet has been deserialized. The object should be treated as immutable upon receipt to prevent inconsistent state.
- **Ignoring Validation:** In any context where the network stream could be compromised, failing to use `validateStructure` before `deserialize` can expose the client to buffer overflow exceptions or excessive memory allocation caused by maliciously crafted packets.

## Data Pipeline
The flow of this data is unidirectional from server to client.

> Flow:
> Server World State -> **BuilderToolSelectionToolReplyWithClipboard (Server)** -> `serialize()` -> Network Byte Stream -> **BuilderToolSelectionToolReplyWithClipboard (Client)** -> Builder Tool System -> Client UI / Local Cache Update

