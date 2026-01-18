---
description: Architectural reference for PrimitiveCodec
---

# PrimitiveCodec

**Package:** com.hypixel.hytale.codec
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface PrimitiveCodec {
}
```

## Architecture & Concepts
The PrimitiveCodec interface serves as a foundational contract within the engine's serialization framework. It is a marker interface, meaning it declares no methods of its own. Its sole purpose is to establish a common type for all codecs responsible for serializing and deserializing Java's primitive data types (and their boxed equivalents) into a byte-stream representation for network transmission or disk storage.

This interface is a core component of the data serialization pipeline. A central CodecRegistry or a similar service discovers and manages all implementations of PrimitiveCodec. When a higher-level system, such as a PacketSerializer, needs to encode a field, it queries the registry for the appropriate codec based on the field's data type. This design decouples the high-level object serialization logic from the low-level, byte-level manipulation of individual data types, promoting modularity and extensibility.

Implementations are expected to be highly optimized, stateless, and focused on a single data type (e.g., an IntCodec for integers, a BooleanCodec for booleans).

## Lifecycle & Ownership
As an interface, PrimitiveCodec itself has no lifecycle. The following describes the lifecycle of its concrete implementations.

- **Creation:** Implementations are typically instantiated once at application startup. They are discovered and registered with a central, engine-wide CodecRegistry during the bootstrap sequence.
- **Scope:** Implementations are singletons that persist for the entire application session. They are fundamental, low-level services required for all data I/O.
- **Destruction:** Instances are destroyed during application shutdown when the managing CodecRegistry is cleared.

## Internal State & Concurrency
- **State:** All implementations of PrimitiveCodec **must be stateless**. A codec should be a pure function that transforms an input to an output without retaining any information between invocations. Caching or maintaining any form of internal state is a severe violation of this contract.
- **Thread Safety:** Due to their stateless nature, all implementations are inherently thread-safe. A single codec instance can be safely shared and invoked by multiple threads concurrently without the need for locks or synchronization. This is critical for performance in a multi-threaded networking environment.

**WARNING:** Introducing state into a PrimitiveCodec implementation will lead to non-deterministic serialization errors, data corruption, and severe concurrency bugs.

## API Surface
The PrimitiveCodec interface itself exposes no public methods. It acts as a marker to identify and group concrete codec implementations. Concrete implementations are expected to provide methods for encoding and decoding, which are typically invoked by the serialization framework via a more specific sub-interface or reflection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| (None) | - | - | This is a marker interface and has no public API contract. |

## Integration Patterns

### Standard Usage
Developers do not interact with PrimitiveCodec implementations directly. Instead, they rely on higher-level serialization services that use these codecs under the hood. The framework automatically selects and invokes the correct codec.

```java
// Hypothetical high-level serialization service
// The service internally finds and uses the correct PrimitiveCodec
// for the integer and string fields.
PacketBuffer buffer = new PacketBuffer();
serializer.writeObject(buffer, myGameObject); // Uses various PrimitiveCodecs
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Creating a codec that stores data between calls is the most critical anti-pattern. This breaks thread safety and leads to unpredictable behavior.
- **Direct Invocation:** Manually looking up and invoking a specific codec (e.g., `new IntCodec().write(buffer, 123)`) bypasses the central registry and couples game logic to low-level serialization details. Always use the engine's primary serialization facade.

## Data Pipeline
PrimitiveCodec implementations are the final, low-level stage in the outbound data pipeline and the first stage in the inbound pipeline. They are the components that perform the direct translation between in-memory data types and raw bytes.

> Outbound Flow:
> Game Object -> Field Accessor -> Serializer Facade -> Codec Registry -> **PrimitiveCodec Implementation** -> Byte Buffer

> Inbound Flow:
> Byte Buffer -> **PrimitiveCodec Implementation** -> Deserializer Facade -> Object Hydrator -> Game Object

