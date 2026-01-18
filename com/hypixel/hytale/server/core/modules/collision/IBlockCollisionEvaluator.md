---
description: Architectural reference for IBlockCollisionEvaluator
---

# IBlockCollisionEvaluator

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IBlockCollisionEvaluator {
```

## Architecture & Concepts
The IBlockCollisionEvaluator interface defines a strict contract for block-specific collision evaluation algorithms. It serves as a core component of the server's physics engine, implementing a **Strategy Pattern** to decouple the main collision detection loop from the specific geometric calculations required for different block shapes.

This abstraction allows the physics system to handle a wide variety of block types—from simple cubes to complex shapes like stairs, slabs, and fences—without complicating the primary physics loop. The engine can select the appropriate evaluator implementation at runtime based on the type of block an entity is interacting with. This design is critical for performance and extensibility, as new block behaviors can be added by simply providing a new implementation of this interface.

## Lifecycle & Ownership
As an interface, IBlockCollisionEvaluator itself has no lifecycle. The lifecycle and ownership semantics apply to the **concrete classes** that implement this contract.

- **Creation:** Implementations are typically instantiated once at server startup and registered in a central registry, such as a BlockRegistry or CollisionEvaluatorRegistry. They are designed to be reusable and often stateless.
- **Scope:** Implementations are singletons with a server-wide scope, persisting for the entire duration of the server session. They are not created on-the-fly during collision checks.
- **Destruction:** Instances are destroyed only during server shutdown as part of the overall application context teardown.

## Internal State & Concurrency
- **State:** The interface contract implies that implementations should be **stateless**. All necessary context for a collision check is provided via the setCollisionData method. Storing per-evaluation state within an implementation is a severe anti-pattern and will lead to catastrophic concurrency failures.

- **Thread Safety:** Implementations must be unconditionally thread-safe. The physics engine may perform collision checks across multiple entities in parallel worker threads. Any implementation of this interface must be re-entrant and free of side effects. All calculations should rely solely on the arguments provided in the method calls.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setCollisionData(data, config, id) | void | O(1) | Configures the evaluator for a specific collision check. This method must be called before getCollisionStart. It primes the instance with block data, entity configuration, and block ID. |
| getCollisionStart() | double | O(N) | Executes the core collision algorithm and returns the time of impact, typically a value from 0.0 to 1.0. A value of 1.0 or greater indicates no collision occurred in the current tick. |

## Integration Patterns

### Standard Usage
The physics engine retrieves a pre-registered evaluator instance based on a block's type and uses it to perform a highly specific collision test. The evaluator is treated as a transient calculator.

```java
// Within the physics engine's collision resolution step
Block targetBlock = world.getBlockAt(pos);
IBlockCollisionEvaluator evaluator = CollisionRegistry.getEvaluatorFor(targetBlock.getType());

// Prime the evaluator with context for this specific check
evaluator.setCollisionData(blockData, entityCollisionConfig, targetBlock.getId());

// Execute the evaluation
double timeOfImpact = evaluator.getCollisionStart();

if (timeOfImpact < 1.0) {
    // A collision will occur; resolve it
    resolveCollision(entity, timeOfImpact);
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not store any state in member variables of an implementing class between method calls. This will break in a multi-threaded environment.
- **Incorrect Invocation Order:** Calling getCollisionStart before setCollisionData is an API contract violation and will result in undefined behavior or runtime exceptions.
- **Caching Results:** Do not attempt to cache the result of getCollisionStart within the evaluator. The physics engine is responsible for all caching strategies.

## Data Pipeline
The evaluator is a processing node within the broader entity physics pipeline. It receives highly specific, pre-filtered data and produces a single scalar value representing the collision result.

> Flow:
> Physics Tick -> Entity Movement Intent -> Broad-Phase Collision Check -> **IBlockCollisionEvaluator** -> Collision Resolution -> Final Entity Position

