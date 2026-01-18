---
description: Architectural reference for WorldPathBuilder
---

# WorldPathBuilder

**Package:** com.hypixel.hytale.builtin.path
**Type:** Component (Data)

## Definition
```java
// Signature
public class WorldPathBuilder implements Component<EntityStore> {
```

## Architecture & Concepts
The WorldPathBuilder is a server-side **Component** within Hytale's Entity-Component-System (ECS) architecture. Its sole responsibility is to attach a predefined `WorldPath` data structure to a server entity. It functions as a data container, not a service or manager.

Despite its name, it does not follow the traditional "Builder" design pattern. Instead, it *holds* a fully constructed `WorldPath` object. This component is the bridge between an entity's abstract concept of a path and the concrete list of waypoints it must follow. It is typically consumed by systems responsible for AI navigation and movement, such as a `PathFollowingSystem`, which would read the `WorldPath` from this component to direct the entity's physics and transform.

Its implementation of `Component<EntityStore>` signals to the engine that it is managed by the server's core `EntityStore` and its lifecycle is intrinsically tied to a parent entity.

### Lifecycle & Ownership
- **Creation:** A WorldPathBuilder is never instantiated directly in system logic. It is added to an entity, either programmatically via `entity.addComponent()` during entity construction or, more commonly, defined within an entity prefab (e.g., a JSON or HOCON file). The `PathPlugin` is responsible for registering this component's type with the engine at startup, making it available for use.
- **Scope:** The component's lifetime is identical to the entity to which it is attached. It persists as long as the entity exists in the world.
- **Destruction:** The component is marked for garbage collection when its parent entity is removed from the `EntityStore`. There is no manual destruction logic required.

## Internal State & Concurrency
- **State:** The component's state is **Mutable**. It holds a single reference to a `WorldPath` object, which can be replaced at any time via the `setPath` method. The `clone` method performs a deep copy of the contained `WorldPath`, ensuring that duplicated entities receive a distinct path instance.
- **Thread Safety:** This component is **not thread-safe** and must not be treated as such. As with most ECS components, it is designed to be accessed exclusively by systems operating on the main server thread within a designated phase of the game tick. Unsynchronized access from other threads will lead to race conditions and world state corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the PathPlugin. |
| getPath() | WorldPath | O(1) | Returns the current path instance associated with the entity. |
| setPath(WorldPath) | void | O(1) | Overwrites the entity's current path with a new instance. |
| clone() | Component | O(N) | Creates a deep copy of the component and its contained WorldPath. N is the number of waypoints. |

## Integration Patterns

### Standard Usage
The WorldPathBuilder is typically retrieved from an entity within a server-side `System`. The system then reads the path data to inform its logic.

```java
// Inside a server-side System that processes entities with paths
Entity entity = ...; // The entity being processed this tick
WorldPathBuilder pathComponent = entity.getComponent(WorldPathBuilder.getComponentType());

if (pathComponent != null) {
    WorldPath path = pathComponent.getPath();
    // Logic to make the entity follow the waypoints in 'path'
}
```

### Anti-Patterns (Do NOT do this)
- **Standalone Instantiation:** Do not create instances of WorldPathBuilder for any purpose other than adding them to an entity. They are data containers and have no utility on their own.
- **Cross-Thread Modification:** Never call `setPath` or modify the returned `WorldPath` object from a thread other than the main server thread. All state changes must be synchronized with the game loop.
- **Assumed Persistence:** Do not cache a reference to a WorldPathBuilder component. Always retrieve it from the entity on the frame you need it, as the component could be removed at any time.

## Data Pipeline
This component acts as a data source, not a processing stage. It injects pre-configured path data into the live server environment.

> Flow:
> World Editor / Prefab File -> Entity Definition -> **WorldPathBuilder** (on spawned entity) -> PathFollowingSystem -> Entity Physics Update

