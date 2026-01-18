---
description: Architectural reference for CodecMapCodec
---

# CodecMapCodec

**Package:** com.hypixel.hytale.codec.lookup
**Type:** Transient

## Definition
```java
// Signature
public class CodecMapCodec<T> extends StringCodecMapCodec<T, Codec<? extends T>> {
```

## Architecture & Concepts
The CodecMapCodec is a fundamental component of the data serialization framework, designed to handle polymorphism. It acts as a high-level dispatcher or a "codec of codecs". Its primary role is to serialize and deserialize objects where the concrete type is not known at compile time, but is instead determined at runtime from the data stream itself.

This is achieved by mapping string identifiers to specific Codec implementations.
- **During serialization**, the CodecMapCodec first writes a unique string identifier for the object's class to the data stream. It then delegates the serialization of the object's actual data to the specific Codec registered for that identifier.
- **During deserialization**, it first reads the string identifier from the stream. It uses this identifier to look up the corresponding Codec in its internal map and then invokes that specific Codec to parse the remainder of the data, correctly reconstructing the original concrete object.

This mechanism is critical for systems like network packets, entity data, or block types, where a stream might contain a heterogeneous collection of objects derived from a common base class T. This class is a thin, fluent wrapper over its parent, StringCodecMapCodec, specializing the mapped value to be a Codec.

### Lifecycle & Ownership
- **Creation:** A CodecMapCodec is instantiated directly by a developer or a higher-level system that needs to define a polymorphic serialization scheme. It is not managed by a dependency injection framework or a global registry. For example, a PacketRegistry would create and configure its own instance of a CodecMapCodec for Packets.
- **Scope:** The lifetime of a CodecMapCodec instance is bound to its owner. It persists as long as the system that configured it (e.g., the PacketRegistry) is in scope. It is not a global singleton and multiple distinct instances can exist to handle different polymorphic types.
- **Destruction:** The object is managed by the Java garbage collector. There are no manual cleanup or disposal methods. It is reclaimed once its owning object is no longer referenced.

## Internal State & Concurrency
- **State:** This object is highly stateful, but its state is typically established during an initialization phase. It internally maintains a map of string identifiers to Codec instances. This map is mutable and is populated via calls to the register method. After this initial configuration, the object is primarily used for read-only lookup operations during encoding and decoding.

- **Thread Safety:** This class is **not thread-safe** for modification. The registration process must be completed from a single thread before the codec is used for any serialization or deserialization operations. Concurrent calls to the register method will lead to unpredictable behavior and a corrupted internal state. Once fully configured, its use in serialization pipelines is generally safe, assuming the underlying map in the parent class is safe for concurrent reads.

**WARNING:** All registration calls must be completed before the CodecMapCodec is passed to any system that may operate on multiple threads, such as a network pipeline.

## API Surface
The primary API surface for this specific class is for configuration. The core serialization logic is inherited from its parent hierarchy.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(id, aClass, codec) | CodecMapCodec<T> | O(1) | Registers a Codec for a given string ID and class type. Returns itself for fluent chaining. |
| register(priority, id, aClass, codec) | CodecMapCodec<T> | O(1) | Registers a Codec with a specified priority, allowing for more complex override or ordering logic. |

## Integration Patterns

### Standard Usage
The intended usage is to instantiate the class and then use its fluent API to build a complete mapping of all possible subtypes. This configured instance is then used by a higher-level serialization system.

```java
// Example: Defining a codec for a polymorphic 'Event' base class
CodecMapCodec<Event> eventCodec = new CodecMapCodec<>("eventId");

// Use the fluent API to register all known event types
eventCodec
    .register("player_join", PlayerJoinEvent.class, new PlayerJoinEvent.Codec())
    .register("player_quit", PlayerQuitEvent.class, new PlayerQuitEvent.Codec())
    .register("chat_msg", ChatMessageEvent.class, new ChatMessageEvent.Codec());

// The fully configured 'eventCodec' is now ready to be used by a network
// or file I/O system to process streams of different Event types.
```

### Anti-Patterns (Do NOT do this)
- **Post-Initialization Registration:** Do not call register after the CodecMapCodec has been actively used for serialization. This can create a state mismatch, where a client and server have different mappings, leading to deserialization failures. All registrations should occur during a single, well-defined initialization phase.
- **Concurrent Registration:** Never call register from multiple threads. The internal map is not designed for concurrent writes. All configuration should be performed on the main application thread or within a synchronized initialization block.

## Data Pipeline
The CodecMapCodec acts as a routing layer in the data pipeline, directing data to the appropriate sub-codec.

**Serialization Flow:**
> Object (e.g., PlayerJoinEvent) -> **CodecMapCodec** -> Writes "player_join" ID to Stream -> Delegates to PlayerJoinEvent.Codec -> PlayerJoinEvent.Codec writes object fields to Stream

**Deserialization Flow:**
> Data Stream -> **CodecMapCodec** reads "player_join" ID -> Looks up PlayerJoinEvent.Codec from internal map -> Delegates to PlayerJoinEvent.Codec -> PlayerJoinEvent.Codec reads fields and constructs a new PlayerJoinEvent instance

