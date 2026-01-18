---
description: Architectural reference for BuilderToolBrushData
---

# BuilderToolBrushData

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient

## Definition
```java
// Signature
public class BuilderToolBrushData {
```

## Architecture & Concepts
BuilderToolBrushData is a Data Transfer Object (DTO) designed to encapsulate the complete state of a player's in-game building tool. Its primary role is to act as a data container for serialization and deserialization across the network. It represents a snapshot of all configurable brush properties, from its dimensions (width, height) to its material and complex masking rules.

The class is architected for extreme network efficiency, which is evident in its custom binary format. The serialization protocol employs several optimization techniques:

1.  **Nullable Field Bitmask:** A 3-byte bitfield (`nullBits`) at the start of the payload acts as a presence map. Each bit corresponds to a nullable field, indicating whether its data is included in the packet. This avoids transmitting empty or default values, significantly reducing payload size.
2.  **Fixed and Variable Data Blocks:** The serialized structure is split into two sections. A fixed-size header (`FIXED_BLOCK_SIZE`) contains simple, constant-size fields. This is followed by a table of integer offsets that point to the locations of variable-sized fields (like arrays or strings) in a subsequent data block. This hybrid layout provides both fast random access to fixed fields and flexibility for dynamic data.

This object is a fundamental component of the creative mode and world editing systems, allowing the client's tool configuration to be synchronized with the server authoritatively.

### Lifecycle & Ownership
-   **Creation:** An instance is created under two primary circumstances:
    1.  **Client-Side:** When a player modifies their builder tool settings via the user interface. The UI state is collected and used to populate a new BuilderToolBrushData object.
    2.  **Network Deserialization:** When a network packet containing brush data arrives, the static `deserialize` factory method is invoked to construct a new object from the raw bytes in a Netty ByteBuf.
-   **Scope:** This object is short-lived and transactional. It is not designed to be stored or referenced long-term. Its scope is confined to the immediate task of data transfer: either being prepared for sending or being processed immediately upon receipt.
-   **Destruction:** The object is managed by the Java garbage collector and is eligible for cleanup as soon as it is no longer referenced, typically after a network packet has been fully processed.

## Internal State & Concurrency
-   **State:** The object is highly **mutable**. All of its fields are public and directly accessible. This design choice simplifies the process of populating the DTO before serialization and reading from it after deserialization. It is a pure data container with no internal logic governing its state.
-   **Thread Safety:** **This class is not thread-safe.** It is intended for use within a single thread, such as the main game thread for state changes or a Netty network thread for protocol encoding and decoding. Concurrent modification and serialization will result in corrupted data and unpredictable behavior. All synchronization must be handled externally.

## API Surface
The primary contract of this class is its static serialization and validation interface, not its instance fields.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static BuilderToolBrushData | O(N) | Constructs an object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Encodes the object's state into the provided ByteBuf according to the custom binary protocol. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a security and integrity check on the buffer data without full deserialization. Critical for preventing crashes from malformed packets. |
| computeSize() | int | O(M) | Calculates the total byte size the object will occupy when serialized. M is the number of elements in variable-sized arrays. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Measures the size of a serialized object already present in a buffer, allowing a parser to skip it. |

## Integration Patterns

### Standard Usage
This object is almost exclusively used within the network layer, typically as a field within a larger packet object. The packet handler is responsible for invoking serialization and deserialization.

```java
// Example: Client sending an updated brush configuration
BuilderToolBrushData brushData = new BuilderToolBrushData();
brushData.shape = new BuilderToolBrushShapeArg(Shape.SPHERE);
brushData.width = new BuilderToolIntArg(10);
brushData.material = new BuilderToolBlockArg(Material.STONE);

// The data is then embedded in a packet and sent
UpdateBrushPacket packet = new UpdateBrushPacket(brushData);
networkService.sendPacket(packet);

// Example: Server receiving and processing the data
// Inside the packet's deserialization logic...
this.brushData = BuilderToolBrushData.deserialize(buffer, offset);

// Inside the packet handler...
Player targetPlayer = context.getPlayer();
targetPlayer.getBuilderToolSystem().applyBrushData(packet.getBrushData());
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term State:** Do not retain instances of BuilderToolBrushData as part of a player's state. It is a message, not a state container. The data within it should be copied to the authoritative game state objects, and the DTO itself should be discarded.
-   **Concurrent Access:** Do not serialize an instance from a network thread while a game thread is modifying its fields. This is a classic race condition that will lead to sending partially updated, corrupted data.
-   **Skipping Validation:** **CRITICAL:** Never call `deserialize` on a buffer received from a client without first calling `validateStructure`. A malicious or malformed payload could exploit the complex offset calculations to cause buffer over-reads, leading to exceptions that could crash the server's network pipeline.

## Data Pipeline
The flow of this data is unidirectional from the point of change (usually the client) to the authority (usually the server).

> Flow:
> Client UI Interaction -> Game State Update -> **BuilderToolBrushData** instance created -> `serialize()` -> Network Packet -> Netty ByteBuf -> Server Network Layer -> `validateStructure()` -> `deserialize()` -> **BuilderToolBrushData** instance created -> Server-Side Game Logic -> Player State Updated

