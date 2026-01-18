---
description: Architectural reference for DetailBox
---

# DetailBox

**Package:** com.hypixel.hytale.server.core.asset.type.model.config
**Type:** Transient

## Definition
```java
// Signature
public class DetailBox implements NetworkSerializable<com.hypixel.hytale.protocol.DetailBox> {
```

## Architecture & Concepts
The DetailBox class is a server-side data structure representing a component of a larger model's physical configuration. It encapsulates a geometric volume, defined by a **Box**, and its relative position, defined by a **Vector3d** offset.

Its primary architectural roles are:
1.  **Asset Configuration:** It serves as a deserialization target for model definition files (e.g., JSON or HOCON). The static **CODEC** field is the contract used by the asset pipeline to parse and construct DetailBox instances from disk. This allows designers and artists to define physical properties like hitboxes or selection volumes declaratively.
2.  **Network Bridging:** By implementing NetworkSerializable, this class acts as a bridge between the server's internal, high-precision game state and the lower-precision network protocol. The toPacket method is responsible for converting the server's double-precision vectors (Vector3d) into the network-optimized float-precision vectors (Vector3f), a critical pattern for conserving network bandwidth.

This class is not a service or manager; it is a fundamental building block used to compose more complex game objects.

### Lifecycle & Ownership
-   **Creation:** DetailBox instances are primarily created by the asset loading system during server bootstrap or hot-reloading. The static **CODEC** is invoked to deserialize model configuration files. They can also be instantiated programmatically at runtime, often as temporary objects for geometric calculations or as copies of existing asset data.
-   **Scope:** The lifetime of a DetailBox is strictly tied to its owning object, typically a model asset definition or a component of a game entity. It is a value object and holds no global state.
-   **Destruction:** Instances are marked for garbage collection when their parent asset is unloaded or the owning entity is destroyed. No manual resource management is required.

## Internal State & Concurrency
-   **State:** The class is mutable. Its internal **offset** and **box** fields are mutable objects and can be modified after construction. Operations like the copy constructor or the **scaled** method create new instances rather than modifying in-place, but the underlying fields remain publicly accessible and modifiable via their respective getters.
-   **Thread Safety:** This class is **not thread-safe**. It is a simple data container and provides no internal locking. It is designed to be owned and manipulated by a single thread, such as the main server tick loop or a dedicated asset loading thread. Unsynchronized access from multiple threads will lead to race conditions and undefined behavior.

## API Surface
The public API provides access to the underlying geometric data and the network serialization contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getOffset() | Vector3d | O(1) | Returns the internal offset vector. **Warning:** The returned object is mutable. |
| getBox() | Box | O(1) | Returns the internal bounding box. **Warning:** The returned object is mutable. |
| scaled(float scale) | DetailBox | O(1) | Creates and returns a new, independent DetailBox with its offset and box dimensions scaled. |
| toPacket() | com.hypixel.hytale.protocol.DetailBox | O(1) | Converts the server-side object into its network protocol equivalent, including precision down-casting. |

## Integration Patterns

### Standard Usage
A DetailBox is typically retrieved from a parent model asset. Developers should treat it as read-only data for calculations. If modification is needed, a copy should be made first.

```java
// Assume 'modelAsset' is a loaded model configuration
DetailBox mainHitbox = modelAsset.getCollisionBox();

// Use the data for a physics query
Vector3d worldPosition = entity.getPosition().add(mainHitbox.getOffset());
boolean isIntersecting = physicsEngine.query(worldPosition, mainHitbox.getBox());
```

### Anti-Patterns (Do NOT do this)
-   **Shared State Mutation:** Modifying a DetailBox retrieved from a cached asset is a critical error. This will alter the base asset for all subsequent users, leading to unpredictable and widespread bugs.

    ```java
    // DO NOT DO THIS
    ModelAsset asset = AssetCache.get("my_model");
    DetailBox hitbox = asset.getCollisionBox();
    hitbox.getOffset().setX(100.0); // This corrupts the cached asset for everyone!
    ```

-   **Ignoring Precision Loss:** Relying on the network packet representation for server-side logic is incorrect. The conversion to a packet involves a loss of precision from double to float, which is acceptable for client-side rendering but can cause significant errors in server-side physics or gameplay calculations.

## Data Pipeline
The DetailBox is a key component in the pipeline that transforms asset definitions on disk into synchronized data on the client.

> Flow:
> Model JSON File -> AssetManager (using **CODEC**) -> **DetailBox** (in-memory on server) -> toPacket() -> Network Packet -> Client Deserialization -> Client-side Rendering/Physics

---

