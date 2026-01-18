---
description: Architectural reference for TransientPathDefinition
---

# TransientPathDefinition

**Package:** com.hypixel.hytale.builtin.path.path
**Type:** Transient

## Definition
```java
// Signature
public class TransientPathDefinition {
```

## Architecture & Concepts
The TransientPathDefinition class serves as an immutable blueprint or template for a path. It does not represent a concrete path existing within the game world, but rather the abstract, reusable definition of a path's shape and scale.

Its primary role is to decouple the design of a path (e.g., a patrol route defined in a data file) from its runtime instantiation. It holds a sequence of waypoints defined *relative* to one another. The core functionality is exposed through the `buildPath` method, which acts as a factory. This method takes a concrete starting position and orientation in world-space and generates a fully realized IPath object that an entity can follow.

This pattern allows a single path definition to be reused by multiple entities at different locations and orientations throughout the world, promoting efficient asset management and simplifying AI behavior logic.

### Lifecycle & Ownership
-   **Creation:** Instances are created via the public constructor, typically by systems that load and parse game data, such as AI behavior definitions, quest triggers, or scripted cinematic sequences. It is a standard Java object, not managed by a service locator.
-   **Scope:** The lifetime of a TransientPathDefinition is bound to the component that requires it. For example, if it defines a monster's patrol route, it will exist as long as the monster's AI definition is loaded in memory. It is not a global or session-scoped object.
-   **Destruction:** The object is managed by the Java garbage collector and is reclaimed when no longer referenced. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The class is designed to be **immutable**. Its internal state, consisting of the `waypointDefinitions` list and a `scale` factor, is established at construction time and cannot be changed. Both fields are marked `final`.
-   **Thread Safety:** Due to its immutable nature, TransientPathDefinition is inherently **thread-safe**. The `buildPath` method is a pure function with no side effects on the instance's state. It can be safely called by multiple threads concurrently without external locking.

**WARNING:** While the class itself is immutable, the `List` of waypoint definitions passed to the constructor is not defensively copied. Modifying this list externally after construction is a severe anti-pattern that will lead to unpredictable behavior and violate the thread-safety guarantee. The provided list should be treated as immutable once passed to the constructor.

## API Surface
The public contract is minimal, focusing exclusively on path instantiation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| buildPath(position, rotation) | IPath | O(N) | Factory method that builds a concrete IPath instance. N is the number of waypoints in the definition. Throws NullPointerException if arguments are null. |

## Integration Patterns

### Standard Usage
The intended use is to create a definition once from a data source and then use it repeatedly to spawn path instances for game entities.

```java
// 1. Obtain the list of relative waypoints, typically from an asset loader.
List<RelativeWaypointDefinition> waypoints = loadMyPathDefinition();
double scale = 1.0;

// 2. Create the immutable path definition. This can be cached and reused.
TransientPathDefinition pathDefinition = new TransientPathDefinition(waypoints, scale);

// 3. At runtime, when an entity needs to follow the path, build a concrete instance.
Vector3d entityPosition = entity.getPosition();
Vector3f entityRotation = entity.getRotation();
IPath concretePath = pathDefinition.buildPath(entityPosition, entityRotation);

// 4. Pass the concrete path to a movement or AI system.
entity.getAI().followPath(concretePath);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation:** Do not modify the source `List<RelativeWaypointDefinition>` after it has been passed to the constructor. This breaks the immutability contract.
-   **Frequent Re-creation:** Do not create a new TransientPathDefinition every time an entity needs a path. The definition object is designed to be lightweight and reusable. Create it once and cache it.
-   **Path Caching:** Do not cache the `IPath` object returned by `buildPath` for reuse by another entity at a different location. The returned path is specific to the position and rotation provided at the time of the call.

## Data Pipeline
TransientPathDefinition acts as a transformation stage, converting abstract, relative data into a concrete, world-space object.

> Flow:
> Game Asset (e.g., JSON) -> Asset Loader -> `List<RelativeWaypointDefinition>` -> **TransientPathDefinition** -> `buildPath(pos, rot)` -> `IPath` Instance -> Entity Movement System

