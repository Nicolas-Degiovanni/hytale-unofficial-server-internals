---
description: Architectural reference for BuilderToolOptionArg
---

# BuilderToolOptionArg

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class BuilderToolOptionArg {
```

## Architecture & Concepts
The BuilderToolOptionArg class is a serializable data structure, not a complete network packet. It functions as a Data Transfer Object (DTO) designed to be embedded within larger packets related to the in-game building and creation toolset. Its primary purpose is to define a configurable property that has a default value and a list of possible string-based options, such as a material selector in a user interface.

The architectural significance of this class lies in its highly optimized, custom binary serialization format. It eschews standard Java serialization in favor of a low-level, byte-perfect layout managed directly on a Netty ByteBuf. This design is critical for minimizing network bandwidth and reducing CPU overhead for encoding and decoding, which are paramount in a high-performance game client-server architecture.

The binary format consists of three main parts:
1.  **Null Bit Field:** A single byte at the start indicates which of the nullable fields (defaultValue, options) are present in the data stream.
2.  **Fixed-Size Block:** A block containing integer offsets that point to the location of variable-sized data.
3.  **Variable-Size Block:** A contiguous data area where the actual string and array data are written.

This structure allows for efficient random-access parsing and validation without needing to read the entire data stream sequentially.

## Lifecycle & Ownership
-   **Creation:** An instance is created under two distinct circumstances:
    1.  **Deserialization:** The static factory method `deserialize` is invoked by the network protocol layer when decoding an incoming packet that contains this structure. This is the most common creation path on the receiving end.
    2.  **Direct Instantiation:** Game logic, typically on the server, creates an instance using `new BuilderToolOptionArg(...)` to define the properties of a builder tool before serializing it into a packet for transmission to a client.

-   **Scope:** This object is **transient** and short-lived. Its lifetime is strictly bound to the lifecycle of its containing network packet or the immediate scope of the game logic that created it. It is not intended to be cached or persisted.

-   **Destruction:** The object is managed by the Java Garbage Collector. It is eligible for collection as soon as all references to it (and its parent packet) are released. No manual memory management is required.

## Internal State & Concurrency
-   **State:** The internal state is composed of the `defaultValue` string and the `options` string array. This state is **fully mutable** via direct field access.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without explicit external synchronization. It is designed for use within a single, well-defined thread context, such as a Netty event loop thread or the main game thread. Modifying its fields while another thread is serializing or reading from it will result in data corruption and undefined behavior.

## API Surface
The public API is centered around the serialization and deserialization contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static BuilderToolOptionArg | O(N) | Constructs an object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Encodes the object's state into the provided ByteBuf. Throws ProtocolException if data constraints are violated. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will occupy when serialized. N is the total length of all strings. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a safe, read-only check of the binary structure in a buffer to ensure validity before attempting deserialization. |

## Integration Patterns

### Standard Usage
This class is almost never used in isolation. It is constructed, populated, and then passed to a parent packet for serialization, or it is received as the result of a parent packet's deserialization process. The typical flow involves a full serialization-deserialization cycle.

```java
// Example: Server-side creation and serialization
BuilderToolOptionArg toolArgs = new BuilderToolOptionArg(
    "Oak Planks",
    new String[]{"Oak Planks", "Spruce Planks", "Birch Planks"}
);

// Assume this is part of a larger packet's serialization logic
ByteBuf buffer = Unpooled.buffer();
toolArgs.serialize(buffer);

// --- Network Transmission ---

// Example: Client-side validation and deserialization
ValidationResult result = BuilderToolOptionArg.validateStructure(buffer, 0);
if (!result.isOk()) {
    throw new IllegalStateException("Received malformed packet data: " + result.getReason());
}

BuilderToolOptionArg receivedArgs = BuilderToolOptionArg.deserialize(buffer, 0);
// Now use receivedArgs to configure the client-side UI
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Do not modify the public fields of this object from one thread while another thread is calling `serialize`. The serialization process reads these fields directly and is not atomic.
-   **Skipping Validation:** On a server processing untrusted client input, failing to call `validateStructure` before `deserialize` is a security risk. A malicious client could send a malformed packet with invalid offsets or lengths, potentially causing server exceptions or, in worse cases, denial-of-service.
-   **Long-Term Storage:** Do not hold references to BuilderToolOptionArg instances in long-lived caches or data stores. They are DTOs meant for immediate processing after creation or deserialization.

## Data Pipeline
The BuilderToolOptionArg class is a critical link in the network data pipeline, acting as a codec for a specific data shape.

**Outbound (Serialization):**
> Game Logic (Server) -> **new BuilderToolOptionArg()** -> Parent Packet Construction -> **serialize()** -> Netty ByteBuf -> Network Socket

**Inbound (Deserialization):**
> Network Socket -> Netty ByteBuf -> Parent Packet Deserializer -> **BuilderToolOptionArg.deserialize()** -> Game Logic (Client)

