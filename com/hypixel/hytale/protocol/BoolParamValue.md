---
description: Architectural reference for BoolParamValue
---

# BoolParamValue

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class BoolParamValue extends ParamValue {
```

## Architecture & Concepts
The BoolParamValue class is a low-level, high-performance data structure within the Hytale network protocol layer. It serves as the canonical representation for a boolean value during binary serialization and deserialization. As a concrete implementation of the abstract ParamValue, it is part of a family of types that form the building blocks for all network messages.

Its design is strictly optimized for efficiency. By enforcing a fixed block size of exactly one byte, it allows the protocol engine to perform fast, predictable buffer operations without the need for size prefixes or terminators. The static methods for validation and deserialization operate directly on Netty ByteBuf objects, indicating its tight coupling with the underlying network stack. This class is not intended for use in general game logic; it is a specialized tool for the protocol codec.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1.  By the protocol deserialization pipeline when an incoming network buffer is parsed. The static factory method *deserialize* is the designated entry point for this path.
    2.  By higher-level game or service logic when constructing a packet to be sent over the network.

- **Scope:** The lifetime of a BoolParamValue instance is exceptionally short. It is designed to be ephemeral, existing only for the duration of a single packet's encoding or decoding cycle.

- **Destruction:** Instances are managed by the Java garbage collector. They become eligible for collection immediately after their internal state has been written to an outbound buffer or read by the consuming game logic. There are no manual cleanup requirements.

## Internal State & Concurrency
- **State:** Mutable. The core *value* field is public, allowing direct modification. However, the design intent is for instances to be treated as immutable value objects after their initial construction. Post-construction mutation is a significant anti-pattern.

- **Thread Safety:** **This class is not thread-safe.** All operations assume single-threaded access. The public mutable state and lack of synchronization primitives make it inherently unsafe for concurrent use. Instances MUST be confined to the thread that is processing the network packet, typically a Netty I/O worker thread.

## API Surface
The public API is minimal, focusing exclusively on serialization and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static BoolParamValue | O(1) | Factory method. Reads one byte from the buffer at a given offset and constructs a new instance. |
| serialize(ByteBuf) | int | O(1) | Writes the boolean state as a single byte (0 or 1) to the provided buffer. Returns bytes written. |
| computeSize() | int | O(1) | Returns the constant size of the serialized data, which is always 1. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-flight check to ensure the buffer contains enough data for a valid read. |
| clone() | BoolParamValue | O(1) | Creates a shallow copy of the object. |

## Integration Patterns

### Standard Usage
Interaction with BoolParamValue should always occur within the context of packet serialization or deserialization logic. It acts as an intermediary between raw bytes and the engine's native boolean type.

```java
// Example: Serializing an outgoing value
BoolParamValue playerIsInvulnerable = new BoolParamValue(true);
playerIsInvulnerable.serialize(networkBuffer);

// Example: Deserializing an incoming value
int currentOffset = ...;
ValidationResult result = BoolParamValue.validateStructure(networkBuffer, currentOffset);
if (!result.isOk()) {
    throw new ProtocolException("Invalid buffer for BoolParamValue: " + result.getReason());
}

BoolParamValue incomingFlag = BoolParamValue.deserialize(networkBuffer, currentOffset);
boolean isInvulnerable = incomingFlag.value;
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not modify the public *value* field after an instance has been created. Treat it as an immutable DTO to prevent unpredictable side effects during serialization.
- **Cross-Thread Sharing:** Never pass a BoolParamValue instance between threads. Create a new instance or extract the primitive boolean value instead.
- **Manual Deserialization:** Do not use `new BoolParamValue()` and then attempt to populate it by reading from a buffer manually. The static *deserialize* method is the only supported pathway for object creation from a byte stream.

## Data Pipeline
BoolParamValue is a fundamental component in the network data flow, acting as a translation point between the byte stream and in-memory game state.

> **Outbound Flow (Serialization):**
> Game State (boolean) -> **new BoolParamValue(state)** -> serialize() -> Netty ByteBuf -> Network Socket

> **Inbound Flow (Deserialization):**
> Network Socket -> Netty ByteBuf -> Protocol Parser -> **deserialize()** -> BoolParamValue Instance -> Game Logic (reads .value)

