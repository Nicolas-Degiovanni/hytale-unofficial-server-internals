---
description: Architectural reference for ObjectCodecMapCodec
---

# ObjectCodecMapCodec<K, T>

**Package:** com.hypixel.hytale.codec.lookup
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class ObjectCodecMapCodec<K, T> extends ACodecMapCodec<K, T, Codec<? extends T>> {
```

## Architecture & Concepts
The ObjectCodecMapCodec is a critical component of the Hytale serialization framework, acting as a **polymorphic dispatcher** or **multiplexer**. Its primary architectural role is to enable the encoding and decoding of object hierarchies where a base type can be represented by one of many concrete subtypes.

This class solves the classic serialization problem for inheritance. For example, a game event system might have a base type *Event*, with concrete implementations like *PlayerJoinEvent* and *BlockBreakEvent*. When serializing a list of these events, the system must know which concrete type to instantiate upon deserialization.

The ObjectCodecMapCodec achieves this by prepending a unique key to the serialized data of the object.

*   **During Encoding:** It first identifies the concrete class of the object being serialized. It looks up the corresponding key in its internal registry, writes that key to the output stream, and then delegates the serialization of the object's actual data to the specific Codec registered for that type.
*   **During Decoding:** It first reads the key from the input stream. It uses this key to look up the appropriate Codec for the concrete subtype, and then invokes that Codec to deserialize the rest of the data stream into a fully-formed object.

This mechanism makes it a foundational building block for network packets, component-based entity systems, and complex configuration files. It is a concrete implementation of the more generic ACodecMapCodec, tailored for the common case of mapping a key directly to a Codec for a subtype.

## Lifecycle & Ownership
- **Creation:** An ObjectCodecMapCodec is instantiated directly via its constructor, typically during the static initialization phase of a larger system or registry. It is not managed by a dependency injection framework and is intended to be constructed manually.
- **Scope:** The object's lifecycle is tied to the lifecycle of the system that defines it. In most cases, it is defined once as part of a static, final Codec field and persists for the entire duration of the application session.
- **Destruction:** The instance is eligible for garbage collection when its containing class or registry is unloaded, which typically only occurs at application shutdown.

## Internal State & Concurrency
- **State:** The internal state is highly **mutable** during the initial configuration phase. Each call to the register method adds an entry to its internal lookup maps. After this bootstrap phase, the object's state should be considered **effectively immutable**. No further registrations should occur.

- **Thread Safety:** This class is **not thread-safe** during its configuration phase. All calls to the register method **must** be performed by a single thread before the Codec is used in a multi-threaded environment. The core encode and decode operations, inherited from the parent class, are thread-safe, assuming the registration phase is complete and no further mutations occur.

## API Surface
The primary API is designed for a fluent builder pattern to configure the key-to-codec mappings. The core serialization methods are inherited from the Codec interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(id, aClass, codec) | ObjectCodecMapCodec | O(1) | Registers a subtype, its identifying key, and its dedicated Codec. Throws an exception if the key is already registered. Returns itself for chaining. |
| register(priority, id, aClass, codec) | ObjectCodecMapCodec | O(1) | Registers a subtype with a given priority, allowing for fine-grained control over lookup order in special cases. |

## Integration Patterns

### Standard Usage
The ObjectCodecMapCodec is intended to be defined once and configured using its fluent API during static initialization. The resulting Codec is then used throughout the application.

```java
// Example: Defining a Codec for a polymorphic "Effect" base type.
public static final Codec<Effect> EFFECT_CODEC = new ObjectCodecMapCodec<String, Effect>("effect_type", new StringCodec())
    .register("hytale:heal", HealEffect.class, HEAL_EFFECT_CODEC)
    .register("hytale:speed", SpeedEffect.class, SPEED_EFFECT_CODEC)
    .register("hytale:poison", PoisonEffect.class, POISON_EFFECT_CODEC);
```

### Anti-Patterns (Do NOT do this)
- **Lazy Registration:** Do not add registrations to the Codec after it has been put into use. Modifying the internal map during active serialization across multiple threads will lead to race conditions, ConcurrentModificationExceptions, and non-deterministic behavior.

- **Redundant Instantiation:** Do not create a new ObjectCodecMapCodec for each serialization operation. This is highly inefficient and defeats the purpose of a reusable Codec. Define it as a static final constant.

## Data Pipeline
The ObjectCodecMapCodec acts as a routing layer in the data stream, prepending a type discriminator to the payload.

> **Encoding Flow:**
> Concrete Object (e.g., SpeedEffect) -> **ObjectCodecMapCodec** -> Looks up key "hytale:speed" -> Writes key to stream -> Delegates to SpeedEffectCodec -> SpeedEffectCodec writes its data -> Final Serialized Stream

> **Decoding Flow:**
> Serialized Stream -> **ObjectCodecMapCodec** -> Reads key "hytale:speed" -> Looks up SpeedEffectCodec from key -> Delegates remainder of stream to SpeedEffectCodec -> SpeedEffectCodec constructs a SpeedEffect object -> Final Deserialized Object

