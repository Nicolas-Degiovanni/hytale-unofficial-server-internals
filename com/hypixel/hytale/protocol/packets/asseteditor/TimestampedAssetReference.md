---
description: Architectural reference for TimestampedAssetReference
---

# TimestampedAssetReference

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class TimestampedAssetReference {
```

## Architecture & Concepts

TimestampedAssetReference is a fundamental data structure, not a service or manager. Its sole purpose is to serve as a Data Transfer Object (DTO) for the Hytale network protocol, specifically for systems related to the asset editor. It encapsulates a reference to a game asset via an AssetPath and couples it with a string-based timestamp.

The core architectural significance of this class lies in its highly optimized custom binary serialization format. This format is designed for minimal size and maximum performance over the network, eschewing more verbose, self-describing formats like JSON. The serialization strategy employs a bitmask field, known as *nullBits*, to efficiently flag the presence or absence of its nullable fields. This is followed by a fixed-size block containing relative offsets to the variable-length data blocks. This design allows for extremely fast parsing and validation, as a reader can determine the full size of the object and validate its structure without needing to deserialize every field.

This class is a foundational component for any real-time asset synchronization or live-editing functionality between the game client and a server or external tool.

## Lifecycle & Ownership

-   **Creation:** Instances are primarily created by the network protocol layer during the deserialization of an incoming data stream. A packet handler will invoke the static *deserialize* method on a raw ByteBuf. Alternatively, game logic may instantiate this object directly when preparing to send an asset reference to a remote endpoint.
-   **Scope:** Transient and short-lived. The lifecycle of a TimestampedAssetReference is typically bound to the scope of a single network packet processing event. It is created, its data is consumed by a handler, and it is then eligible for garbage collection.
-   **Destruction:** Managed exclusively by the Java Garbage Collector. There are no manual cleanup or resource release methods. Once all references are dropped, typically after a packet handler completes its execution, the object is destroyed.

## Internal State & Concurrency

-   **State:** Mutable. The class is a simple data container with public, non-final fields for *path* and *timestamp*. Its state is defined entirely by the data it was constructed with or deserialized from. It performs no caching and has no internal logic beyond serialization.
-   **Thread Safety:** **This class is not thread-safe.** Direct access to its public fields makes it inherently unsafe for concurrent modification or for reading while another thread writes.

    **Warning:** Instances of TimestampedAssetReference must not be shared across threads without explicit, external synchronization. It is designed for use within a single-threaded context, such as a Netty event loop or a specific game thread.

## API Surface

The public contract is dominated by static methods for serialization, deserialization, and validation, reflecting its role as a protocol-level DTO.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | TimestampedAssetReference | O(N) | **[Primary Constructor]** Deserializes an object from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Serializes the object's state into the provided ByteBuf according to the custom binary format. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a security and integrity check on the binary data in a buffer without full deserialization. Critical for preventing protocol attacks. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total number of bytes the serialized object occupies in a buffer. |
| computeSize() | int | O(N) | Calculates the number of bytes the object will consume when serialized. |
| clone() | TimestampedAssetReference | O(M) | Creates a shallow copy of the object. The timestamp is shared; the AssetPath is cloned. M is the complexity of cloning the AssetPath. |

## Integration Patterns

### Standard Usage

The canonical use case is within a network packet handler. The handler receives a buffer, validates its structure, and then deserializes it into an object for further processing by game logic.

```java
// Executed within a network thread or packet processor
void handleAssetUpdate(ByteBuf packetBuffer) {
    // 1. Validate the structure before processing to ensure safety
    ValidationResult result = TimestampedAssetReference.validateStructure(packetBuffer, packetBuffer.readerIndex());
    if (!result.isValid()) {
        throw new ProtocolException("Invalid TimestampedAssetReference: " + result.error());
    }

    // 2. Deserialize into a usable object
    TimestampedAssetReference ref = TimestampedAssetReference.deserialize(packetBuffer, packetBuffer.readerIndex());

    // 3. Pass the DTO to the appropriate system
    AssetService.INSTANCE.processReference(ref);
}
```

### Anti-Patterns (Do NOT do this)

-   **Ignoring Validation:** Never call *deserialize* on a buffer received from an untrusted source without first calling *validateStructure*. Failure to do so can result in uncaught exceptions, buffer over-reads, and potential denial-of-service vulnerabilities.
-   **Long-Term Storage:** Do not hold references to this object in long-lived collections or caches. It is designed as a transient object for data transfer, not as a persistent state container.
-   **Concurrent Modification:** Do not pass an instance to another thread while retaining a reference to it. If data must be passed between threads, create a defensive copy using the copy constructor or the *clone* method.

## Data Pipeline

TimestampedAssetReference acts as a data payload during its journey through the network stack.

**Outbound (Serialization Flow):**
> Game Logic -> `new TimestampedAssetReference()` -> `serialize(ByteBuf)` -> Netty Channel -> Network Socket

**Inbound (Deserialization Flow):**
> Network Socket -> Netty Channel -> ByteBuf -> `validateStructure()` -> **`TimestampedAssetReference.deserialize()`** -> Packet Handler -> Game Logic

