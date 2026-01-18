---
description: Architectural reference for PlayAnimation
---

# PlayAnimation

**Package:** com.hypixel.hytale.protocol.packets.entities
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class PlayAnimation implements Packet {
```

## Architecture & Concepts
The PlayAnimation packet is a network message DTO responsible for instructing a client to render an animation on a specific game entity. It serves as a fundamental data structure for synchronizing visual state between the server and client. This class is not a service or manager; it is a pure data container with specialized serialization logic tailored for the Hytale network protocol.

Its design prioritizes network efficiency through a custom binary format. The packet structure is split into two main sections:
1.  **Fixed-Size Block:** Contains predictable, constant-size fields like entityId and the animation slot. This allows for rapid, direct-memory access during deserialization.
2.  **Variable-Size Block:** Contains data of unpredictable length, such as the string-based animation identifiers.

A key architectural pattern employed is the use of a **nullable bit field**. The first byte of the packet is a bitmask where each bit corresponds to a nullable field (itemAnimationsId, animationId). If a bit is set, the corresponding field is present in the payload. This avoids transmitting empty strings or sentinel values, minimizing bandwidth usage.

Furthermore, the fixed-size block does not contain the variable data directly. Instead, it contains integer offsets that point to the location of the variable data within the packet's payload. This allows the deserializer to quickly parse the fixed block and then jump to the correct memory locations to read the variable-length strings.

## Lifecycle & Ownership
-   **Creation:**
    -   **Outbound (Sending):** Instantiated by server-side game logic (e.g., an AnimationSystem) when an entity's state changes in a way that requires a new animation. The object is populated with data and passed to the network layer.
    -   **Inbound (Receiving):** Instantiated by the client-side network protocol handler. A central packet factory reads the packet ID from the incoming byte stream and invokes the static `deserialize` method to construct a PlayAnimation object from the raw buffer.

-   **Scope:** Short-lived and stateless. An instance of PlayAnimation exists only for the brief period it is being created, serialized, or processed by a network event handler. It is a message, not a persistent entity.

-   **Destruction:** The object is eligible for garbage collection as soon as the network layer has finished serializing it (outbound) or the event handler has finished processing it (inbound). There is no manual resource management.

## Internal State & Concurrency
-   **State:** The class is a mutable data container. Its fields are public and can be modified after construction. It holds no caches, external resources, or connections. Its state is entirely defined by its member variables.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and processed within a single thread, such as a Netty I/O worker or the main game logic thread. Concurrent modification of its fields from multiple threads will result in unpredictable behavior. All serialization and deserialization operations on a shared Netty ByteBuf must be strictly synchronized externally.

## API Surface
The primary contract is defined by its static utility methods for interacting with the network protocol.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static PlayAnimation | O(N) | Constructs a PlayAnimation object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. N is the total length of the string fields. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the given ByteBuf according to the Hytale binary protocol. N is the total length of the string fields. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a read-only check of the buffer to ensure it contains a structurally valid PlayAnimation packet. Crucial for security and stability. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will occupy on the wire when serialized. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N) | Calculates the total size of a serialized PlayAnimation packet directly from a buffer without full deserialization. |
| getId() | int | O(1) | Returns the static network identifier (162) for this packet type. |

## Integration Patterns

### Standard Usage
The class is used either to prepare an outbound message or to interpret an inbound one.

**Sending a Packet (Server-Side Logic)**
```java
// Game logic determines an animation needs to play
PlayAnimation packet = new PlayAnimation(
    targetEntityId,
    "hytale:weapon_anims",
    "sword_swing_1",
    AnimationSlot.Action
);

// The network layer (e.g., a Netty Channel) serializes and sends it
playerConnection.sendPacket(packet);
```

**Receiving a Packet (Client-Side Handler)**
```java
// In a network event handler after the packet has been deserialized
public void handlePlayAnimation(PlayAnimation packet) {
    Entity entity = world.getEntityById(packet.entityId);
    if (entity != null) {
        AnimationController controller = entity.getAnimationController();
        controller.play(packet.animationId, packet.slot);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Object Re-use:** Do not hold onto and modify a single PlayAnimation instance to send multiple, different animation events. This is error-prone. Always create a new, clean instance for each distinct network message.
-   **Bypassing Validation:** On a publicly accessible server, failing to call `validateStructure` on an incoming buffer before attempting to `deserialize` it can create security vulnerabilities, such as denial-of-service via maliciously crafted packets that declare impossibly large string lengths.
-   **Manual Buffer Manipulation:** Do not call `serialize` and then attempt to manually alter the resulting ByteBuf. The internal offsets for variable-length fields are carefully calculated and will become invalid, leading to protocol corruption.

## Data Pipeline
The PlayAnimation class acts as a bridge between high-level game logic and the low-level byte stream.

**Outbound Flow (e.g., Server to Client)**
> Game Event -> `new PlayAnimation(...)` -> Network Channel -> **PlayAnimation.serialize()** -> Netty ByteBuf -> TCP/IP Stack

**Inbound Flow (e.g., Client from Server)**
> TCP/IP Stack -> Netty ByteBuf -> Protocol Dispatcher (reads Packet ID 162) -> **PlayAnimation.deserialize()** -> PlayAnimation Instance -> Client-Side Animation System -> Entity Visual Update

