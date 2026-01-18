---
description: Architectural reference for BuilderToolsSetSoundSet
---

# BuilderToolsSetSoundSet

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class BuilderToolsSetSoundSet implements Packet {
```

## Architecture & Concepts
The BuilderToolsSetSoundSet class is a network packet definition within the Hytale protocol layer. It serves as a simple, fixed-size Data Transfer Object (DTO) designed for high-performance network communication. Its sole purpose is to transmit a command from a game client to the server, indicating that the player has selected a new sound set within the in-game builder tools.

This class is not a service or manager; it is pure data. The static fields such as PACKET_ID, FIXED_BLOCK_SIZE, and IS_COMPRESSED provide critical metadata to the core network protocol engine. This metadata allows the engine's packet dispatcher to identify the packet type from an incoming byte stream, validate its structure, and route it to the correct deserialization logic without needing to inspect the payload itself.

This packet represents a client-initiated command, forming part of a request-response or fire-and-forget interaction pattern common in client-server game architectures.

### Lifecycle & Ownership
- **Creation:** An instance is created in one of two scenarios:
    1.  On the client, by game logic responding to a user interface event (e.g., a player clicking a new sound in the builder tools menu).
    2.  On the server, by the static `deserialize` factory method when the network layer receives a byte stream with the corresponding PACKET_ID (418).
- **Scope:** Extremely short-lived and transient. An instance exists only for the brief moment it takes to be serialized into a network buffer on the sending side, or from the moment of deserialization to the completion of its processing on the receiving side.
- **Destruction:** Instances are eligible for garbage collection immediately after use. There is no persistent ownership of these objects; they are ephemeral messages.

## Internal State & Concurrency
- **State:** Mutable. The class contains a single public integer field, `soundSetIndex`, which is directly accessible. The object is intended to be created and immediately populated before being passed to the network layer. Once serialized or after deserialization, it should be treated as an immutable value object.
- **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization primitives. It is designed to be created, serialized, and deserialized within a single thread, typically a Netty I/O thread or the main game logic thread. Sharing instances across threads without external synchronization will result in race conditions and undefined behavior.

## API Surface
The public API is primarily for use by the protocol engine itself. Direct interaction by game logic developers is limited to construction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static packet identifier (418). |
| serialize(ByteBuf) | void | O(1) | Writes the `soundSetIndex` into the provided Netty buffer. |
| computeSize() | int | O(1) | Returns the fixed size of the packet in bytes (4). |
| deserialize(ByteBuf, int) | BuilderToolsSetSoundSet | O(1) | **Static Factory.** Reads from a buffer to construct a new packet instance. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static Validator.** Checks if the buffer has enough readable bytes for this packet. |

## Integration Patterns

### Standard Usage
The typical use case involves creating an instance, setting its data, and handing it off to the network system for delivery. The receiving end gets a fully-formed object to act upon.

```java
// Client-side: A UI handler sends the packet
int selectedSoundIndex = 5; // The index chosen by the player
BuilderToolsSetSoundSet packet = new BuilderToolsSetSoundSet(selectedSoundIndex);
networkClient.sendPacket(packet);

// Server-side: A packet handler processes the received data
public void handleSetSoundSet(Player player, BuilderToolsSetSoundSet packet) {
    player.getBuilderState().setActiveSoundSet(packet.soundSetIndex);
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not modify and re-send an existing packet instance. This is not safe, especially in an asynchronous network environment. Always create a new instance for each distinct message.
- **Manual Serialization:** Avoid calling `serialize` directly. The network engine is responsible for managing the serialization pipeline. Passing the packet object to a `sendPacket` method is the correct abstraction.
- **Cross-Thread Sharing:** Do not create a packet on one thread and pass its reference to another for modification or sending. This violates the threading model and can corrupt the network buffer.

## Data Pipeline
The flow for this packet is unidirectional from the client to the server.

> Flow:
> Client UI Event -> Game Logic creates `BuilderToolsSetSoundSet` -> Network Engine calls `serialize` -> TCP/UDP Stream -> Server Network Engine receives bytes -> Packet Dispatcher uses PACKET_ID -> **`deserialize` creates BuilderToolsSetSoundSet** -> Server Packet Handler -> Player State is updated

