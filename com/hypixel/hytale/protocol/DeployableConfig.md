---
description: Architectural reference for DeployableConfig
---

# DeployableConfig

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class DeployableConfig {
```

## Architecture & Concepts
The DeployableConfig class is a specialized Data Transfer Object (DTO) designed for high-performance network communication within the Hytale protocol layer. It represents the configuration of a game object that can be placed or "deployed" in the world, such as a block, a piece of furniture, or a trap.

Its primary architectural role is to serve as a concrete, language-specific representation of a well-defined binary data structure. The class and its static methods provide the contract for serializing game state into a byte stream for network transmission and deserializing a byte stream back into a usable object.

The underlying binary format is custom-designed for efficiency, eschewing more verbose formats like JSON or XML. It employs a fixed-size header followed by a variable-size data block.

-   **Null-Bit Field:** The first byte is a bitmask indicating which of the nullable fields (like *model* or *modelPreview*) are present in the data stream. This avoids wasting bytes for null pointers.
-   **Fixed Block:** A small, constant-size block at the start contains primitive values (*allowPlaceOnWalls*) and, critically, integer offsets.
-   **Variable Block:** Complex, variable-sized objects (*Model*) are written to a data block that follows the fixed block. The offsets in the fixed block point to the starting position of each of these objects, allowing for fast, non-linear access during deserialization.

This structure is highly optimized for the network layer, minimizing both packet size and the CPU cycles required for serialization and deserialization.

### Lifecycle & Ownership
-   **Creation:** An instance is created under two circumstances:
    1.  **Inbound:** The network layer instantiates it via the static `deserialize` factory method when processing an incoming network packet.
    2.  **Outbound:** Game logic instantiates it directly using a constructor (`new DeployableConfig(...)`) to prepare state that needs to be sent over the network.
-   **Scope:** DeployableConfig is a short-lived, transient object. Its lifetime is typically confined to the scope of a single network packet processing cycle or a single game logic update. It is not intended to be a long-lived component.
-   **Destruction:** The object is managed by the Java garbage collector and is eligible for cleanup as soon as it is no longer referenced. No manual memory management is required.

## Internal State & Concurrency
-   **State:** The class is a mutable data container. Its public fields can be directly accessed and modified after instantiation. It holds no hidden state and performs no caching.
-   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization primitives. It is designed to be created, populated, and read within a single thread, such as a Netty I/O thread or the main game loop thread.

    **Warning:** Sharing a DeployableConfig instance across multiple threads without external, user-managed synchronization will result in data corruption and unpredictable application behavior. For multi-threaded use, a deep copy should be created for each thread using the `clone` method.

## API Surface
The public API is focused entirely on serialization, data validation, and state access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static DeployableConfig | O(N) | Constructs an object by reading from a binary buffer. Throws if the buffer is malformed. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the given binary buffer. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a read-only check on a buffer to ensure it contains a structurally valid object. Crucial for security and stability. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N) | Calculates the total size of a serialized object within a buffer without full deserialization. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will occupy when serialized. |
| clone() | DeployableConfig | O(N) | Creates a deep copy of the object and its contained Models. |

*N refers to the total size of the serialized data, including nested objects.*

## Integration Patterns

### Standard Usage
The most common pattern is for the network protocol handler to deserialize the object from a received buffer. Game logic then consumes the data.

```java
// In a network handler, where 'buffer' is a received ByteBuf
// and 'offset' is the start of the DeployableConfig data.

ValidationResult result = DeployableConfig.validateStructure(buffer, offset);
if (!result.isValid()) {
    // Disconnect client or log error; do not proceed.
    throw new ProtocolException("Invalid DeployableConfig: " + result.error());
}

DeployableConfig config = DeployableConfig.deserialize(buffer, offset);

// Pass the config object to the game engine for processing.
gameWorld.handleDeployableConfig(player, config);
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring Validation:** Never call `deserialize` on data from an external source without first calling `validateStructure`. A malformed packet could otherwise trigger uncaught exceptions, leading to a denial-of-service vulnerability.
-   **Shared Mutable State:** Do not retain a reference to a DeployableConfig object in a long-lived component and pass it to other threads. Its mutable nature makes it unsafe for concurrent use. If state must be shared, clone the object first.
-   **Manual Serialization:** Do not attempt to manually read or write the binary format. The internal layout is an implementation detail. Always use the provided `serialize` and `deserialize` methods to ensure forward compatibility.

## Data Pipeline
DeployableConfig acts as a data marshalling component at the boundary between the network and game logic layers.

> **Inbound Flow (Receiving Data):**
> Raw TCP Stream -> Netty ByteBuf -> **DeployableConfig.validateStructure** -> **DeployableConfig.deserialize** -> Game Logic Object

> **Outbound Flow (Sending Data):**
> Game Logic Object -> `new DeployableConfig()` -> **instance.serialize(ByteBuf)** -> Netty Channel -> Raw TCP Stream

