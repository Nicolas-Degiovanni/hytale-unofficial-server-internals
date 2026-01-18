---
description: Architectural reference for AssetEditorSetGameTime
---

# AssetEditorSetGameTime

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (Packet)

## Definition
```java
// Signature
public class AssetEditorSetGameTime implements Packet {
```

## Architecture & Concepts
The AssetEditorSetGameTime class is a Data Transfer Object (DTO) that represents a single, specific command within the Hytale network protocol. It is not a service or manager, but rather a structured container for data transmitted between a client and a server.

Its primary role is to instruct the game engine to modify the state of the in-game clock. The class name strongly implies its use within a specialized development context, such as an asset editor, allowing a developer to manipulate game time for testing or content creation purposes.

This packet is designed for high performance and low overhead. It has a fixed binary size (14 bytes) and uses direct serialization to and from a Netty ByteBuf. The protocol relies on a unique Packet ID (352) to distinguish this packet type from others on the wire, enabling the network layer to route the raw byte stream to the correct deserialization logic.

## Lifecycle & Ownership
- **Creation:**
    - **Sending Peer:** An instance is created using its constructor, for example `new AssetEditorSetGameTime(new InstantData(...), true)`, typically in response to a user action within a development tool.
    - **Receiving Peer:** The object is instantiated exclusively by the network protocol layer, which calls the static factory method `AssetEditorSetGameTime.deserialize(...)` after identifying the packet ID from an incoming byte stream.

- **Scope:** Extremely short-lived and transient. An instance exists only for the duration of a single network transaction. It is created, serialized, transmitted, deserialized, processed by a handler, and then becomes eligible for garbage collection.

- **Destruction:** Managed by the Java Garbage Collector. There are no manual cleanup or `close` methods required.

## Internal State & Concurrency
- **State:** Mutable. The public fields `gameTime` and `paused` can be modified after construction. This is a deliberate design choice for DTOs, allowing them to be configured incrementally before being passed to the network layer for serialization.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and serialized within a single thread, or handled by a single network I/O thread upon receipt. Concurrent modification and serialization will lead to corrupted data on the wire. All synchronization must be handled externally.

## API Surface
The public contract is focused on serialization, deserialization, and protocol conformance.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf buf) | void | O(1) | Writes the object's state into a binary buffer for network transmission. |
| deserialize(ByteBuf buf, int offset) | static AssetEditorSetGameTime | O(1) | Constructs a new instance by reading from a binary buffer. The primary entry point for incoming data. |
| getId() | int | O(1) | Returns the constant packet identifier (352) required by the protocol. |
| computeSize() | int | O(1) | Returns the fixed size of the packet in bytes (14). |
| validateStructure(ByteBuf buffer, int offset) | static ValidationResult | O(1) | Performs a low-cost check to ensure a buffer is large enough to contain this packet. |

## Integration Patterns

### Standard Usage
The class is used as part of a client-server command flow. The sending peer constructs and sends the packet, while the receiving peer deserializes it and acts upon the data.

**Sending a command:**
```java
// Example: Pausing the game time at a specific moment
InstantData specificTime = new InstantData(/* ... */);
AssetEditorSetGameTime command = new AssetEditorSetGameTime(specificTime, true);

// The network service will internally call command.serialize(buffer)
networkChannel.sendPacket(command);
```

**Receiving and handling a command:**
```java
// This logic resides deep within the network layer's packet dispatcher
// The dispatcher has already read the packet ID (352)
AssetEditorSetGameTime packet = AssetEditorSetGameTime.deserialize(buffer, offset);

// The packet is then passed to a game logic handler
gameWorld.getTimeSystem().applyTimeUpdate(packet.gameTime, packet.paused);
```

### Anti-Patterns (Do NOT do this)
- **Object Re-use:** Do not attempt to pool or reuse packet instances. They are lightweight objects, and re-using them can lead to subtle bugs where old state is accidentally transmitted. Always create a new instance for each new command.
- **Manual Deserialization:** Do not call `new AssetEditorSetGameTime()` and manually populate its fields from a ByteBuf. The static `deserialize` method is the canonical way to construct an object from network data, as it correctly handles the internal binary layout, including the null-bit field.
- **Concurrent Access:** Do not share an instance across threads. For example, do not allow a game logic thread to modify a packet while a network thread is in the process of serializing it.

## Data Pipeline
The AssetEditorSetGameTime packet follows a clear, unidirectional data flow from a control client (the Asset Editor) to a game instance (the Server).

> Flow:
> Asset Editor UI Event -> Client Logic creates **AssetEditorSetGameTime** -> `serialize()` -> Network Transmission -> Server receives bytes -> Protocol Dispatcher -> `deserialize()` -> **AssetEditorSetGameTime** instance -> Game Time System -> World State Updated

