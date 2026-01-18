---
description: Architectural reference for SetBlockPlacementOverride
---

# SetBlockPlacementOverride

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Transient

## Definition
```java
// Signature
public class SetBlockPlacementOverride implements Packet {
```

## Architecture & Concepts
The SetBlockPlacementOverride class is a network Packet, a specialized Data Transfer Object (DTO) used within the Hytale protocol layer. Its sole purpose is to communicate a server-authoritative state change to the client regarding block placement rules.

This packet functions as a server-to-client command. When a client receives this packet, it toggles a client-side flag that determines whether the player can place blocks in normally restricted locations. This is typically used for administrative or creative mode features, allowing privileged users to bypass standard building constraints. The client's input handling and world interaction logic must consult the state set by this packet before attempting to send a block placement request to the server.

It is a simple, fixed-size packet, reflecting its nature as a lightweight boolean toggle. It does not contain complex data structures and is designed for high-frequency, low-latency state synchronization.

## Lifecycle & Ownership
- **Creation:**
    - On the server, an instance is created via its constructor (`new SetBlockPlacementOverride(true)`) when game logic dictates a change in a player's building permissions.
    - On the client, an instance is created exclusively by the static `deserialize` factory method. This occurs deep within the Netty network pipeline when a raw byte buffer with packet ID 103 is decoded.
- **Scope:** Extremely short-lived. A packet object exists only for the duration of its processing within a single network tick or event loop cycle. It is immediately eligible for garbage collection after its data has been transferred to a more permanent state holder, such as a PlayerController or GameState service.
- **Destruction:** Managed by the Java Garbage Collector. There are no native resources or explicit cleanup methods. Holding a long-term reference to a packet object is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** The class holds a single mutable boolean field, `enabled`. While technically mutable, it should be treated as immutable after its initial construction or deserialization. Its state represents a snapshot of the server's intent at a specific moment.
- **Thread Safety:** This class is **not thread-safe**. Packet processing is designed to occur synchronously on a single network thread (e.g., the Netty event loop). Any attempt to access or modify a packet instance from multiple threads will result in race conditions and undefined behavior. All state transitions resulting from this packet must be dispatched to the main game thread if necessary.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type, 103. |
| serialize(ByteBuf) | void | O(1) | Encodes the `enabled` state into a single byte and writes it to the provided buffer. |
| deserialize(ByteBuf, int) | SetBlockPlacementOverride | O(1) | Static factory. Decodes a single byte from the buffer to construct a new packet instance. |
| computeSize() | int | O(1) | Returns the constant size of the packet's payload, which is 1 byte. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a bounds check to ensure the buffer contains at least 1 byte for decoding. |

## Integration Patterns

### Standard Usage
This packet is not intended for direct use by general game systems. It is created and consumed by the protocol layer. A server-side system sends it, and a client-side packet handler receives it.

**Server-Side (Sending Logic):**
```java
// Example: A command handler granting a player override permissions
Player targetPlayer = ...;
SetBlockPlacementOverride packet = new SetBlockPlacementOverride(true);
targetPlayer.getConnection().sendPacket(packet);
```

**Client-Side (Packet Handling Logic):**
```java
// In a client-side packet processor
public void handle(SetBlockPlacementOverride packet) {
    // Dispatch the state change to the main game thread
    gameContext.getGameThreadExecutor().execute(() -> {
        PlayerController controller = gameContext.getPlayerController();
        controller.setBlockPlacementOverride(packet.enabled);
    });
}
```

### Anti-Patterns (Do NOT do this)
- **State Storage:** Do not retain instances of this packet. Its data should be immediately copied to a persistent state manager (e.g., a Player or World object). The packet itself is ephemeral.
- **Client-Side Instantiation:** Do not create instances of this packet on the client using `new`. The client only receives and deserializes this packet; it never sends it.
- **Cross-Thread Access:** Do not access a deserialized packet instance from any thread other than the one that decoded it (typically a Netty I/O thread) without proper synchronization or dispatching.

## Data Pipeline
The data flow for this packet is unidirectional from the server to the client.

> Flow:
> Server Game Logic -> **new SetBlockPlacementOverride()** -> Network Channel `serialize()` -> TCP/IP Stack -> Client Network Channel `deserialize()` -> Client Packet Processor -> Game State Update (e.g., PlayerController)

