---
description: Architectural reference for PortalDef
---

# PortalDef

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class PortalDef {
```

## Architecture & Concepts
The PortalDef class is a pure Data Transfer Object, not a service or manager. Its sole responsibility is to define the binary wire format for portal configuration data exchanged between the Hytale client and server. It serves as a strict data contract, ensuring that both endpoints can correctly interpret the byte stream associated with portal definitions.

This class is a fundamental component of the network protocol layer. Its design is heavily optimized for network efficiency and parsing performance, incorporating several key patterns:

*   **Hybrid Layout:** It combines a fixed-size block for predictable, always-present data (explorationSeconds, breachSeconds) with a variable-size block for optional or dynamic data (nameKey). This allows for rapid, direct-offset reads of the fixed portion.
*   **Bitmasking for Nulls:** A single byte, `nullBits`, is used as a bitfield to indicate the presence or absence of nullable fields. This is significantly more space-efficient than sending sentinel values or length prefixes of zero for absent data.
*   **Variable-Length Encoding:** The length of the variable-sized `nameKey` string is encoded using VarInt, which uses fewer bytes for smaller integers, a common case for string lengths in game protocols.

This object represents a serialized *message*, not a live game entity. It is a snapshot of data at a specific point in time, intended for transmission over the network.

## Lifecycle & Ownership
- **Creation:**
    - **Receiving End:** Instantiated exclusively by the static `deserialize` factory method. This method is called by a higher-level packet parser (e.g., a hypothetical `S2CInterfaceUpdatePacket` parser) when it needs to decode a PortalDef from a raw ByteBuf payload.
    - **Sending End:** Instantiated directly via its constructors (`new PortalDef(...)`) by game logic that needs to construct a network packet. The fields are populated with current game state before the object is passed to a serializer.

- **Scope:** Transient and short-lived. The lifetime of a PortalDef instance is tightly coupled to the processing of a single network packet. It is created, its data is consumed by the relevant game system, and it becomes eligible for garbage collection immediately afterward.

- **Destruction:** Handled by the Java Garbage Collector. There are no manual cleanup or `close` methods. Holding long-term references to PortalDef instances is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** The object's state is fully **Mutable**. All data fields are public and can be modified at any time after instantiation. This design choice prioritizes performance and ease of use for serializers and deserializers, which often populate objects in multiple steps.

- **Thread Safety:** This class is **not thread-safe**. It contains no locks or other synchronization primitives. It is designed to be created, populated, and read within the confines of a single thread, typically a Netty I/O worker or the main game logic thread.

    **Warning:** Concurrent reads and writes to a PortalDef instance from multiple threads will result in data corruption and undefined behavior. Any cross-thread usage must be managed externally with explicit synchronization.

## API Surface
The public contract is dominated by static methods for serialization, deserialization, and validation, operating directly on Netty ByteBufs.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static PortalDef | O(N) | Constructs a new PortalDef by reading from a buffer at a given offset. N is the length of the nameKey string. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the instance's state into the provided buffer according to the defined wire format. |
| computeBytesConsumed(ByteBuf, int) | static int | O(1) | Calculates the total byte size of a serialized PortalDef within a buffer without allocating a new object. Critical for parsers that need to skip records. |
| computeSize() | int | O(N) | Calculates the number of bytes this instance will occupy when serialized. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a low-cost check to verify if a buffer likely contains a valid PortalDef structure. Does not perform a full deserialization. Used for network security and stability. |

## Integration Patterns

### Standard Usage
PortalDef is never used in isolation. It is always embedded within a larger network packet's serialization or deserialization logic. A system receiving data will use the static `deserialize` method to hydrate the object from a buffer.

```java
// Hypothetical packet processing logic
void handleInterfaceUpdate(ByteBuf packetPayload) {
    // Assume the payload starts with a PortalDef at offset 0
    ValidationResult result = PortalDef.validateStructure(packetPayload, 0);
    if (!result.isOk()) {
        throw new ProtocolException("Invalid PortalDef received: " + result.getReason());
    }

    PortalDef portalInfo = PortalDef.deserialize(packetPayload, 0);

    // Use the deserialized data to update game state
    game.getPortalManager().updatePortal(portalInfo.nameKey, portalInfo);
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term State:** Do not store PortalDef instances as part of your long-term game state. They are network models, not domain models. Map the data from a PortalDef to a dedicated game state class upon receipt.
- **Manual Serialization:** Do not attempt to manually read or write the fields to a buffer. The binary layout is complex (bitmasks, VarInts). Always use the provided `serialize` and `deserialize` methods to ensure protocol compliance.
- **Ignoring Validation:** Bypassing `validateStructure` on untrusted input can expose the server or client to resource exhaustion attacks (e.g., a maliciously crafted large string length) or buffer overflow reads.

## Data Pipeline
PortalDef acts as a deserialization endpoint in the data pipeline, converting a raw byte stream into a structured Java object that game logic can understand.

> **Flow (Receiving):**
> Raw TCP Stream -> Netty ByteBuf -> Parent Packet Deserializer -> **PortalDef.deserialize** -> In-Memory PortalDef Instance -> Game Logic Consumer

> **Flow (Sending):**
> Game Logic -> New PortalDef Instance -> **portalDef.serialize(buffer)** -> Parent Packet Serializer -> Netty ByteBuf -> Raw TCP Stream

