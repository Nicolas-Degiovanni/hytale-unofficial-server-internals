---
description: Architectural reference for BuilderToolState
---

# BuilderToolState

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class BuilderToolState {
```

## Architecture & Concepts
BuilderToolState is a data transfer object (DTO) that represents the complete state of a player's selected in-game building tool. It is a critical component of the client-server communication protocol for creative and building-centric game modes.

Its primary architectural purpose is to serve as a serializable snapshot of a tool's configuration, including its type, brush settings, and any dynamic arguments. The class is designed for high-performance network transmission, employing a custom binary format rather than a more verbose format like JSON or XML.

The binary layout is a key concept. It consists of two main parts:
1.  **Fixed-Size Header (14 bytes):** This block contains a null-bit field, a boolean flag, and three integer offsets. The null-bit field is a bitmask used to efficiently track which of the nullable, variable-sized fields (id, brushData, args) are present in the payload.
2.  **Variable-Size Data Block:** This block contains the actual data for the fields indicated by the null-bit mask. The offsets in the header point to the start of each data segment within this block, allowing for fast, non-sequential access and validation without parsing the entire object.

This design minimizes packet size and allows the receiving end to perform cheap structural validation before committing to a full, potentially expensive, deserialization process.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderToolState is created under two circumstances:
    1.  **Client-Side:** Instantiated by game logic when a player selects or modifies a building tool via the user interface. The object is populated with the new configuration.
    2.  **Server-Side (or Receiving Peer):** Instantiated within the network protocol layer by the static factory method `deserialize`. This occurs when a packet containing tool state data is received and needs to be parsed into a usable object.
- **Scope:** This object is ephemeral and has a very short-lived scope. It is intended to exist only for the duration of a single transaction: being built, serialized into a packet, and sent; or being deserialized from a packet and immediately consumed by game logic.
- **Destruction:** As a standard Java object with no external resource handles, it is managed by the garbage collector. It becomes eligible for collection as soon as it is no longer referenced, which is typically immediately after the network or game logic operation completes.

## Internal State & Concurrency
- **State:** The object's state is entirely mutable. All fields are public, allowing for direct modification after construction. It is a simple data container and does not cache any information or manage complex internal transitions.
- **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded access, typically confined to either the main game thread (for state manipulation) or a Netty network thread (for serialization and deserialization). Concurrent modification from multiple threads without external locking will result in a corrupted state and is strictly unsupported. The serialization methods directly manipulate `ByteBuf` indices, which is an inherently unsafe operation in a concurrent context.

## API Surface
The public API is dominated by static methods for serialization, deserialization, and validation, reflecting its role as a protocol data structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static BuilderToolState | O(N) | Constructs a BuilderToolState object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the custom binary protocol. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total byte size of a serialized BuilderToolState within a buffer without performing a full deserialization. |
| computeSize() | int | O(N) | Calculates the byte size the current object instance will occupy when serialized. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a pre-flight check on a buffer to ensure the data represents a structurally valid BuilderToolState. **WARNING:** This should always be called on untrusted input before attempting to deserialize. |
| clone() | BuilderToolState | O(N) | Creates a deep copy of the object and its contained data structures. |

## Integration Patterns

### Standard Usage
The class is almost never used in isolation. It is embedded within a larger network packet structure. The typical flow involves a network handler deserializing the state and passing it to a game system for processing.

```java
// On a network thread, after a packet has been decoded
public void handlePacket(SomePacketContainer container) {
    ByteBuf payload = container.getPayload();

    // Validate before deserializing to prevent protocol errors from untrusted data
    ValidationResult result = BuilderToolState.validateStructure(payload, 0);
    if (!result.isValid()) {
        // Disconnect client or log error
        return;
    }

    BuilderToolState toolState = BuilderToolState.deserialize(payload, 0);

    // Pass the state object to the main game thread for processing
    gameLogic.updatePlayerTool(player, toolState);
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not hold references to BuilderToolState instances in long-lived components. They represent a point-in-time snapshot and should be discarded after use. Storing them can lead to bugs related to stale state.
- **Manual Serialization/Deserialization:** Do not attempt to read or write the binary format manually. The layout is complex, involving null-bit masks and relative offsets. Always use the provided `serialize` and `deserialize` methods to ensure correctness.
- **Ignoring Validation on Server:** Never call `deserialize` on a server with data received from a client without first calling `validateStructure`. A malicious client could send a malformed payload (e.g., with invalid lengths or offsets) that could cause exceptions and potentially crash a server network thread if not handled.

## Data Pipeline
The BuilderToolState acts as the data payload during the transfer of tool information between client and server.

> **Client to Server Flow:**
> Player UI Interaction → Game Logic creates `BuilderToolState` → `serialize()` → Packet Buffer → Network Layer → Server

> **Server to Client Flow:**
> Server Network Layer → Packet Buffer → `deserialize()` → `BuilderToolState` object → Game Logic updates Client State

