---
description: Architectural reference for PlaceBlockInteraction
---

# PlaceBlockInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class PlaceBlockInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The PlaceBlockInteraction class is a Data Transfer Object (DTO) that represents a specific, serializable network message. It encapsulates all the parameters required for a player to perform a "place block" action within the game world. This class is a fundamental component of the client-server communication protocol.

Architecturally, it is designed for high-performance network I/O. Its serialization format is custom-tailored to minimize payload size and processing overhead. The binary layout is split into two distinct sections:

1.  **Fixed-Size Block:** A 45-byte header containing primitive types and offsets. This allows for extremely fast, direct memory access to core fields without parsing the entire message.
2.  **Variable-Size Block:** A subsequent data region that stores complex, nullable objects like InteractionEffects, settings maps, and tag arrays. The fixed-size block contains integer offsets pointing to the start of each of these objects within the variable block.

A key feature is the `nullBits` byte field, which acts as a bitmask. Each bit corresponds to a nullable, variable-sized field, indicating its presence or absence in the payload. This avoids the need for null terminators or other markers, further optimizing for size and deserialization speed.

This class is not a service; it is pure data. It should never contain game logic. Its sole responsibility is to define the structure of the `PlaceBlockInteraction` message and provide the mechanisms for converting it to and from a byte stream.

### Lifecycle & Ownership
- **Creation:** An instance is created under two circumstances:
    1.  **Inbound:** The network protocol layer instantiates it by calling the static `deserialize` method when a corresponding packet is received from a Netty ByteBuf.
    2.  **Outbound:** Game logic instantiates it directly via its constructor (`new PlaceBlockInteraction(...)`) to prepare a message for sending.
- **Scope:** The object's lifetime is extremely short and transactional. It exists only for the duration of a single network event processing cycle. Once serialized for sending or after its data has been consumed by a handler, it is immediately eligible for garbage collection.
- **Destruction:** The object is managed by the Java garbage collector. There are no manual cleanup or `close` methods. Holding long-term references to PlaceBlockInteraction instances is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** The state is entirely mutable. All fields are public and intended for direct access after construction or deserialization. The class is a simple container for data and performs no internal caching.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and read within a single thread, typically a Netty I/O thread or the main game logic thread. Concurrent modification from multiple threads will lead to data corruption and unpredictable behavior during serialization. All synchronization must be handled externally by the calling systems.

## API Surface
The public contract is dominated by static utility methods for protocol handling and the instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | PlaceBlockInteraction | O(N) | **[Static]** Constructs an object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state into the provided ByteBuf. Returns the number of bytes written. |
| computeBytesConsumed(buf, offset) | int | O(N) | **[Static]** Calculates the total size of a serialized object in a buffer without full deserialization. Critical for advancing buffer read pointers. |
| validateStructure(buf, offset) | ValidationResult | O(N) | **[Static]** Performs a structural integrity check on the binary data in a buffer. Verifies offsets and lengths before attempting a full deserialization. |
| clone() | PlaceBlockInteraction | O(N) | Creates a deep copy of the interaction object. |
| computeSize() | int | O(N) | Calculates the expected byte size of the object if it were to be serialized. |

## Integration Patterns

### Standard Usage
The class is used by network handlers to decode incoming data or by game systems to encode outgoing actions. It should never be passed across broad system boundaries as a long-lived state object.

**Deserializing an Inbound Message:**
```java
// In a network handler receiving a ByteBuf
ValidationResult result = PlaceBlockInteraction.validateStructure(buffer, offset);
if (!result.isValid()) {
    throw new ProtocolException("Invalid PlaceBlockInteraction: " + result.error());
}
PlaceBlockInteraction interaction = PlaceBlockInteraction.deserialize(buffer, offset);
// ... process the interaction data
```

**Serializing an Outbound Message:**
```java
// In game logic preparing to send an action
PlaceBlockInteraction interaction = new PlaceBlockInteraction();
interaction.blockId = 1; // Set required fields
interaction.allowDragPlacement = true;
// ...

// Pass to a network service that writes it to the buffer
networkService.send(interaction); // Internally calls interaction.serialize(buffer)
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and resend the same PlaceBlockInteraction instance. It is cheap to create and should be treated as immutable after its initial population.
- **Cross-Thread Access:** Do not read fields from one thread while another thread is calling `serialize`. This will cause race conditions where partially updated data is written to the network stream.
- **Long-Term Storage:** Do not store instances of this class in caches or as member variables of long-lived services. They are messages, not state.

## Data Pipeline
The PlaceBlockInteraction class is a data model that sits at the boundary between raw network bytes and structured game events.

**Outbound Flow (Client to Server):**
> Player Input -> Game Logic -> **new PlaceBlockInteraction()** -> `serialize()` -> Netty ByteBuf -> TCP/IP Stack

**Inbound Flow (Server to Client):**
> TCP/IP Stack -> Netty ByteBuf -> `deserialize()` -> **PlaceBlockInteraction instance** -> Network Event Handler -> Game Logic Update

