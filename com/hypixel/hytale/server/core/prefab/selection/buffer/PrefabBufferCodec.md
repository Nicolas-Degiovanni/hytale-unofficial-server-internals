---
description: Architectural reference for PrefabBufferCodec
---

# PrefabBufferCodec

**Package:** com.hypixel.hytale.server.core.prefab.selection.buffer
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface PrefabBufferCodec<T> extends PrefabBufferSerializer<T>, PrefabBufferDeserializer<T> {
}
```

## Architecture & Concepts
The PrefabBufferCodec interface is a composite contract that unifies serialization and deserialization logic for a specific data type. It serves as a foundational component within the server's data persistence and networking layers. By extending both PrefabBufferSerializer and PrefabBufferDeserializer, it establishes a mandatory bidirectional data transformation capability for any implementing class.

This design pattern is central to the engine's strategy for handling prefab data. Prefabs, which represent complex arrangements of game objects and their state, must be efficiently converted to and from a binary format (PrefabBuffer) for storage, network transmission, and replication. The PrefabBufferCodec provides the specific, type-aware logic required for this conversion, acting as the translator between a high-level Java object and its low-level binary representation.

Implementations of this interface are effectively data adapters, registered with a central serialization service that invokes them polymorphically when a corresponding data type needs to be processed.

## Lifecycle & Ownership
As an interface, PrefabBufferCodec has no lifecycle itself. The following applies to its concrete implementations.

- **Creation:** Implementations are typically stateless singletons or utility classes. They are discovered and instantiated by a dependency injection framework or a dedicated CodecRegistry during server bootstrap.
- **Scope:** A codec implementation is designed to be application-scoped. Once registered, it persists for the entire lifetime of the server process.
- **Destruction:** Codecs are not typically destroyed. They are held by the central registry until server shutdown.

## Internal State & Concurrency
- **State:** The contract implies statelessness. Implementations of PrefabBufferCodec **must not** maintain any mutable state related to a specific serialization or deserialization operation. Each method call should be an independent, idempotent transformation.
- **Thread Safety:** Implementations must be thread-safe. The server's networking and world-saving systems may operate on multiple threads, and a single codec instance will be shared to process data concurrently. Adherence to statelessness is the primary mechanism for achieving thread safety.

**WARNING:** Introducing mutable instance variables into a codec implementation will lead to severe data corruption and race conditions. All required context must be passed via method arguments.

## API Surface
The API surface is inherited entirely from the parent interfaces, PrefabBufferSerializer and PrefabBufferDeserializer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(T, PrefabBuffer) | void | O(N) | Inherited from PrefabBufferSerializer. Encodes the state of the object of type T into the provided PrefabBuffer. Complexity is proportional to the complexity of the object graph. |
| deserialize(PrefabBuffer) | T | O(N) | Inherited from PrefabBufferDeserializer. Decodes binary data from the PrefabBuffer to construct a new instance of type T. Complexity is proportional to the amount of data read. |

## Integration Patterns

### Standard Usage
A developer's primary interaction with this interface is to implement it for a custom component or data structure that needs to be part of a prefab. The implementation is then registered with the engine's serialization system.

```java
// 1. Implement the codec for a custom data type
public class CustomDataCodec implements PrefabBufferCodec<CustomData> {
    @Override
    public void serialize(CustomData data, PrefabBuffer buffer) {
        // write data fields to buffer
    }

    @Override
    public CustomData deserialize(PrefabBuffer buffer) {
        // read fields from buffer and construct new CustomData
        return new CustomData(...);
    }
}

// 2. Register the codec with the central registry (typically at startup)
// Note: The exact registry mechanism may vary.
codecRegistry.register(CustomData.class, new CustomDataCodec());
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Storing temporary data as instance fields within a codec is a critical error. This breaks re-entrancy and thread safety.
- **Direct Invocation:** Manually calling a specific codec's serialize or deserialize method. The engine's serialization orchestrator is responsible for selecting the correct codec at runtime. Bypassing this system breaks polymorphism and versioning.
- **Incomplete Implementation:** Since this interface combines a serializer and deserializer, both methods must be fully implemented for the contract to be useful in a client-server architecture.

## Data Pipeline
The PrefabBufferCodec is a critical transformation step in both outbound (serialization) and inbound (deserialization) data flows.

**Outbound (Serialization)**
> Flow:
> Game State Change -> Prefab System marks object for serialization -> Serialization Orchestrator retrieves **PrefabBufferCodec** for object type -> `serialize()` is called -> PrefabBuffer -> Network Layer / Disk Writer

**Inbound (Deserialization)**
> Flow:
> Network Packet / Disk Read -> PrefabBuffer -> Serialization Orchestrator retrieves **PrefabBufferCodec** for data type -> `deserialize()` is called -> New Game Object Instance -> World State Update

