---
description: Architectural reference for CodecKey
---

# CodecKey

**Package:** com.hypixel.hytale.codec.store
**Type:** Value Object / Identifier

## Definition
```java
// Signature
public class CodecKey<T> {
```

## Architecture & Concepts
The CodecKey class provides a type-safe, unique identifier for data serialization and deserialization handlers, known as Codecs. It is a foundational component of the engine's data codec system, designed to prevent runtime errors by enforcing type-safety at compile time.

Architecturally, CodecKey acts as a typed handle. Instead of using raw String identifiers to look up a Codec in a central registry, developers use an instance of CodecKey<T>, where T is the target data type. This pattern ensures that a request for a Codec capable of handling a Player object can only ever return a Codec<Player>, eliminating an entire class of type-mismatch bugs.

The class is intentionally minimal, containing only a String identifier. Its primary value lies in its generic type parameter, which links the key to a specific data contract.

## Lifecycle & Ownership
- **Creation:** CodecKey instances are intended to be defined as **public static final** constants, typically within a central registry class or alongside the data model they represent. This promotes reuse and establishes a canonical set of known codec types.
- **Scope:** When defined as static constants, their lifecycle is tied to the application's lifecycle. They are loaded with their containing class and persist until the application terminates.
- **Destruction:** Instances are managed by the Java garbage collector. Statically referenced keys are effectively never destroyed during normal operation.

## Internal State & Concurrency
- **State:** The CodecKey is **immutable**. Its internal state, the String id, is declared final and is assigned only once during construction. It does not cache any data or change after creation.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutability, instances can be safely shared, published, and accessed across multiple threads without any external synchronization or locking mechanisms.

## API Surface
The public contract is minimal, focusing exclusively on its function as an identifier.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CodecKey(id) | constructor | O(1) | Creates a new key with the specified string identifier. |
| getId() | String | O(1) | Returns the underlying string identifier of the key. |
| equals(o) | boolean | O(N) | Compares two keys based on their string identifiers. |
| hashCode() | int | O(N) | Computes a hash code based on the string identifier. |

## Integration Patterns

### Standard Usage
The canonical usage pattern involves defining keys as static constants and using them to retrieve a corresponding Codec from a registry or store.

```java
// In a central constants or model class
public static final CodecKey<Player> PLAYER = new CodecKey<>("hytale:player");

// In a service or game logic class
CodecStore store = context.getService(CodecStore.class);
Codec<Player> playerCodec = store.getCodec(PLAYER);
ByteBuffer buffer = playerCodec.encode(myPlayerInstance);
```

### Anti-Patterns (Do NOT do this)
- **Dynamic Instantiation in Loops:** Avoid creating new CodecKey instances repeatedly for the same identifier. This is inefficient and defeats the purpose of reusable, constant keys.
- **Using Raw Types:** Using a raw CodecKey without its generic parameter circumvents the compile-time type safety it is designed to provide, leading to unsafe type casts downstream.

## Data Pipeline
CodecKey does not process data itself; it is a critical addressing mechanism within the serialization pipeline. It initiates the lookup process for the correct data handler.

> Flow:
> System Logic -> **CodecKey<T>** -> CodecStore Lookup -> Codec<T> -> Serialization/Deserialization Engine

