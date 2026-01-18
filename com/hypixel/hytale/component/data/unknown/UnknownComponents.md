---
description: Architectural reference for UnknownComponents
---

# UnknownComponents

**Package:** com.hypixel.hytale.component.data.unknown
**Type:** Data Component

## Definition
```java
// Signature
public class UnknownComponents<ECS_TYPE> implements Component<ECS_TYPE> {
```

## Architecture & Concepts

The UnknownComponents class is a critical forward-compatibility mechanism within the Hytale Entity-Component-System (ECS) framework. Its primary function is to act as a container for component data that the current running version of the engine does not recognize. This prevents data loss when entities are processed by older clients or servers that have not been updated with definitions for newer components.

When an entity is deserialized from a network packet or storage, the entity loader attempts to map each component ID to a registered Codec. If a component ID is encountered for which no Codec exists, instead of discarding the data or failing the entire deserialization, the system preserves the raw BSON data by storing it within an instance of UnknownComponents attached to that entity.

This design ensures that if the entity is later serialized, the unrecognized component data is written back out, unmodified. This allows for seamless round-trips of entity data through systems with heterogeneous versions, a common scenario in live-service game environments. It is fundamentally a data-preservation component, not intended for direct manipulation during standard gameplay logic.

## Lifecycle & Ownership

-   **Creation:** An UnknownComponents instance is typically created on-demand by the entity deserialization pipeline. When the pipeline encounters the first unrecognized component for a given entity, it instantiates UnknownComponents and attaches it to that entity. It can also be instantiated manually, though this is an exceptional use case.

-   **Scope:** The lifecycle of an UnknownComponents instance is strictly bound to the lifecycle of the entity to which it is attached. It persists as long as its parent entity exists in the world or in memory.

-   **Destruction:** The component is marked for garbage collection and destroyed when its parent entity is unloaded or destroyed. There is no manual destruction method; its cleanup is managed by the parent entity's lifecycle.

## Internal State & Concurrency

-   **State:** The internal state is mutable and consists of a single map, **unknownComponents**, which maps a component ID string to its raw BsonDocument representation. The class is essentially a specialized wrapper around this map, using an `Object2ObjectOpenHashMap` for performance.

-   **Thread Safety:** This class is **not thread-safe**. All methods that modify the internal map (**addComponent**, **removeComponent**) are unsynchronized. The use of `ExtraInfo.THREAD_LOCAL` during codec operations further reinforces the expectation that all interactions with an instance must occur on a single, controlled thread, typically the main game loop or a dedicated deserialization thread.

    **WARNING:** Concurrent access from multiple threads without external locking will lead to data corruption and unpredictable behavior. The method **getUnknownComponents** returns a direct reference to the internal map, making it highly susceptible to unsafe concurrent modification.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addComponent(id, component, codec) | void | O(1) avg | Encodes a known component into BSON and stores it. Primarily for system-level data migration tasks. |
| addComponent(id, component) | void | O(1) avg | Stores a pre-serialized component document. This is the primary ingress path for the deserialization pipeline. |
| contains(componentId) | boolean | O(1) avg | Checks for the existence of an unknown component by its ID. |
| removeComponent(id, codec) | T | O(1) avg | Removes a component by ID and attempts to deserialize it into a typed object. Returns null if not found. |
| getUnknownComponents() | Map | O(1) | Returns a direct reference to the internal map of unknown components. **Warning:** Modifying this map externally is an anti-pattern. |
| clone() | Component | O(N) | Creates a new UnknownComponents instance with a shallow copy of the internal map. N is the number of unknown components. |

## Integration Patterns

### Standard Usage

The UnknownComponents class is almost exclusively managed by the core entity serialization and deserialization systems. A developer would typically not interact with it directly. The most common interaction is passive: the system automatically preserves and re-serializes data without developer intervention.

```java
// This component is typically managed by the engine.
// A developer would not write this code in game logic.

// Hypothetical Deserialization Logic
Entity entity = new Entity();
BsonDocument entityData = ...; // Data from network

for (Entry<String, BsonValue> entry : entityData.entrySet()) {
    String componentId = entry.getKey();
    Codec<?> codec = componentRegistry.getCodec(componentId);

    if (codec != null) {
        // Known component, decode normally
        Component component = codec.decode(entry.getValue());
        entity.addComponent(component);
    } else {
        // Unknown component, preserve it
        UnknownComponents unknown = entity.getOrCreateComponent(UnknownComponents.class);
        TempUnknownComponent temp = new TempUnknownComponent(entry.getValue().asDocument());
        unknown.addComponent(componentId, temp);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **General Purpose Data Storage:** Do not use UnknownComponents as a generic key-value store for application data. It is strictly for forward-compatibility of unrecognized ECS components. Using it for other purposes pollutes its intended function and bypasses the strongly-typed nature of the ECS framework.

-   **External Map Modification:** Do not modify the map returned by **getUnknownComponents**. This breaks encapsulation and can lead to an inconsistent state, especially if the component is being accessed by the serialization system concurrently.

    ```java
    // DO NOT DO THIS
    UnknownComponents unknown = entity.getComponent(UnknownComponents.class);
    if (unknown != null) {
        // This bypasses all internal logic and is unsafe.
        unknown.getUnknownComponents().clear();
    }
    ```

-   **Multi-threaded Access:** Do not access or modify an UnknownComponents instance from multiple threads without implementing an external locking strategy. The class is fundamentally single-threaded.

## Data Pipeline

The UnknownComponents class serves as a bypass and preservation container within the primary entity data pipeline.

> Flow:
> Serialized Entity (BSON) -> Entity Deserializer -> Component Registry Lookup -> (Failure) -> **UnknownComponents.addComponent** -> Entity -> Entity Serializer -> **UnknownComponents.CODEC** -> Serialized Entity (BSON)

