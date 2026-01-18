---
description: Architectural reference for AOECircleSelector
---

# AOECircleSelector

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class AOECircleSelector extends Selector {
```

## Architecture & Concepts
The AOECircleSelector is a specialized Data Transfer Object used within the Hytale network protocol. It represents a circular area-of-effect (AOE) selector, a fundamental concept in game logic for abilities, spells, or environmental triggers.

Its primary architectural role is to serve as a concrete, serializable data structure. It is not a service or manager; it is a passive container for data that describes a circular region in 3D space, defined by a radius and an optional center offset. The class is designed for extreme performance and minimal memory allocation, evident in its fixed-size binary layout and direct serialization to and from Netty's ByteBuf.

This class is a building block for more complex network packets. It is never used in isolation but is instead embedded within larger protocol messages that define game actions or states.

### Lifecycle & Ownership
- **Creation:** Instances are created in two scenarios:
    1. By game logic on the sending side (client or server) to define the parameters of an action.
    2. By the protocol layer's deserialization logic, which invokes the static `deserialize` factory method when decoding an incoming network packet.
- **Scope:** The lifetime of an AOECircleSelector instance is exceptionally short and transactional. It exists only for the duration of a single network operation (packet creation or packet processing). It is not intended to be stored long-term.
- **Destruction:** The object is managed by the Java Garbage Collector. As it is a simple DTO with no external resources, it is cleaned up as soon as it falls out of the scope of the method processing it. There is no manual destruction or cleanup required.

## Internal State & Concurrency
- **State:** The internal state is **mutable** and consists of two public fields: a floating-point `range` and a nullable `Vector3f` named `offset`. The fixed-size nature of its serialized form (17 bytes) is a critical design constraint.
    - A single byte (`nullBits`) is used as a bitfield during serialization to efficiently encode the presence or absence of the nullable `offset` field. This is a common pattern in the Hytale protocol to optimize for network bandwidth.
- **Thread Safety:** This class is **not thread-safe**. It is a plain data object with no internal locking or synchronization. It is designed to be created, populated, and processed within a single thread, typically a Netty I/O thread or the main game logic thread.

**WARNING:** Sharing an AOECircleSelector instance across multiple threads without external, explicit synchronization will lead to race conditions and unpredictable behavior. Do not store instances in shared collections or access them from parallel tasks.

## API Surface
The public API is minimal and focused entirely on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | AOECircleSelector | O(1) | **Static Factory.** Reads 17 bytes from the buffer and constructs a new instance. Does not advance the buffer's reader index. |
| serialize(ByteBuf) | int | O(1) | Writes the object's state into the provided buffer as 17 bytes. Returns the number of bytes written. |
| computeSize() | int | O(1) | Calculates the size of the serialized object. Always returns the constant 17. |
| clone() | AOECircleSelector | O(1) | Creates a deep copy of the object. The nested Vector3f is also cloned if present. |

## Integration Patterns

### Standard Usage
The class is intended to be used as part of a larger packet's serialization or deserialization process. Game logic creates and populates the object, which is then passed to a serializer.

```java
// Example: Defining a spell effect
AOECircleSelector spellArea = new AOECircleSelector();
spellArea.range = 5.0f;
spellArea.offset = new Vector3f(0, 1.0f, 0);

// This object would then be assigned to a field in a larger
// network packet before that packet is sent.
SomeGameActionPacket packet = new SomeGameActionPacket();
packet.selector = spellArea;
networkManager.send(packet);
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not modify an AOECircleSelector instance after it has been passed to a serialization routine. The network layer may process buffers asynchronously.
- **Long-Term Storage:** Do not hold references to AOECircleSelector instances in caches or long-lived game state objects. They are meant to be transient. Create new instances as needed.
- **Cross-Thread Sharing:** Never pass an instance from the main game thread to a worker thread (or vice-versa) without deep-cloning it first.

## Data Pipeline
The AOECircleSelector is a data payload that travels through the network stack. It does not actively process data itself.

**Outbound Flow (Sending Data):**
> Game Logic Instantiation -> **AOECircleSelector** -> Packet Serialization -> Netty ByteBuf -> Network Socket

**Inbound Flow (Receiving Data):**
> Network Socket -> Netty ByteBuf -> Packet Deserialization -> **AOECircleSelector** -> Game Logic Processing

