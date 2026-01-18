---
description: Architectural reference for AssetEditorUpdateSecondsPerGameDay
---

# AssetEditorUpdateSecondsPerGameDay

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class AssetEditorUpdateSecondsPerGameDay implements Packet {
```

## Architecture & Concepts
AssetEditorUpdateSecondsPerGameDay is a network packet definition within the Hytale protocol layer. It serves as a simple, fixed-size Data Transfer Object (DTO) for synchronizing the duration of the in-game day/night cycle. Its primary role is to convey configuration changes made within the Asset Editor to a running game instance, allowing for real-time updates to world parameters.

As an implementation of the Packet interface, this class adheres to a strict contract for network communication. It contains metadata, such as PACKET_ID, which is used by the protocol dispatcher to route incoming byte streams to the correct deserialization logic. The class structure is optimized for high-performance serialization and deserialization, using a predefined, fixed-block layout on the wire.

This packet is a leaf node in the protocol system; it carries data but contains no business logic. Its design prioritizes simplicity and performance over flexibility, as evidenced by its fixed size and direct field access.

### Lifecycle & Ownership
- **Creation:** An instance is created under two distinct circumstances:
    1.  **Inbound:** The network protocol layer instantiates the object via the static `deserialize` factory method when a raw network buffer with a matching packet ID (353) is received.
    2.  **Outbound:** The application logic, typically within the Asset Editor service, creates a new instance using its constructor to prepare a message for transmission.
- **Scope:** The object's lifetime is extremely brief and transactional. It exists only long enough to be serialized into a network buffer for sending, or to be read by a handler after being deserialized. It is not intended to be stored or referenced long-term.
- **Destruction:** Ownership is never transferred. The object is eligible for garbage collection immediately after the network operation (send or receive) is complete. There are no manual resource management or cleanup procedures.

## Internal State & Concurrency
- **State:** The internal state is fully **mutable** and consists of two public integer fields: daytimeDurationSeconds and nighttimeDurationSeconds. The class acts as a simple data container with no encapsulation, caching, or derived state.
- **Thread Safety:** This class is **not thread-safe**. It provides no internal synchronization mechanisms. It is designed to be created, populated, and processed within a single thread, such as a Netty I/O worker or the main game loop thread.

**WARNING:** Sharing an instance of this packet across multiple threads without explicit, external locking will lead to race conditions and unpredictable behavior.

## API Surface
The public API is minimal, focusing exclusively on the requirements of the network protocol framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the unique, static identifier (353) for this packet type. |
| serialize(ByteBuf) | void | O(1) | Encodes the object's state into the provided Netty ByteBuf using little-endian byte order. |
| computeSize() | int | O(1) | Returns the constant size (8 bytes) of the packet on the wire. |
| deserialize(ByteBuf, int) | AssetEditorUpdateSecondsPerGameDay | O(1) | A static factory method that decodes a new instance from a network buffer at a given offset. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | A static utility to verify if a buffer has sufficient readable bytes to contain this packet. |

## Integration Patterns

### Standard Usage
This packet is exclusively managed by the network protocol and application-level handlers. Developers should not interact with its serialization methods directly.

**Outbound (Sending from Asset Editor):**
```java
// Application logic creates and populates the packet
AssetEditorUpdateSecondsPerGameDay packet = new AssetEditorUpdateSecondsPerGameDay();
packet.daytimeDurationSeconds = 600; // 10 minutes
packet.nighttimeDurationSeconds = 480; // 8 minutes

// The network layer handles serialization and transmission
gameConnection.sendPacket(packet);
```

**Inbound (Receiving by Game Client/Server):**
```java
// A handler method is invoked by the protocol dispatcher
public void handlePacket(AssetEditorUpdateSecondsPerGameDay packet) {
    // Read the values and apply them to the game world state
    WorldTimeManager timeManager = context.getService(WorldTimeManager.class);
    timeManager.setDayDuration(packet.daytimeDurationSeconds);
    timeManager.setNightDuration(packet.nighttimeDurationSeconds);
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify a packet instance after it has been sent or received. For subsequent messages, create a new instance to ensure message integrity and avoid side effects.
- **Cross-Thread Modification:** Do not create a packet on one thread and pass it to a network thread for sending without a proper memory barrier or synchronization. The Java Memory Model does not guarantee visibility of the fields.
- **Manual Serialization:** Never call `serialize` directly. This is the responsibility of the Netty pipeline, which manages buffer allocation and lifecycle.

## Data Pipeline
The flow of data for this packet is linear and unidirectional, from the editor to the game.

**Outbound Flow:**
> Asset Editor UI (User changes value) -> Event Handler -> **AssetEditorUpdateSecondsPerGameDay (Instance Created)** -> Network Channel Write -> Protocol Encoder (calls `serialize`) -> TCP Socket

**Inbound Flow:**
> TCP Socket -> Netty ByteBuf -> Protocol Decoder (calls `deserialize`) -> **AssetEditorUpdateSecondsPerGameDay (Instance Created)** -> Packet Dispatcher -> Game Logic Handler -> WorldTimeManager Update

