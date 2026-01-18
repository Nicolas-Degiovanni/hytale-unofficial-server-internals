---
description: Architectural reference for AssetEditorRebuildCaches
---

# AssetEditorRebuildCaches

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (Packet)

## Definition
```java
// Signature
public class AssetEditorRebuildCaches implements Packet {
```

## Architecture & Concepts
The AssetEditorRebuildCaches class is a network packet that functions as a command message within the Hytale engine's live asset editing framework. It is not a service or a manager; it is strictly a data container used to signal a remote client or server that specific, performance-critical asset caches must be invalidated and rebuilt from source.

Architecturally, this packet serves as the primary communication mechanism between an external asset editing tool and the running game instance. Its existence implies a sophisticated development pipeline where artists and designers can push asset changes to the game in real-time without requiring a full restart. The packet's structure, a series of boolean flags, allows for granular control over which cache types to target, preventing unnecessary and costly rebuilds of unrelated assets.

This class adheres to the Hytale network protocol's serialization contract, defined by the Packet interface. It contains the logic to convert its state to and from a Netty ByteBuf, ensuring a fixed and predictable on-the-wire data format.

## Lifecycle & Ownership
- **Creation:** An instance of AssetEditorRebuildCaches is created under two distinct circumstances:
    1.  **On the Sender:** An external tool or developer console instantiates this object, configures the boolean flags, and passes it to the network layer for serialization and transmission.
    2.  **On the Receiver:** The network protocol layer instantiates the object by invoking the static `deserialize` factory method when an incoming data frame with PACKET_ID 348 is identified.

- **Scope:** The object's lifetime is exceptionally brief and transactional. It exists only for the duration of a single network event. On the sending side, it is typically eligible for garbage collection immediately after serialization. On the receiving side, it is processed by a network handler and then discarded.

- **Destruction:** The object is managed by the Java garbage collector. There are no native resources or manual cleanup procedures required. Holding a long-term reference to a packet instance is considered a design error.

## Internal State & Concurrency
- **State:** The internal state is entirely mutable and consists of five public boolean fields. This class does not cache any data itself; it is a command that *triggers* caching operations in other, more complex systems like an AssetManager.

- **Thread Safety:** This class is **not thread-safe**. Its public, non-final fields are exposed to direct modification. In practice, this is not a concern because network packets are almost always handled by a single network thread (e.g., a Netty event loop).

    **Warning:** Sharing an instance of this packet across multiple threads without external synchronization will lead to unpredictable behavior and race conditions. It should be treated as thread-confined.

## API Surface
The public contract is focused on serialization and deserialization for network transport.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf buf) | void | O(1) | Encodes the five boolean flags into the provided buffer. Each flag consumes one byte. |
| deserialize(ByteBuf buf, int offset) | AssetEditorRebuildCaches | O(1) | Static factory method. Decodes 5 bytes from the buffer to construct a new packet instance. |
| computeSize() | int | O(1) | Returns the constant size of the packet's payload, which is always 5 bytes. |
| validateStructure(ByteBuf buffer, int offset) | ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough data for a valid read. |

## Integration Patterns

### Standard Usage
This packet is never used directly by gameplay logic. It is received by a network handler and dispatched to a dedicated asset management service.

```java
// Example of a receiver-side network handler
public void handlePacket(Packet packet) {
    if (packet.getId() == AssetEditorRebuildCaches.PACKET_ID) {
        AssetEditorRebuildCaches command = (AssetEditorRebuildCaches) packet;

        // Delegate the command to the appropriate system
        AssetService assetService = context.getService(AssetService.class);
        assetService.processCacheRebuildCommand(command);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not store an instance of this class in a field or collection. It represents a transient event, not persistent state.
- **State Modification on Receiver:** Do not modify the fields of the packet after it has been deserialized. Treat it as an immutable command to avoid side effects in the handling logic.
- **Manual Instantiation on Receiver:** Never use `new AssetEditorRebuildCaches()` on the receiving side of the network. Always use the static `deserialize` method to ensure the object is correctly populated from the network buffer.

## Data Pipeline
The primary role of this class is to carry a command through the network stack. The data itself originates from an external tool and terminates as a series of actions performed by the asset system.

> Flow:
> Asset Editor Tool -> Network Encoder -> **AssetEditorRebuildCaches (Serialized)** -> TCP/IP Stream -> Network Decoder -> **AssetEditorRebuildCaches (Deserialized)** -> AssetService -> Cache Invalidation & Filesystem I/O

