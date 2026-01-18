---
description: Architectural reference for DirectionViewProvider
---

# DirectionViewProvider

**Package:** com.hypixel.hytale.math.hitdetection.view
**Type:** Transient

## Definition
```java
// Signature
public class DirectionViewProvider implements MatrixProvider {
```

## Architecture & Concepts
The DirectionViewProvider is a stateful component responsible for calculating a 4x4 view matrix, a fundamental operation in 3D rendering pipelines. It translates a camera's position, direction (defined by yaw and pitch), and orientation (the "up" vector) into the mathematical representation required by the graphics engine to render a scene from that specific point of view.

It implements the MatrixProvider interface, signaling its primary contract is to supply a valid Matrix4d on demand.

The core architectural pattern employed is **lazy evaluation with a dirty flag**. The internal boolean field, *invalid*, tracks whether the input parameters (position, direction) have changed since the last matrix calculation. Any call to a setter method such as setPosition or setDirection will mark the state as invalid. The computationally expensive matrix calculation is deferred until the getMatrix method is invoked, and only if the *invalid* flag is true. This prevents redundant matrix calculations if the view has not changed between frames, providing a critical performance optimization.

The static CODEC field indicates that this component is designed for serialization and deserialization, allowing view configurations to be defined in data files and loaded at runtime.

## Lifecycle & Ownership
- **Creation:** Instances are created in two primary ways:
    1.  Directly via `new DirectionViewProvider()`.
    2.  By the engine's codec system during deserialization of game data (e.g., loading an entity's camera attachment from a configuration file).
- **Scope:** The lifetime of a DirectionViewProvider is bound to its owner. It is a transient object, typically held as a member field within a more complex component like a Camera, a Player entity, or a cinematic controller. It persists as long as its owner does.
- **Destruction:** The object is managed by the Java garbage collector. No manual cleanup or destruction methods are required. It is reclaimed when its owner is no longer referenced.

## Internal State & Concurrency
- **State:** The DirectionViewProvider is highly **mutable**. Its internal state, including position, direction, and the cached matrix, is expected to be modified frequently, often once per frame. The entire design is centered around mutating the object's properties and then retrieving the calculated result.

- **Thread Safety:** This class is **NOT thread-safe** and must not be shared across threads without external synchronization. The dirty flag check-then-set logic in getMatrix presents a classic race condition. Concurrent calls to setter methods and getMatrix from different threads will lead to unpredictable behavior, including returning stale matrices or rendering artifacts.

    **WARNING:** All interactions with a DirectionViewProvider instance must be confined to a single thread, typically the main game loop or rendering thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setPosition(...) | DirectionViewProvider | O(1) | Sets the world-space position. Marks the internal state as invalid. |
| setDirection(...) | DirectionViewProvider | O(1) | Sets the view direction via vector or yaw/pitch. Marks the internal state as invalid. |
| setUp(...) | DirectionViewProvider | O(1) | Sets the "up" vector for orientation. Marks the internal state as invalid. |
| getMatrix() | Matrix4d | O(1) | Returns the cached view matrix. If the state is invalid, triggers a recalculation before returning. |

## Integration Patterns

### Standard Usage
The intended use is to maintain an instance of DirectionViewProvider within a controlling object. The controller updates the provider's state each frame and then retrieves the matrix for use in the rendering pipeline.

```java
// Within a Camera or Entity update method
// Assume 'provider' is a member field: DirectionViewProvider provider;

// 1. Update state based on game logic or player input
Vector3d newPosition = calculateCurrentPosition();
Vector3d newDirection = calculateLookDirection();
provider.setPosition(newPosition);
provider.setDirection(newDirection);

// 2. Retrieve the resulting matrix for the renderer
Matrix4d viewMatrix = provider.getMatrix();
renderingSystem.setViewMatrix(viewMatrix);
```

### Anti-Patterns (Do NOT do this)
- **Multi-threaded Access:** Never call setter methods from one thread while calling getMatrix from another. This will cause severe and difficult-to-debug concurrency issues.
- **Redundant Instantiation:** Do not create a `new DirectionViewProvider()` every frame inside a game loop. This generates excessive garbage collection pressure. Reuse and update a single instance.
- **State Negligence:** Calling getMatrix multiple times in a frame without updating the state is valid but inefficient if the state *should* have been updated. Ensure setters are called whenever the view perspective changes.

## Data Pipeline
The component serves two primary data flows: a runtime update flow and a data-driven initialization flow.

**Runtime Update Flow:**
> Flow:
> Game Logic (e.g., Player Input) -> `setPosition()` / `setDirection()` -> **DirectionViewProvider** (Internal state mutated, `invalid` flag set to true) -> `getMatrix()` -> Matrix Recalculation -> Render Engine

**Data-driven Initialization Flow:**
> Flow:
> Game Asset (e.g., JSON file) -> Engine Codec System -> **DirectionViewProvider** (Instance created and populated) -> Game Object (e.g., Entity) -> Render Engine

