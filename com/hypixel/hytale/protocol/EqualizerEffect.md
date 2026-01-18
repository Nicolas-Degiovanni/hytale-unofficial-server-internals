---
description: Architectural reference for EqualizerEffect
---

# EqualizerEffect

**Package:** com.hypixel.hytale.protocol
**Type:** Transient / Data Structure

## Definition
```java
// Signature
public class EqualizerEffect {
```

## Architecture & Concepts
The **EqualizerEffect** class is a Data Transfer Object (DTO) designed for network serialization. It represents a complete set of parameters for a multi-band audio equalizer. Its primary role is to serve as a well-defined data contract between the client and server for transmitting audio effect configurations.

This class is a fundamental component of the audio-visual protocol layer. It does not contain any logic for applying the effect; it is purely a data container. The serialization and deserialization logic is tightly coupled with the Netty ByteBuf, indicating its direct use in network packet handlers. The structure is optimized for performance, using a fixed-size block for numeric data and a variable-size block for the optional string identifier, preceded by a bitmask for null fields.

## Lifecycle & Ownership
- **Creation:** An **EqualizerEffect** instance is created under two primary circumstances:
    1.  **Inbound:** The static `deserialize` method is called by a network protocol decoder (e.g., a Netty ChannelInboundHandler) when a corresponding packet is read from the network stream. The network layer owns the newly created object.
    2.  **Outbound:** A game system, such as the Audio Engine, instantiates a new **EqualizerEffect** via its constructor to define an effect that needs to be sent to a remote peer.

- **Scope:** The object is transient and has a very short lifetime. It exists only long enough to be serialized into a network buffer or to have its data copied into the active audio processing graph after being deserialized. It is not intended to be stored or cached long-term.

- **Destruction:** The object is managed by the Java Garbage Collector. Once all references are released (typically after a single network read or write operation), it becomes eligible for collection. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The state of an **EqualizerEffect** is entirely contained within its public fields. It is a fully **mutable** object; its fields can be modified at any time after construction. It does not cache any external data or maintain connections to other systems.

- **Thread Safety:** This class is **not thread-safe**. All fields are public and non-volatile, and there are no internal locks or synchronization mechanisms.

    **WARNING:** Concurrent access from multiple threads will lead to race conditions and undefined behavior. Instances of **EqualizerEffect** should be confined to a single thread, such as a Netty event loop thread or the main game update thread. Do not share instances across thread boundaries without explicit external synchronization.

## API Surface
The public API is designed for serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | EqualizerEffect | O(N) | **[Factory]** Constructs an object by reading from a ByteBuf at a given offset. N is the length of the string `id`. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided ByteBuf. N is the length of the string `id`. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | Performs a non-deserializing check on a buffer to ensure it contains a structurally valid object. Crucial for preventing deserialization attacks. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this object will consume when serialized. |
| computeBytesConsumed(ByteBuf, int) | int | O(N) | Calculates the size of an object already serialized in a buffer without fully deserializing it. |

## Integration Patterns

### Standard Usage
The primary integration pattern involves a network handler decoding the object from an incoming buffer and passing it to a consumer system, such as an audio manager.

```java
// Example within a hypothetical Netty handler
ByteBuf incomingPacket = ...;

// 1. Validate the structure before attempting to read
ValidationResult result = EqualizerEffect.validateStructure(incomingPacket, 0);
if (!result.isOk()) {
    throw new ProtocolException("Invalid EqualizerEffect: " + result.getReason());
}

// 2. Deserialize into a concrete object
EqualizerEffect effect = EqualizerEffect.deserialize(incomingPacket, 0);

// 3. Pass the data object to the relevant system
audioManager.applyEffect(effect);
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Do not read an **EqualizerEffect** on one thread while another thread is writing to its public fields. This will cause data corruption.
- **Ignoring Validation:** Never call `deserialize` on untrusted data without first calling `validateStructure`. Skipping validation exposes the system to potential buffer overflows or denial-of-service attacks via malformed length fields.
- **Object Re-use:** Avoid re-using a single **EqualizerEffect** instance for multiple distinct effects. Because it is mutable, this can lead to subtle bugs where old data is not properly cleared. It is safer to create a new instance for each new effect.

## Data Pipeline
The class acts as a deserialization endpoint in the inbound data pipeline and a serialization source in the outbound pipeline.

> **Inbound Flow:**
> Raw TCP Stream -> Netty ByteBuf -> **EqualizerEffect.deserialize** -> In-Memory EqualizerEffect Instance -> Audio Engine State Update

> **Outbound Flow:**
> Game Event -> Audio Engine -> **new EqualizerEffect()** -> **EqualizerEffect.serialize** -> Netty ByteBuf -> Raw TCP Stream

