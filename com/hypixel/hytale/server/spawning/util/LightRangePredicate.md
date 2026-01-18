---
description: Architectural reference for LightRangePredicate
---

# LightRangePredicate

**Package:** com.hypixel.hytale.server.spawning.util
**Type:** Transient

## Definition
```java
// Signature
public class LightRangePredicate {
```

## Architecture & Concepts
The LightRangePredicate is a stateful, single-use component designed to evaluate environmental lighting conditions at a specific world coordinate. It serves as a critical bridge between the declarative spawn definitions (loaded from game assets) and the physical state of the world's lighting engine.

Its primary role within the server architecture is to act as a highly specific, configurable filter for the entity spawning system. Instead of embedding complex lighting checks directly into spawner logic, a spawner instantiates and configures a LightRangePredicate based on a creature's spawn rules. This predicate then encapsulates the logic required to query the world's chunk data and determine if the light levels at a potential spawn location meet the required criteria.

This class decouples the high-level concept of a "spawn condition" from the low-level implementation of light storage and calculation within a BlockChunk, promoting cleaner, data-driven spawning logic. It supports a variety of light channels, including general block light, sky light, dynamic sunlight, and individual RGB channels, reflecting the sophistication of the underlying lighting engine.

## Lifecycle & Ownership
- **Creation:** An instance is created using a direct constructor call, typically by a higher-level system like an entity spawner that is processing a specific spawn rule. It is a plain Java object and is not managed by a dependency injection container or service registry.
- **Scope:** The object's lifetime is intentionally short and tied to a specific evaluation task. A new instance is typically created, configured via its setter methods, used to test one or more locations for a single spawn attempt, and then immediately becomes eligible for garbage collection.
- **Destruction:** The object is managed by the Java garbage collector. There are no manual cleanup or disposal methods required. Its minimal state ensures a low memory footprint.

## Internal State & Concurrency
- **State:** The LightRangePredicate is highly mutable. Its internal state consists of minimum and maximum light level thresholds for various light types, along with boolean flags that activate or deactivate specific checks. This state is configured post-instantiation through the setLightRange methods. The object is useless until this state is configured.

- **Thread Safety:** **This class is not thread-safe.** Its internal fields are accessed and modified without any synchronization mechanisms. Concurrent calls to setter methods or a mix of setter and test method calls from different threads will lead to race conditions and unpredictable behavior.

    **WARNING:** Instances of LightRangePredicate must be confined to a single thread. They are designed for use within the server's main world tick loop and should never be shared across different worlds or worker threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setLightRange(LightType type, double[] range) | void | O(1) | Configures the min/max light range for a given light type. The range is specified as a percentage (0-100). |
| test(World world, Vector3d position, ...) | boolean | O(1) | The primary evaluation method. Checks if the light levels at the given world position satisfy all configured predicates. |
| test(BlockChunk chunk, int x, int y, int z, ...) | boolean | O(1) | A lower-level evaluation method for performance-critical paths where the BlockChunk is already available. |
| calculateLightValue(...) | static byte | O(1) | A static utility to calculate the maximum effective light at a block, considering both block light and sunlight. |

## Integration Patterns

### Standard Usage
The intended use is a create-configure-test-discard pattern, driven by a higher-level system processing data-driven rules.

```java
// Example from within a hypothetical Spawner class
LightRangePredicate predicate = new LightRangePredicate();

// Configure the predicate based on spawn rule data (e.g., from JSON)
// Rule: Must spawn in an area with 0% to 20% light.
double[] lightCondition = { 0.0, 20.0 };
predicate.setLightRange(LightType.Light, lightCondition);

// Test a potential spawn location
Vector3d potentialPosition = new Vector3d(100, 64, 250);
boolean canSpawnHere = predicate.test(world, potentialPosition, componentAccessor);

if (canSpawnHere) {
    // Proceed with spawning logic
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not create a single LightRangePredicate and reuse it for different spawn rules. Its internal state will persist, leading to incorrect evaluations. Always create a new instance for each distinct set of rules.
- **Concurrent Access:** Never share an instance across multiple threads. This will cause severe data corruption and non-deterministic behavior. Each thread of execution must own its own instances.
- **Testing Without Configuration:** Calling the test method on a default-constructed instance is meaningless. All internal test flags are false, so the method will always return true, bypassing all light checks.

## Data Pipeline
The LightRangePredicate acts as a processing node that consumes world state and produces a boolean decision.

> Flow:
> Spawn Rule Asset -> Spawning System -> **LightRangePredicate.test(world, position)** -> World Chunk Cache -> BlockChunk Light Data -> Boolean Result -> Spawning System

