---
description: Architectural reference for UpdateType
---

# UpdateType

**Package:** com.hypixel.hytale.protocol
**Type:** Enum Constant

## Definition
```java
// Signature
public enum UpdateType {
```

## Architecture & Concepts
The UpdateType enum is a fundamental component of the network protocol layer, responsible for encoding the nature of a state change for a replicated object. It serves as a type-safe discriminator that instructs the client or server on how to process an incoming data payload.

This pattern is central to the engine's delta-based replication strategy. Instead of transmitting the entire state of an entity or component on every update, the server sends a more compact message containing only the changed data, prefixed with an UpdateType. This design significantly reduces network bandwidth and processing overhead.

- **Init:** Signals the creation of a new object. The associated payload contains the full initial state.
- **AddOrUpdate:** Signals a modification to an existing object. The payload contains only the fields that have changed.
- **Remove:** Signals the destruction of an object. The payload typically contains only the object's unique identifier.

The static `fromValue` method acts as a deserialization factory, converting a raw integer from the network byte stream into a safe, managed enum instance.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine during class loading. They are compile-time singletons.
- **Scope:** Application-wide. An instance of UpdateType exists for the entire lifetime of the process once the class is loaded.
- **Destruction:** Instances are reclaimed by the JVM only upon application shutdown. There is no manual lifecycle management.

## Internal State & Concurrency
- **State:** Deeply immutable. Each enum constant is a singleton with a final, primitive `value` field. The static `VALUES` array is also final and populated only once, acting as a read-only cache.
- **Thread Safety:** Inherently thread-safe. As immutable singletons, UpdateType constants can be safely shared and read across any number of threads without synchronization. The static factory method `fromValue` is also fully re-entrant and thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer representation for network serialization. |
| fromValue(int value) | UpdateType | O(1) | Deserializes an integer from a network stream into an UpdateType. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is almost always used within a switch statement in a network packet handler to delegate logic based on the type of update received.

```java
// Typical usage inside a packet processing method
void handleEntityUpdate(int entityId, int updateTypeValue, Buffer data) {
    UpdateType type = UpdateType.fromValue(updateTypeValue);

    switch (type) {
        case Init:
            createEntityFrom(entityId, data);
            break;
        case AddOrUpdate:
            updateEntityWith(entityId, data);
            break;
        case Remove:
            world.destroyEntity(entityId);
            break;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Using Magic Numbers:** Never use raw integers like 0, 1, or 2 in your application logic. This defeats the purpose of the enum, making the code brittle and difficult to read. Always use the named constants like UpdateType.Init.
- **Unsafe Deserialization:** Do not attempt to manually map integers to enums without bounds checking. Always use the provided `fromValue` method, which safely handles invalid data from the network.

## Data Pipeline
UpdateType is a critical control-flow component in the data pipeline, directing how raw network data is applied to the game state.

> Flow:
> Network Byte Stream -> Protocol Framer -> Integer `updateTypeValue` -> **UpdateType.fromValue()** -> Game State Replicator -> Entity Created/Updated/Removed

