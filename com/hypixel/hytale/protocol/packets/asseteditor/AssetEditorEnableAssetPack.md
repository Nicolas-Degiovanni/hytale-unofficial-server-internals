---
description: Architectural reference for AssetEditorEnableAssetPack
---

# AssetEditorEnableAssetPack

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class AssetEditorEnableAssetPack implements Packet {
```

## Architecture & Concepts
The AssetEditorEnableAssetPack class is a Data Transfer Object (DTO) that represents a single, discrete message within the Hytale network protocol. Its sole purpose is to communicate a state change—enabling or disabling a specific asset pack—from a client to the server, typically within the context of the in-game asset editor.

As an implementation of the Packet interface, it is a fundamental building block of the network layer. It is not a service or a manager; it is inert data. The class contains the logic for its own serialization and deserialization, converting its object state to and from the Hytale wire format. This self-contained design allows the central network processor to handle any packet type polymorphically without needing to know its internal structure.

The serialization format is highly optimized for network performance, using a bitmask for nullable fields and variable-length integers (VarInt) to minimize payload size. Static constants like PACKET_ID and MAX_SIZE provide metadata to the protocol engine for dispatching, validation, and security checks.

## Lifecycle & Ownership
- **Creation:**
    - **Outbound:** Instantiated directly via its constructor, for example, `new AssetEditorEnableAssetPack("core", true)`, by a higher-level system (e.g., an Asset Editor UI controller) in response to a user action.
    - **Inbound:** Instantiated by the network protocol engine, which calls the static `deserialize` factory method after identifying the packet's ID from the incoming byte stream.
- **Scope:** Extremely short-lived. An instance exists only for the brief duration of a single network transaction. It is created, populated, serialized, and then becomes eligible for garbage collection on the sending side. On the receiving side, it is deserialized, processed by a handler, and then immediately becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or `close` methods required.

## Internal State & Concurrency
- **State:** The class holds a simple, mutable state consisting of a nullable String `id` and a boolean `enabled`. While technically mutable, instances are treated as immutable-after-creation in practice. The object's purpose is to be a snapshot of data at a specific point in time. It performs no caching.
- **Thread Safety:** This class is **not thread-safe**. Its public fields are accessed directly without any synchronization. This is intentional. The network protocol architecture guarantees that a single packet instance is handled by a single thread at any given time (e.g., a Netty I/O worker thread or a main game-loop thread). Accessing an instance from multiple threads is a severe design violation and will lead to race conditions.

## API Surface
The public contract is primarily for the network protocol engine. Application-level code typically only interacts with the constructor and public fields.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AssetEditorEnableAssetPack(id, enabled) | constructor | O(1) | Creates a new packet instance for outbound transmission. |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided Netty byte buffer. N is the length of the asset pack ID. |
| deserialize(ByteBuf, int) | static AssetEditorEnableAssetPack | O(N) | Decodes a byte buffer into a new packet instance. Throws ProtocolException on malformed data. |
| computeSize() | int | O(N) | Calculates the exact number of bytes the serialized packet will occupy. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a lightweight, read-only check of the buffer to validate lengths and prevent malformed packet attacks. |
| getId() | int | O(1) | Returns the unique, static network identifier for this packet type (318). |

## Integration Patterns

### Standard Usage
The primary pattern is to create, populate, and pass the packet to a network service for sending. On the receiving end, a handler consumes the packet's data.

```java
// SCENARIO: Sending a packet from the client
// A network service is retrieved to handle the transmission.
NetworkManager net = context.getService(NetworkManager.class);

// The packet is instantiated with the relevant data.
AssetEditorEnableAssetPack packet = new AssetEditorEnableAssetPack("com.example.mymod", true);

// The packet is sent. The network layer handles serialization.
net.sendPacket(packet);
```

```java
// SCENARIO: Receiving a packet in a handler
// This method would be invoked by the protocol dispatcher.
public void handleEnableAssetPack(AssetEditorEnableAssetPack packet) {
    // The handler acts upon the data within the packet.
    AssetService assetService = context.getService(AssetService.class);
    assetService.setAssetPackEnabled(packet.id, packet.enabled);
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not modify and re-send the same packet instance. This can cause unpredictable behavior, especially in asynchronous network environments. Always create a new instance for each message.
- **Manual Serialization/Deserialization:** Application-level code should never call `serialize` or `deserialize`. These methods are the exclusive responsibility of the core protocol engine.
- **Cross-Thread Modification:** Do not create a packet on one thread and modify it on another before sending. The state of the packet at the moment of serialization is not guaranteed.

## Data Pipeline
The class acts as a data container that is passed between stages of the network and application layers.

> **Outbound Flow:**
> User Input -> UI Controller -> `new AssetEditorEnableAssetPack()` -> NetworkManager.send() -> **AssetEditorEnableAssetPack.serialize()** -> Netty I/O Thread -> Network Socket

> **Inbound Flow:**
> Network Socket -> Netty I/O Thread -> Protocol Dispatcher -> **AssetEditorEnableAssetPack.deserialize()** -> Packet Handler -> Asset Service -> Game State Update

