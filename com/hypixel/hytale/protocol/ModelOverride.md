---
description: Architectural reference for ModelOverride
---

# ModelOverride

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ModelOverride {
```

## Architecture & Concepts
The ModelOverride class is a fundamental Data Transfer Object (DTO) within the Hytale network protocol layer. It is not a service or a manager, but rather a passive data structure that represents a message for dynamically altering an entity's visual appearance.

Its primary architectural role is to serve as a standardized, serializable container for visual properties like 3D models, textures, and animation sets. This allows the server to instruct the client to render an entity differently from its default configuration without requiring new client-side assets or code changes.

The serialization format is custom and highly optimized for network efficiency. It employs a fixed-size header block (13 bytes) that contains a bitmask for nullable fields and integer offsets pointing to the location of variable-length data. This design allows for:
1.  **Efficient Null-Field Handling:** Optional fields consume no space in the variable data block.
2.  **Random Access:** A parser can read the header and immediately jump to a specific field's data without parsing preceding fields.
3.  **Pre-computation:** The size of the entire structure within a buffer can be calculated without a full deserialization pass via the computeBytesConsumed method.

## Lifecycle & Ownership
-   **Creation:** An instance is created under two primary circumstances:
    1.  **On the sending side (e.g., Server):** Instantiated via its constructor (`new ModelOverride(...)`) by game logic that needs to dispatch a visual change.
    2.  **On the receiving side (e.g., Client):** Instantiated by the network protocol decoder, which invokes the static `deserialize` factory method on an incoming Netty ByteBuf.
-   **Scope:** The object's lifetime is intentionally brief. It is scoped to the transaction of a single network packet or a single game-tick update. It is designed to be processed immediately and then discarded.
-   **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as no more references point to it, which is typically right after the relevant game system (e.g., an EntityVisualsSystem) has consumed its data.

## Internal State & Concurrency
-   **State:** The class is **mutable**. Its public fields can be directly modified after instantiation. It acts as a simple data container and holds no internal caches, derived state, or hidden logic. Its state is a direct 1-to-1 mapping of the data it was deserialized from or the data it will be serialized into.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and read within a single-threaded context, such as a Netty event loop or the main game thread.
    -   **WARNING:** Sharing a ModelOverride instance across threads without explicit external synchronization will result in memory consistency errors and unpredictable behavior. The intended pattern is to process it entirely on the thread that created it.

## API Surface
The public API is dominated by static methods for serialization, reflecting its role as a protocol-level data structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ModelOverride | O(N) | Constructs a ModelOverride instance by reading from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the custom protocol format. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total byte size of a serialized ModelOverride in a buffer without full deserialization. |
| computeSize() | int | O(N) | Calculates the required byte size to serialize the current instance. Useful for pre-allocating buffers. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a security and integrity check on a buffer to ensure it contains a valid structure before attempting deserialization. |

## Integration Patterns

### Standard Usage
The canonical use case involves a network handler deserializing the object from a packet and passing it to a consumer system for immediate application.

```java
// On the client's network thread or subsequent game tick
ByteBuf packetData = ...;
ModelOverride override = ModelOverride.deserialize(packetData, 0);

// Pass the data to the entity system to apply the visual update
Entity targetEntity = world.getEntity(entityId);
entityVisualsSystem.applyOverride(targetEntity, override);
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage:** Do not retain instances of ModelOverride in components or systems as part of their long-term state. They are transient messages. If the data must be saved, copy the primitive values (model string, texture string) into a dedicated state component.
-   **Cross-Thread Sharing:** Never deserialize an object on a network thread and then pass the mutable instance to the main game thread for processing. This is a classic race condition. The correct pattern is to pass an immutable copy or use a thread-safe queue to transfer the raw data for processing on the main thread.
-   **Ignoring Validation:** In a server context, failing to call `validateStructure` on data received from a client before calling `deserialize` can expose the server to denial-of-service attacks via maliciously crafted packets designed to cause exceptions or excessive memory allocation.

## Data Pipeline
ModelOverride acts as a data payload that flows from a logical instruction on one machine to a rendering update on another.

> Flow:
> Server Game Logic -> **ModelOverride (Instance)** -> `serialize()` -> Outbound ByteBuf -> Network -> Inbound ByteBuf -> `deserialize()` -> **ModelOverride (Instance)** -> Client Entity System -> Render Command Queue

