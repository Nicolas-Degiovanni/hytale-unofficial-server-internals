---
description: Architectural reference for DirectDecodeCodec
---

# DirectDecodeCodec<T>

**Package:** com.hypixel.hytale.codec
**Type:** Contract Interface

## Definition
```java
// Signature
public interface DirectDecodeCodec<T> extends Codec<T> {
```

## Architecture & Concepts
The DirectDecodeCodec interface defines a specialized, high-performance contract for BSON deserialization. It is a core component of the engine's data serialization framework, sitting between raw BSON data streams (from network packets or disk) and the live game state objects.

Unlike a standard Codec which typically instantiates a new object from the source data, a DirectDecodeCodec operates on a pre-existing target object. This pattern is known as **in-place deserialization**. Its primary purpose is to minimize memory allocation and reduce garbage collector pressure by reusing objects instead of creating new ones for every update. This is critical for performance in systems that process a high volume of state updates, such as player position synchronization or entity property changes.

Implementations of this interface are responsible for parsing a BsonValue and directly mutating the fields of the provided target object instance.

## Lifecycle & Ownership
As an interface, DirectDecodeCodec itself does not have a lifecycle. It is a compile-time contract. The lifecycle concerns apply to its concrete implementations.

- **Creation:** Implementations are typically stateless singletons, instantiated once during application bootstrap by a dependency injection framework or a central service registry.
- **Scope:** An implementation's lifecycle is tied to the CodecRegistry that holds it. It is expected to be available for the entire application session.
- **Destruction:** Implementations are garbage collected when the application shuts down and the classloader is unloaded. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** The interface contract is stateless. Implementations **must** be stateless and fully re-entrant. They should not hold any data related to a specific decoding operation between method calls. All necessary state is passed in via method arguments (the BsonValue, the target object, and ExtraInfo).
- **Thread Safety:** Implementations must be thread-safe. The engine's networking and persistence layers may invoke codecs from multiple threads concurrently. Any shared resources used within an implementation must be managed with appropriate synchronization. However, the target object being mutated (`T var2`) is **not** guaranteed to be thread-safe; synchronization of the target object is the responsibility of the caller.

## API Surface
The DirectDecodeCodec extends the base Codec interface but adds a more specialized method for in-place decoding.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, T, ExtraInfo) | void | O(N) | Deserializes the BsonValue data directly into the provided object instance `var2`. N is the complexity of the BSON data. Throws CodecException on parsing or validation errors. |

## Integration Patterns

### Standard Usage
Implementations are registered with a central CodecRegistry. The system then uses the registry to find the appropriate codec to apply updates to existing game objects. This pattern is fundamental to the client-side entity interpolation and prediction systems.

```java
// Example of a system using a DirectDecodeCodec
// Assume 'registry' is a populated CodecRegistry and 'player' is an existing object

BsonValue positionUpdate = receiveNetworkData();
DirectDecodeCodec<Player> codec = registry.getDirectCodec(Player.class);

// The player object is mutated directly, no new Player is created
codec.decode(positionUpdate, player, extraInfo);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not store any per-request or session-specific state as fields within a codec implementation. This will break in a multi-threaded environment and lead to severe data corruption.
- **Incorrect Target:** Do not use this interface for objects that are conceptually immutable or are not designed for reuse. The contract is explicitly for mutable, stateful objects.
- **Ignoring Inheritance:** If an object's parent class has decodable fields, the implementation must remember to call the parent's codec or handle its fields manually. Forgetting to do so results in partial or corrupted object state.

## Data Pipeline
This interface is a key stage in the data deserialization pipeline, focused on efficiency by avoiding object allocation.

> Flow:
> BSON Byte Stream (Network/Disk) -> BSON Parser -> BsonValue -> CodecRegistry -> **DirectDecodeCodec Implementation** -> Mutated Game Object State

