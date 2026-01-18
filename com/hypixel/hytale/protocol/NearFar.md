---
description: Architectural reference for NearFar
---

# NearFar

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class NearFar {
```

## Architecture & Concepts
The NearFar class is a low-level Data Transfer Object (DTO) used within the Hytale network protocol layer. It is not a service or manager, but rather a fundamental data structure representing a pair of floating-point values for near and far clipping planes. Its primary role is to provide a standardized, fixed-size representation for serialization and deserialization over the network.

This class is designed for maximum performance and minimal overhead. The public fields, static deserialization methods, and fixed-size constants (FIXED_BLOCK_SIZE = 8) are characteristic of a highly optimized protocol entity. It acts as a self-contained "protocol struct," encapsulating the logic required to read and write its state to a Netty ByteBuf, ensuring consistent byte ordering (Little Endian) across the client and server.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1.  By a higher-level protocol decoder calling the static NearFar.deserialize method when processing an incoming network packet.
    2.  By game logic (e.g., a rendering or camera system) that needs to construct a packet to be sent, instantiating it directly via `new NearFar(near, far)`.

- **Scope:** The lifecycle of a NearFar instance is extremely short and ephemeral. It is intended to exist only for the duration of a single packet encoding or decoding operation. It is a value object, not a long-lived entity.

- **Destruction:** Instances are managed by the Java Garbage Collector and are typically eligible for collection immediately after the network packet has been processed or serialized. No manual resource management is required.

## Internal State & Concurrency
- **State:** The state is fully mutable and exposed via public fields (near, far). This design prioritizes direct, high-performance access over encapsulation, which is a common trade-off in low-level networking code. The object itself is a simple container for its two float values.

- **Thread Safety:** **This class is not thread-safe.** Direct access to its public fields from multiple threads without external synchronization will lead to race conditions and data corruption. It is designed to be operated on by a single thread at a time, such as a Netty I/O worker thread or the main game logic thread.

    **WARNING:** Do not share instances of NearFar across threads. If data must be passed between threads, create a new copy.

## API Surface
The public API is focused exclusively on serialization, deserialization, and validation within a network buffer context.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | NearFar | O(1) | **Static Factory.** Deserializes 8 bytes from the buffer at the given offset into a new NearFar instance. |
| serialize(ByteBuf) | void | O(1) | Writes the instance's near and far values (8 bytes) into the provided buffer. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static Method.** Performs a pre-flight check to ensure the buffer contains enough readable bytes for a valid structure. |
| computeSize() | int | O(1) | Returns the constant size of the serialized structure, which is always 8. |

## Integration Patterns

### Standard Usage
NearFar is almost always used by an automated protocol codec. A typical manual interaction involves creating an instance to populate an outgoing packet or reading values from a freshly deserialized instance.

```java
// Example: Preparing an outgoing camera settings packet
CameraSettingsPacket packet = new CameraSettingsPacket();
packet.clippingPlanes = new NearFar(0.1f, 1000.0f);

// The protocol encoder will later call serialize internally
// packet.clippingPlanes.serialize(outputByteBuf);
```

```java
// Example: Reading from an incoming packet
// This is typically done by a packet handler
NearFar planes = NearFar.deserialize(incomingByteBuf, offset);
float nearPlane = planes.near;
// ... use the value in game logic
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not retain instances of NearFar in caches or as part of a long-lived game state object. It is a transient DTO for network communication only. Convert its values to a more stable internal data structure if persistence is needed.
- **Concurrent Modification:** Never modify the public `near` or `far` fields from one thread while another thread might be reading them or serializing the object. This is a critical violation of its intended use.
- **Manual Serialization:** Avoid writing the float values to a buffer manually. Always use the provided `serialize` and `deserialize` methods to guarantee correct byte ordering (Little Endian) and structure.

## Data Pipeline

The NearFar class is a data payload that flows through the network stack.

> **Inbound Flow:**
> Raw TCP Socket -> Netty ByteBuf -> Protocol Decoder -> **NearFar.deserialize()** -> Game Packet Object -> Game Logic

> **Outbound Flow:**
> Game Logic -> new NearFar() -> Game Packet Object -> Protocol Encoder -> **NearFar.serialize()** -> Netty ByteBuf -> Raw TCP Socket

