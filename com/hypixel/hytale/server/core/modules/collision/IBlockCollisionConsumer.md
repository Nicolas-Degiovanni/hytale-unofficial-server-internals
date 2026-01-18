---
description: Architectural reference for IBlockCollisionConsumer
---

# IBlockCollisionConsumer

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Interface / Callback

## Definition
```java
// Signature
public interface IBlockCollisionConsumer {
   IBlockCollisionConsumer.Result onCollision(int var1, int var2, int var3, Vector3d var4, BlockContactData var5, BlockData var6, Box var7);
   IBlockCollisionConsumer.Result probeCollisionDamage(int var1, int var2, int var3, Vector3d var4, BlockContactData var5, BlockData var6);
   void onCollisionDamage(int var1, int var2, int var3, Vector3d var4, BlockContactData var5, BlockData var6);
   IBlockCollisionConsumer.Result onCollisionSliceFinished();
   void onCollisionFinished();

   public static enum Result {
      CONTINUE,
      STOP,
      STOP_NOW;
   }
}
```

## Architecture & Concepts

The IBlockCollisionConsumer interface defines a behavioral contract for systems that need to react to block-level physics collisions. It is a core component of the server-side physics engine, acting as a stateful callback receiver within the collision resolution pipeline.

This interface embodies the **Strategy Pattern**. The primary collision detection system is the *invoker*, responsible for sweeping an entity's bounding box through the world and identifying intersections with blocks. Instead of containing the logic for how to react to these collisions, the invoker delegates that responsibility to a concrete implementation of IBlockCollisionConsumer, the *strategy*.

This design decouples the low-level, performance-critical task of collision detection from the high-level game logic of collision response (e.g., stopping movement, taking damage, playing a sound). The `Result` enum is a critical feature, allowing the consumer to control the execution flow of the collision detection loop, enabling early termination for performance optimization.

## Lifecycle & Ownership

- **Creation:** Implementations are instantiated on-demand by higher-level systems, such as an `EntityMovementSystem`, immediately before a collision resolution check is performed for a single entity.
- **Scope:** The lifecycle of a consumer instance is extremely short and transient. It is scoped exclusively to a single collision resolution operation for one entity within a single server tick.
- **Destruction:** The object is intended to be discarded and garbage collected as soon as the collision resolution process completes. It holds temporary state that must not be carried over to subsequent ticks or other entities.

**WARNING:** Reusing an IBlockCollisionConsumer instance across multiple entities or physics ticks will lead to severe and difficult-to-diagnose bugs, including incorrect state accumulation and invalid physics responses.

## Internal State & Concurrency

- **State:** The interface itself is stateless. However, any concrete implementation is expected to be highly **mutable and stateful**. Its primary purpose is to accumulate the results of multiple collision callbacks (e.g., deepest penetration vector, material type) to compute a final, aggregate result for the physics engine.
- **Thread Safety:** Implementations of this interface are **not thread-safe** and must not be considered as such. The contract assumes a single-threaded execution context where a physics worker thread invokes the callbacks sequentially for a single entity's movement. Accessing a consumer instance from multiple threads will result in race conditions and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onCollision(...) | Result | O(1) | Primary callback invoked for each block intersection. The consumer processes the contact data and returns a Result to control the collision loop. |
| probeCollisionDamage(...) | Result | O(1) | A "dry-run" callback to determine the potential outcome of damage without applying it. Allows the system to predict consequences. |
| onCollisionDamage(...) | void | O(1) | Applies damage or other destructive effects after a successful probe. This separates decision-making from state mutation. |
| onCollisionSliceFinished() | Result | O(1) | Callback invoked after a discrete "slice" or chunk of the collision check is complete. Allows for intermediate processing or early exit. |
| onCollisionFinished() | void | O(1) | Final callback invoked when the entire collision resolution process for the entity has concluded. Used for finalization and cleanup. |

## Integration Patterns

### Standard Usage

A consumer is created by a system managing entity physics, passed to a collision utility, and its final state is queried after the utility returns.

```java
// A system responsible for entity movement creates a consumer
EntityCollisionConsumer consumer = new EntityCollisionConsumer(entity);

// It passes the consumer to the core collision detection service
collisionSystem.resolveMovement(entity.getBoundingBox(), entity.getVelocity(), consumer);

// After resolution, the system retrieves the computed result from the consumer
Vector3d finalVelocity = consumer.getFinalVelocity();
entity.setVelocity(finalVelocity);
```

### Anti-Patterns (Do NOT do this)

- **Instance Caching:** Caching and reusing a consumer instance for the same entity on the next tick. The consumer's internal state (e.g., penetration depth) is only valid for the single operation it was created for.
- **Asynchronous Callbacks:** Invoking consumer methods from a separate thread or with a delay. The collision system relies on the immediate, synchronous return value of `Result` to manage its internal loops.
- **Complex Logic:** Implementing long-running or blocking operations (e.g., database queries, network I/O) within a callback. This will stall the server's main physics thread, causing catastrophic performance degradation.

## Data Pipeline

The IBlockCollisionConsumer acts as a stateful processor in the middle of the server's physics data flow.

> Flow:
> Entity Movement Tick -> `EntityMovementSystem` creates **IBlockCollisionConsumer** -> `CollisionDetector` receives consumer -> `CollisionDetector` invokes `onCollision` repeatedly -> Consumer accumulates state and returns `Result` -> `EntityMovementSystem` reads final state from consumer -> Entity position and velocity are updated.

