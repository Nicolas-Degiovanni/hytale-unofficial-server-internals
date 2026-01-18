---
description: Architectural reference for TempUnknownComponent
---

# TempUnknownComponent

**Package:** com.hypixel.hytale.component.data.unknown
**Type:** Transient Data Holder

## Definition
```java
// Signature
public class TempUnknownComponent<ECS_TYPE> implements Component<ECS_TYPE> {
```

## Architecture & Concepts
The TempUnknownComponent is a critical fallback mechanism within the engine's Entity-Component-System (ECS) data pipeline. Its sole purpose is to preserve unrecognized component data during deserialization, ensuring forward compatibility between different versions of the game client and server.

When the engine loads an entity from a data stream (e.g., a network packet or a world save file), it may encounter component data for which it has no corresponding registered class. This typically occurs when a newer client connects to an older server, or vice-versa.

Instead of discarding this data, which would lead to data loss, the deserialization system wraps the raw BSON data in a TempUnknownComponent instance. This component acts as an opaque container, safeguarding the data until the entity is serialized again. Upon serialization, the system extracts the original, unmodified BSON document and writes it back to the stream. This design guarantees that data integrity is maintained across version boundaries, even for components the current runtime does not understand.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the engine's component deserialization logic via its static COMPONENT_CODEC. This occurs automatically when a component type from a data source does not map to any known, registered Component class. It is never created directly by gameplay code.
- **Scope:** The lifetime of a TempUnknownComponent is strictly bound to the lifetime of the entity to which it is attached. It persists in memory only as long as the entity exists.
- **Destruction:** The instance is marked for garbage collection when its parent entity is destroyed or when the entity is serialized, as its data has been successfully written back to a data stream.

## Internal State & Concurrency
- **State:** The component's state consists of a single, mutable BsonDocument. However, the component is designed to be treated as a value object. The public clone method performs a deep copy of the underlying BsonDocument, which is essential for entity duplication and snapshotting operations to prevent shared state.
- **Thread Safety:** This class is not thread-safe. Like all ECS components, it is designed to be accessed and managed by the single thread responsible for its parent entity's world simulation. Unsynchronized access from other threads will result in undefined behavior and data corruption.

## API Surface
The public API is minimal, intended for internal system use only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDocument() | BsonDocument | O(1) | Returns a reference to the internal BSON document. **Warning:** Modifying this document is an anti-pattern. |
| clone() | Component | O(N) | Creates a new TempUnknownComponent containing a deep copy of the BsonDocument. N is the size of the document. |

## Integration Patterns

### Standard Usage
There is no standard usage pattern for game developers. This component is an internal implementation detail of the data serialization engine and should be considered transparent to all gameplay systems. Its presence on an entity indicates a version mismatch that the engine is handling gracefully.

The following example illustrates the *engine's* internal interaction, not developer code.
```java
// Hypothetical engine deserialization logic
BsonDocument rawComponentData = readDataFromStream();
ComponentId componentId = readComponentIdFromStream();

Component component;
if (componentRegistry.hasClassFor(componentId)) {
    component = componentRegistry.deserialize(componentId, rawComponentData);
} else {
    // This is the only valid creation path
    component = new TempUnknownComponent<>(rawComponentData);
}
entity.addComponent(component);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new TempUnknownComponent()`. The component's lifecycle is managed entirely by the serialization framework.
- **Gameplay Logic Dependency:** Do not write systems that query for this component type. Code such as `entity.has(TempUnknownComponent.class)` creates a fragile dependency on an internal mechanism and attempts to handle a versioning issue that the engine already solves.
- **State Mutation:** Do not retrieve the component, call getDocument, and then modify the returned BsonDocument. This corrupts the "pass-through" nature of the data and will lead to unpredictable behavior and likely data loss upon reserialization.

## Data Pipeline
The TempUnknownComponent serves as a temporary bridge in the data round-trip process, ensuring that what comes in goes back out, even if it is not understood.

> Flow:
> Network Packet / Disk Read -> BSON Deserializer -> Encounter Unknown Component ID -> **TempUnknownComponent** (Created) -> Attached to Entity -> Entity Lifecycle -> BSON Serializer -> **TempUnknownComponent** (Read) -> Original BSON written to stream -> Network Packet / Disk Write

