---
description: Architectural reference for UpdateFeatures
---

# UpdateFeatures

**Package:** com.hypixel.hytale.protocol.packets.setup
**Type:** Transient

## Definition
```java
// Signature
public class UpdateFeatures implements Packet {
```

## Architecture & Concepts
The UpdateFeatures class is a network packet definition within the Hytale protocol layer. It serves as a specialized Data Transfer Object (DTO) used exclusively during the client-server connection setup sequence. Its primary function is to communicate and negotiate the enabled or disabled state of various client-side features, ensuring both endpoints are synchronized.

This class embodies the principle of separating data from logic. It contains no business logic itself; its sole responsibilities are:
1.  To provide a type-safe representation of the feature set data.
2.  To implement the precise serialization and deserialization logic for its binary wire format.

The implementation is tightly coupled with the Netty framework, operating directly on Netty ByteBuf objects. The serialization format is highly optimized for network efficiency, employing a custom binary layout that uses a null-bit field to indicate the presence of the data payload and variable-length integers (VarInt) to encode collection sizes, minimizing bandwidth consumption.

## Lifecycle & Ownership
-   **Creation:**
    -   **Sending Peer:** An instance is created via its constructor, typically by a connection management service that populates the features map before sending it to the network pipeline. Example: `new UpdateFeatures(serverSupportedFeatures)`.
    -   **Receiving Peer:** An instance is never created with `new`. It is materialized by the protocol decoding layer, which invokes the static `deserialize` factory method upon identifying a packet with ID 31 in the inbound network stream.

-   **Scope:** Extremely short-lived and stateless. An instance of UpdateFeatures exists only for the brief moment it is being serialized for transmission or after being deserialized for processing. It is a message, not a persistent entity.

-   **Destruction:** The object is immediately eligible for garbage collection after its contents have been read and applied by a packet handler. Holding long-term references to packet objects is a memory leak anti-pattern.

## Internal State & Concurrency
-   **State:** The class holds a single mutable field, `features`, which is a nullable Map. The state of the object can be changed after its creation, though this is not a recommended practice after it has been handed to the network layer.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, serialized, or deserialized and processed within a single thread, typically a Netty I/O worker. Any concurrent access or modification of the internal `features` map without external synchronization will result in undefined behavior, race conditions, and potential data corruption during serialization.

    **WARNING:** Confine all interactions with an UpdateFeatures instance to the network event loop thread. If data must be passed to other threads, extract the feature map into a thread-safe collection first.

## API Surface
The public API is dominated by static methods for serialization and validation, reinforcing its role as a DTO with associated protocol codecs.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static UpdateFeatures | O(N) | Constructs an UpdateFeatures object by reading from a binary buffer. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided binary buffer according to the defined wire format. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the object. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a read-only check of the buffer to verify if a structurally valid packet exists at the offset. Does not deserialize. |
| getId() | int | O(1) | Returns the static packet identifier, 31. |

*N represents the number of entries in the features map.*

## Integration Patterns

### Standard Usage
The class is used as part of a network protocol handler pipeline.

**Sending a Packet:**
```java
// A server-side connection handler prepares the packet
Map<ClientFeature, Boolean> featuresToSend = new HashMap<>();
featuresToSend.put(ClientFeature.SOME_FEATURE, true);

UpdateFeatures packet = new UpdateFeatures(featuresToSend);

// The packet is passed to the network channel to be encoded and sent
channel.writeAndFlush(packet);
```

**Receiving a Packet:**
The `deserialize` method is invoked by a central packet decoder, not directly by application code. The resulting object is then passed to a registered handler.

```java
// Inside a packet handler method
public void handleUpdateFeatures(UpdateFeatures packet) {
    if (packet.features != null) {
        // Apply the feature flags to the current session
        this.session.getFeatureManager().applyFeatures(packet.features);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not retain instances of UpdateFeatures after processing. They represent a point-in-time message and should be discarded. Caching or reusing packet objects is unsafe.
-   **Direct Deserialization:** Avoid calling `deserialize` directly. This task belongs to a dedicated, pipeline-aware protocol decoder that manages buffer offsets and packet boundaries.
-   **Modifying After Sending:** Do not modify the `features` map after the packet has been passed to the network layer for sending. The serialization may occur on a different thread, leading to a race condition.

## Data Pipeline
The UpdateFeatures object is a transient container for data as it moves through the network stack.

**Outbound Flow (e.g., Server to Client):**
> Session Feature State -> `new UpdateFeatures()` -> **UpdateFeatures Instance** -> `serialize()` -> Netty Channel Pipeline -> Encoded ByteBuf -> Network Socket

**Inbound Flow (e.g., Client receiving from Server):**
> Network Socket -> Raw ByteBuf -> Netty Channel Pipeline -> Packet Decoder identifies ID 31 -> `UpdateFeatures.deserialize()` -> **UpdateFeatures Instance** -> Packet Handler -> Session Feature State Updated

