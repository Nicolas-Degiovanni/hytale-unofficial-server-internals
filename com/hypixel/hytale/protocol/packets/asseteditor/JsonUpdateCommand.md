---
description: Architectural reference for JsonUpdateCommand
---

# JsonUpdateCommand

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class JsonUpdateCommand {
```

## Architecture & Concepts

The JsonUpdateCommand class is a specialized Data Transfer Object (DTO) designed for high-performance network communication between the Hytale Asset Editor and the game client or server. It encapsulates a single, atomic modification to a JSON-based asset, such as setting a property, inserting an element, or removing a node.

Its primary architectural role is to serve as the payload for a network packet. Instead of transmitting verbose JSON text, which is inefficient, this class implements a custom, compact binary serialization format. This design prioritizes network bandwidth and deserialization speed, which are critical for a real-time editing experience.

The binary format is notable for its hybrid fixed-variable layout:
1.  **Nullable Bit Field:** A single byte at the start of the payload acts as a bitmask, indicating which of the nullable fields (like path, value, etc.) are present in the data stream. This avoids wasting space for optional data.
2.  **Fixed-Size Block:** A header of a constant size (23 bytes) follows the bitmask. It contains the command type and a series of integer offsets.
3.  **Variable-Size Block:** All variable-length data (strings and arrays) are packed into a single block at the end of the payload. The offsets in the fixed-size block point to the start of each corresponding data element within this variable block.

This structure allows for extremely fast, non-reflective deserialization, as the parser can directly calculate data locations without scanning the payload.

## Lifecycle & Ownership

-   **Creation:** An instance of JsonUpdateCommand is created in two primary scenarios:
    1.  **On the Sender:** The Asset Editor instantiates a new command object in response to a user action (e.g., changing a value in a property grid). The object is populated with the relevant data (path, new value) and then passed to the network layer for serialization.
    2.  **On the Receiver:** The network protocol layer invokes the static `deserialize` method upon receiving a corresponding packet. This creates a new JsonUpdateCommand instance by reading directly from the network `ByteBuf`.

-   **Scope:** The object's lifetime is exceptionally short and tied to a single network operation. It is a transient object, designed to be created, processed, and then immediately discarded.

-   **Destruction:** The object is managed by the Java Garbage Collector. Once the network handler or game system has processed the command, all references to the instance are dropped, making it eligible for garbage collection. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency

-   **State:** The class is **mutable**. Its fields are public and intended for direct access after creation or deserialization. It acts as a simple data container with no internal logic for state management.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context, such as a Netty I/O thread or a main game loop tick.
    -   **WARNING:** Do not share an instance of JsonUpdateCommand across threads without explicit, external synchronization. Concurrent reads and writes will lead to race conditions and unpredictable behavior. The intended pattern is to process the object fully on the thread that deserialized it.

## API Surface

The public contract is centered on serialization, deserialization, and validation for network transport.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static JsonUpdateCommand | O(N) | Constructs a new object by parsing the binary format from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into a ByteBuf using the custom binary format. Throws ProtocolException if data exceeds size limits. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a safe, read-only check of the binary data in a buffer to ensure structural integrity before attempting a full deserialization. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the current object state. Useful for buffer pre-allocation. |

## Integration Patterns

### Standard Usage

The class is almost exclusively used by the networking layer. A network handler receives a `ByteBuf`, validates it, and deserializes the command for further processing by a dedicated system.

```java
// Executed within a Netty channel handler or similar context
ByteBuf packetPayload = ...;

// 1. Validate the packet structure before processing to prevent errors
ValidationResult result = JsonUpdateCommand.validateStructure(packetPayload, 0);
if (!result.isOk()) {
    throw new ProtocolException("Invalid JsonUpdateCommand: " + result.getErrorMessage());
}

// 2. Deserialize into a usable object
JsonUpdateCommand command = JsonUpdateCommand.deserialize(packetPayload, 0);

// 3. Dispatch the command to the appropriate system
assetMutationService.apply(command);
```

### Anti-Patterns (Do NOT do this)

-   **State Reuse:** Do not modify and re-serialize the same JsonUpdateCommand instance. This can lead to subtle bugs if not all fields are correctly reset. Always create a new instance for each distinct command.
-   **Cross-Thread Sharing:** Do not deserialize a command on a network thread and pass the reference to a worker thread for processing without creating a defensive copy. The underlying `ByteBuf` may be recycled by Netty, and the mutable state is not safe for concurrent access.
-   **Ignoring Validation:** Do not call `deserialize` on untrusted data without first calling `validateStructure`. A malformed packet could otherwise throw exceptions that crash the network loop or, in a worst-case scenario, exploit buffer handling bugs.

## Data Pipeline

JsonUpdateCommand is a critical link in the real-time asset editing data flow, translating user interface actions into network-efficient binary commands.

> Flow:
> Asset Editor UI Event -> **JsonUpdateCommand (Instantiation & Population)** -> `serialize()` -> Network Channel -> **JsonUpdateCommand (Deserialization)** -> Asset Management System -> Game State Update

